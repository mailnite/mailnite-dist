# Install Mailnite on Amazon EKS

This guide takes you from an AWS account to a working mail server in about
45 minutes. You get:

- **Mailnite** on EKS, as one stateful pod with its mail store on an
  encrypted EBS gp3 volume.
- **Its own static public IP** (an Elastic IP) on a Network Load Balancer:
  `mail.<your-domain>` for SMTP (25), submissions (465), submission (587),
  IMAPS (993), IMAP (143) and HTTPS (443).
- **Webmail** at `https://mail.<your-domain>/mail`, with a Let's Encrypt
  certificate the server obtains and renews itself.
- **Hourly automatic updates** (see [Updating](#11-updating)), plus a master key
  that is generated inside your cluster and never leaves it.

No mail relay, no ingress and no cert-manager are needed: the load balancer
is the mail server's edge. The guide uses a standard EKS cluster with the
AWS Load Balancer Controller: EKS Auto Mode's load balancers can't take an
Elastic IP, and MX records need an address that never changes.

> **One AWS fact before you start.** AWS limits **outbound port 25** on EC2
> until you ask for the limit to be removed. Receiving mail works; to
> **send** to other servers you use Amazon SES as a smarthost on port 587.
> See [step 9](#9-sending-mail-with-amazon-ses).

---

## Before you start

- **An AWS account** where you may create EKS clusters, IAM roles and
  policies, and Elastic IPs.
- **A domain whose DNS you control.** For a trial, use a subdomain: with
  `test.example.com` as the mail domain, the server lives at
  `mail.test.example.com`.
- **Tools:** AWS CLI v2 (signed in: `aws configure` or `aws sso login`),
  eksctl, kubectl, and Helm 3.8 or later (for OCI charts).

```bash
# macOS
brew install awscli eksctl kubectl helm
# Linux: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html · https://eksctl.io/installation/
```

Set a few names used throughout this guide:

```bash
CLUSTER=mailnite                 # EKS cluster name
REGION=us-east-1                 # any region
ZONE=${REGION}a                  # the node, its disk and the load balancer share this zone
DOMAIN=example.com               # your mail domain: the server becomes mail.$DOMAIN
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
```

Why one zone: an EBS volume lives in one Availability Zone, and a load
balancer with a single Elastic IP lives in one subnet, so in one zone too.
The pod must run where both are.

---

## 1. Create the cluster

`eksctl` creates the VPC, the cluster, its IAM roles and a node group in one
command. EKS needs two zones for its control plane; `--node-zones` keeps the
node in `$ZONE`.

```bash
eksctl create cluster --name $CLUSTER --region $REGION \
  --zones ${REGION}a,${REGION}b --node-zones $ZONE \
  --nodegroup-name default --node-type t3.large --nodes 1 \
  --with-oidc --kubeconfig $HOME/.kube/mailnite-eks
```

Provisioning takes 15 to 20 minutes. `t3.large` has 2 vCPU and 8 GiB. The
console can build the same cluster, but it needs the VPC, two IAM roles and
the node group created one by one; eksctl is the short path.

---

## 2. Connect kubectl

eksctl wrote the cluster's credentials to their own kubeconfig file.
Everything in this shell then targets EKS, and any other cluster in your
`~/.kube/config` stays out of reach of a mistyped command.

```bash
export KUBECONFIG=$HOME/.kube/mailnite-eks
kubectl get nodes                 # one node, Ready

# in a new shell, or on another machine:
# aws eks update-kubeconfig --name $CLUSTER --region $REGION --kubeconfig $KUBECONFIG
```

---

## 3. Add storage and the load balancer controller

An EKS cluster starts without two things Mailnite needs: a driver that
creates EBS volumes, and the controller that builds Network Load Balancers
with Elastic IPs.

**EBS volumes.** Install the EBS CSI driver add-on with its own IAM role, and
a `gp3` storage class:

```bash
eksctl create iamserviceaccount --cluster $CLUSTER --region $REGION \
  --namespace kube-system --name ebs-csi-controller-sa \
  --role-name AmazonEKS_EBS_CSI_DriverRole_$CLUSTER --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
eksctl create addon --cluster $CLUSTER --region $REGION --name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::$ACCOUNT:role/AmazonEKS_EBS_CSI_DriverRole_$CLUSTER \
  --force

kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  encrypted: "true"
EOF
```

**AWS Load Balancer Controller.** Create its IAM policy (once per AWS
account), a role for it, and install it:

```bash
curl -fsSLo iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
eksctl create iamserviceaccount --cluster $CLUSTER --region $REGION \
  --namespace kube-system --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::$ACCOUNT:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts --approve

VPC=$(aws eks describe-cluster --name $CLUSTER --region $REGION \
  --query cluster.resourcesVpcConfig.vpcId --output text)
helm repo add eks https://aws.github.io/eks-charts && helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --version 1.14.0 \
  --set clusterName=$CLUSTER --set region=$REGION --set vpcId=$VPC \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
kubectl -n kube-system rollout status deployment/aws-load-balancer-controller
```

If `create-policy` reports that the policy already exists, it came from an
earlier install in this account. Bring it up to this version:

```bash
aws iam create-policy-version --set-as-default --policy-document file://iam_policy.json \
  --policy-arn arn:aws:iam::$ACCOUNT:policy/AWSLoadBalancerControllerIAMPolicy
```

(AWS keeps five versions of a policy; delete an old one if it refuses.)

---

## 4. Reserve a static public IP

MX records, DNS, and your address's reputation with other mail servers all
depend on an IP that doesn't change. On AWS that is an Elastic IP, attached
to the load balancer in one public subnet of `$ZONE`:

```bash
SUBNET=$(aws ec2 describe-subnets --region $REGION \
  --filters Name=vpc-id,Values=$VPC Name=availability-zone,Values=$ZONE \
            Name=tag:kubernetes.io/role/elb,Values=1 \
  --query 'Subnets[0].SubnetId' --output text)
ALLOC=$(aws ec2 allocate-address --domain vpc --region $REGION --query AllocationId --output text)
IP=$(aws ec2 describe-addresses --region $REGION --allocation-ids $ALLOC \
  --query 'Addresses[0].PublicIp' --output text)
echo $SUBNET $ALLOC $IP
```

The Elastic IP belongs to the account, not to the cluster, so it survives
cluster rebuilds.

---

## 5. Install Mailnite

Write the values file:

```bash
cat > mailnite-aws.yaml <<EOF
domain: $DOMAIN
exposure: loadBalancer          # Mailnite gets its own public IP
platform: eks
mail:
  loadBalancer:
    annotations:
      # one subnet and one Elastic IP: the load balancer lives in $ZONE
      service.beta.kubernetes.io/aws-load-balancer-subnets: $SUBNET
      service.beta.kubernetes.io/aws-load-balancer-eip-allocations: $ALLOC
      # The load balancer resets connections idle for 350 seconds; IMAP IDLE
      # (push mail in Apple Mail, Thunderbird, …) needs longer. Chart 0.2.1+
      # sets these for platform: eks; they're kept here for older charts.
      service.beta.kubernetes.io/aws-load-balancer-listener-attributes.TCP-993: tcp.idle_timeout.seconds=1800
      service.beta.kubernetes.io/aws-load-balancer-listener-attributes.TCP-143: tcp.idle_timeout.seconds=1800
persistence:
  storageClass: gp3
  size: 32Gi                    # the whole mail store lives on this volume
EOF
```

`platform: eks` asks the controller for an internet-facing Network Load
Balancer that sends traffic straight to the pod and keeps each client's
real IP address.

Install the chart:

```bash
helm install mailnite oci://ghcr.io/mailnite/charts/mailnite \
  -n mail --create-namespace -f mailnite-aws.yaml
```

Helm prints the next steps; they match this guide. Watch the pod start:

```bash
kubectl -n mail get pods -w
```

The pod goes `Init:0/1` → `Running 0/1`. **`0/1` is expected** until the setup
wizard's first phase is done: readiness means "configured and serving mail".
Until then the load balancer has no target, and connections to the IP time
out.

On its first start the pod creates the **Server Master Key**, which
encrypts the mail store, in the Secret `mailnite-keys`. **Save a copy
somewhere other than this cluster, now.** Without it, nobody can open the
mailboxes.

```bash
kubectl -n mail get secret mailnite-keys -o jsonpath='{.data.MAIL_SMK}' | base64 -d; echo
```

Check the load balancer. On AWS, `EXTERNAL-IP` shows its DNS name, which
resolves to your Elastic IP:

```bash
kubectl -n mail get svc mailnite-mail      # EXTERNAL-IP: k8s-mail-mailnite-….elb.….amazonaws.com
dig +short $(kubectl -n mail get svc mailnite-mail -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')   # $IP
```

---

## 6. Point DNS at the IP

At your DNS provider, create:

| Name | Type | Value |
|---|---|---|
| `mail.<your-domain>` | A | the Elastic IP (`$IP`) |
| `<your-domain>` | MX | `10 mail.<your-domain>` |

Use an A record to the Elastic IP, not a CNAME to the load balancer's name.
On **Cloudflare**, set it to **DNS only** (grey cloud). Cloudflare's proxy
carries web traffic, not SMTP or IMAP.

```bash
dig +short mail.$DOMAIN            # must print $IP before the TLS step below
```

The setup wizard lists the remaining records (SPF, DKIM, DMARC, MTA-STS). It
can publish them for you if your DNS is on Route 53 or Cloudflare.

---

## 7. Finish setup in the browser

```bash
kubectl -n mail port-forward statefulset/mailnite 8480:8480
```

Open **http://localhost:8480/mail/**. Keep the port-forward running while you
work in the wizard.

1. **Console password and license.** The password protects this setup
   console.
2. **Storage.** Leave the key field empty. Its hint shows `env://MAIL_SMK`:
   the master key from step 5, which the chart supplies to the server.
   Storage limits: defaults are fine.
3. **Organization and admin account.** This is your first mailbox and
   administrator.
4. **Admin console setup:**
   - **Host.** Set **Primary IP address** to your Elastic IP (`$IP`).
     Auto-detection would find the node's address, which differs from the
     load balancer's, and the DNS checklist would then be wrong.
   - **DNS / Domains.** Publish or copy the listed records.
   - **Mail relay.** Choose **No relay — reachable directly**; the load
     balancer is your edge.
   - **TLS.** Choose Let's Encrypt and tick **I accept the Let's Encrypt
     Subscriber Agreement**; Save stays disabled until you do.
     `mail.<your-domain>` must already resolve to `$IP`. Validation runs
     over port 443 through the load balancer (TLS-ALPN-01), so port 80
     isn't needed.
   - **Ports.** **Keep the preset high port numbers** (2525, 2465, 2587, 2993, 2143;
     web 8080, 8443). The load balancer maps the standard ports (25, 465,
     587, 993, 143, 443) onto them.
   - **Servers.** Every server shows as bound, and the certificate line
     turns to "issued" within a few minutes. If you press Next before
     that, the wizard warns once; the next press continues, and issuance
     keeps retrying in the background.

---

## 8. Check that it works

```bash
kubectl -n mail get pods                   # mailnite-0 is now 1/1 Ready

# the server's own Let's Encrypt certificate, served through the AWS load balancer
openssl s_client -connect mail.$DOMAIN:443 -servername mail.$DOMAIN </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

A Network Load Balancer takes a few minutes to start sending traffic to a
pod that has just turned Ready.

- **Webmail:** open `https://mail.<your-domain>/mail` and sign in with the
  admin account.
- **Receiving:** send a message from Gmail or Outlook.com to your new mailbox.
  In webmail, open the message's original headers. The top `Received:` line
  must name the sender's public IP, not a `192.168.x` or `10.x` address. That
  shows the load balancer passes the real client address through, which SPF,
  DNSBL checks and rate limits depend on.
- **Mail clients:** add the account in any IMAP client, with
  `mail.<your-domain>`, IMAP on 993 (SSL/TLS) and SMTP on 587 (STARTTLS) or
  465 (SSL/TLS).

> Your home or office network may block outbound port 25 itself, so
> `telnet mail.<your-domain> 25` from your laptop can fail while the server is
> fine. Test receiving with a real message instead.

---

## 9. Sending mail with Amazon SES

Sending directly doesn't work well here. AWS limits port 25 until you ask,
and the load balancer carries inbound traffic only: mail would leave from
the node's address, not from `$IP`, and that address changes whenever the
node is replaced. **Infrastructure → Outbound → Test outbound SMTP** shows
the port 25 limit. Send through Amazon SES instead:

1. In the **SES console** (same region), go to **Identities → Create
   identity → Domain**, enter your domain, and keep **Easy DKIM**. Publish
   the three CNAME records it lists. On Route 53, SES can publish them for
   you.
2. **Request production access** from the SES account dashboard. A new SES
   account is in the sandbox and can send only to addresses you verified.
3. Under **SMTP settings**, choose **Create SMTP credentials**. You get an
   SMTP user name and password.
4. In Mailnite, open **Infrastructure → Outbound**. Set **Smarthost** to
   `email-smtp.<region>.amazonaws.com:587`, and enter the SES user name and
   password under **Smarthost login**.

DMARC then passes on SES's DKIM signature for your domain. Your SPF record
needs `include:amazonses.com` only if you set up a custom MAIL FROM domain
in SES.

---

## 10. Running it

**Back up the master key** (step 5) and keep it off the cluster. For the mail
data itself, turn on **Administration → Storage → Automatic backups**. They
can also copy to S3.

**Node group upgrades** drain the node. The chart's PodDisruptionBudget
stops the only Mailnite pod from being evicted, so after 15 minutes EKS
gives up and the upgrade fails. For an upgrade window, relax the budget
(mail pauses briefly while the pod moves; sending servers retry):

```bash
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-aws.yaml \
  --set podDisruptionBudget.enabled=false
```

---

## 11. Updating

Three things change over time, each with its own command, and none of them
undoes another: **Mailnite** (the server image), **the chart**, and **your
values file**.

### What is running

```bash
helm list -n mail                                    # the chart version; APP VERSION is the chart's default, not always what runs
kubectl -n mail get statefulset mailnite \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'   # the server version actually running
kubectl -n mail exec mailnite-0 -- /app/mailnite version   # the same, asked from inside
```

The admin console's **Overview** page shows the version too.

### Mailnite updates itself

The chart installs an hourly CronJob, `mailnite-update`. It compares the
running image with the stable release channel (`get.mailnite.com`) and moves
the StatefulSet forward, the way the Linux installer's timer does:

- only after the new image is published on Docker Hub;
- forward only, and never to a new **major** version by itself (set
  `autoUpdate.allowMajor: true` to follow those too);
- if the server isn't Ready again within 10 minutes, it puts the old image
  back and marks that version as failed, so it isn't retried; the next
  release is.

You don't have to do anything. To watch it work:

```bash
kubectl -n mail get cronjob mailnite-update          # LAST SCHEDULE
kubectl -n mail get jobs                             # the last run; failed runs are kept
kubectl -n mail logs job/<job-name>                  # e.g. "up to date: 1.4.10 (channel v1.4.10)"
```

### Update Mailnite now

The same thing, without waiting for the hour:

```bash
kubectl -n mail create job --from=cronjob/mailnite-update update-now
kubectl -n mail logs -f job/update-now
kubectl -n mail delete job update-now
```

The pod restarts once with the new image. Mail pauses for a moment while it
does; sending servers retry, and the load balancer keeps its Elastic IP.

### Update the chart, or change your values

Keep `mailnite-aws.yaml`: Helm applies the whole file on every upgrade. Edit it
to change a setting, then run the upgrade. The same command also brings in a
newer chart version:

```bash
helm show chart oci://ghcr.io/mailnite/charts/mailnite | grep -E '^(version|appVersion):'   # the newest chart
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-aws.yaml
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-aws.yaml --version 0.2.1   # one specific chart version
```

An upgrade never moves the server backwards: the chart reads the image the
updater has installed and keeps it whenever it is newer than the chart's own
default. What each chart version changes is on the
[chart's releases page](https://github.com/mailnite/mailnite-chart/releases).

### Pin a version, or go back one

Turn the automatic updates off and name the version, in `mailnite-aws.yaml` so
the next upgrade keeps it:

```yaml
autoUpdate:
  enabled: false
image:
  tag: "1.4.9"
```

```bash
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-aws.yaml
```

That is also how you return to the previous release when a new one
misbehaves. Read the release notes first: a version that changed the mail
store's format cannot be undone this way. To follow the channel again, remove
the two values and upgrade once more.

### Undo a chart or values change

Fix `mailnite-aws.yaml` and run `helm upgrade` again. Prefer that over `helm
rollback`: a rollback re-applies an old revision exactly as it was rendered,
server version included, which may be older than what the updater has
installed since. If you do roll back, run the updater right after
(**Update Mailnite now**, above).

### An update was rolled back

```bash
kubectl -n mail get statefulset mailnite -o jsonpath='{.metadata.annotations}'   # mailnite.com/update-failed: 1.4.11
kubectl -n mail get jobs                                                           # the failed run's log says why
```

The updater skips that version and waits for the next release. To try it
again after fixing the cause (a full volume, a node with no room, …):

```bash
kubectl -n mail annotate statefulset mailnite mailnite.com/update-failed-
kubectl -n mail create job --from=cronjob/mailnite-update retry
```

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `EXTERNAL-IP` stays `<pending>` | Run `kubectl -n mail describe svc mailnite-mail` and `kubectl -n kube-system logs deploy/aws-load-balancer-controller`. Usually the controller isn't running (step 3), or the number of subnets and Elastic IPs in the annotations differs. |
| PVC stays `Pending`, pod `Pending` | The EBS driver or the `gp3` class is missing: `kubectl -n mail describe pvc`, then step 3. |
| Ports 25/443 time out right after install | The pod isn't Ready yet: finish the wizard's first phase. `kubectl -n mail get pods` must show `1/1`, and then the load balancer needs a few minutes to register it. |
| Pod stuck in `Init` | Read `kubectl -n mail logs mailnite-0 -c keys`. If the mail store exists but the `mailnite-keys` Secret is missing, the pod refuses to create a new key, which could never open that store. Restore your saved key: `kubectl -n mail create secret generic mailnite-keys --from-literal=MAIL_SMK='<saved key>'` |
| No certificate is issued | `mail.<your-domain>` doesn't resolve to `$IP` yet, or it is proxied (Cloudflare's orange cloud). Fix DNS, then retry on the TLS page. |
| Mail apps disconnect every few minutes | The idle-timeout annotations from step 5 are missing. |
| Outgoing mail times out | Port 25 is limited: see [step 9](#9-sending-mail-with-amazon-ses). |
| `localhost:8480` doesn't answer | The `kubectl port-forward` from step 7 must be running. |

---

## Remove everything

```bash
helm uninstall mailnite -n mail            # removes the load balancer; keeps the volume and the key Secret, on purpose
kubectl delete namespace mail              # deletes them
kubectl get pv                             # wait until the mailnite volume is gone
eksctl delete cluster --name $CLUSTER --region $REGION
aws ec2 release-address --allocation-id $ALLOC --region $REGION
rm $HOME/.kube/mailnite-eks; unset KUBECONFIG
```

Delete Mailnite before the cluster: a cluster deleted first leaves the load
balancer and the EBS volume behind. The IAM policy
`AWSLoadBalancerControllerIAMPolicy` stays in the account for the next
cluster. Then delete the DNS records from step 6, and the SES identity if
you no longer need it.

---

## Production notes

- **Keep the volume if the namespace goes:** add `reclaimPolicy: Retain` to
  the `gp3` storage class before installing.
- **Two nodes** let a node group upgrade move the pod. Keep both in `$ZONE`
  (the node group already is):
  `eksctl scale nodegroup --cluster $CLUSTER --region $REGION --name default --nodes 2 --nodes-max 2`
- **Steady load:** `t3` instances are burstable; an `m7i.large` gives
  constant CPU.
- **A zone outage** takes the EBS volume with it. Keep the automatic backups
  in S3 (and EBS snapshots, for example with AWS Backup) so you can restore
  in another zone.
- More values, other clouds and other network setups (behind Cloudflare, a
  home lab behind NAT): the [chart's README](https://github.com/mailnite/mailnite-chart).

---

[mailnite.com](https://mailnite.com) · [Helm chart](https://github.com/mailnite/mailnite-chart) · [Releases](../../releases)

© Karagatan LLC. Mailnite is a product name of Karagatan LLC.
