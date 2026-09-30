# Install Mailnite on Azure Kubernetes Service (AKS)

This guide takes you from an empty Azure subscription to a working mail server
in about 30 minutes. You get:

- **Mailnite** on AKS, as one stateful pod with its mail store on an Azure
  managed disk.
- **Its own static public IP** on an Azure Standard Load Balancer:
  `mail.<your-domain>` for SMTP (25), submissions (465), submission (587),
  IMAPS (993), IMAP (143) and HTTPS (443).
- **Webmail** at `https://mail.<your-domain>/mail`, with a Let's Encrypt
  certificate the server obtains and renews itself.
- **Hourly automatic updates** (see [Updating](#10-updating)), plus a master key
  that is generated inside your cluster and never leaves it.

No mail relay, no ingress controller and no cert-manager are needed: the
load balancer is the mail server's edge.

> **One Azure fact before you start.** Azure blocks **outbound port 25** on
> most subscription types (Enterprise Agreement and MCA-E are the
> exceptions). Receiving mail works everywhere; to **send** to other servers
> you configure a smarthost, a mail relay service on port 587. See
> [step 8](#8-sending-mail-from-azure).

---

## Before you start

- **An Azure subscription** where you may create resource groups and AKS
  clusters.
- **A domain whose DNS you control.** For a trial, use a subdomain: with
  `test.example.com` as the mail domain, the server lives at
  `mail.test.example.com`.
- **Tools:** Azure CLI 2.48.1 or later, kubectl, and Helm 3.8 or later (for
  OCI charts).

```bash
# macOS
brew install azure-cli kubectl helm
# Linux: https://learn.microsoft.com/cli/azure/install-azure-cli · https://helm.sh/docs/intro/install/
```

Set a few names used throughout this guide:

```bash
RG=mailnite                      # resource group
CLUSTER=mailnite                 # AKS cluster name
LOCATION=westeurope              # any region that offers Standard_D2as_v5
DOMAIN=example.com               # your mail domain: the server becomes mail.$DOMAIN
```

---

## 1. Create the cluster

Use either the portal or the CLI. The two produce the same cluster.

### Option A: Azure portal

Go to **Kubernetes services → Create → Kubernetes cluster**. Choose the
standard cluster, **not** "AKS Automatic": Automatic adds policies and managed
add-ons this guide doesn't use.

| Tab | Setting | Value |
|---|---|---|
| Basics | Resource group | create new: `mailnite` |
| | Cluster preset configuration | **Dev/Test** for a trial (see [production notes](#production-notes)) |
| | Kubernetes cluster name | `mailnite` |
| | Region | your choice |
| | Availability zones | **None** |
| | AKS pricing tier | **Free** |
| | Kubernetes version | default |
| | Authentication and Authorization | **Local accounts with Kubernetes RBAC** |
| Node pools | `agentpool` → Node size | **Standard_D2as_v5** (2 vCPU, 8 GiB) |
| | Scale method / Node count | **Manual / 1** |
| Networking | Private cluster | **off** (you'll run kubectl from your machine) |
| | Network configuration | **Azure CNI Overlay** |
| | Network data plane / policy | **Cilium** (enforces the chart's NetworkPolicy) |
| | Load balancer | **Standard** |
| Integrations, Monitoring | Container registry, service mesh, Azure Policy, app routing, Container insights, Prometheus, Grafana, alerts | **off**: not needed for this guide |

Then choose **Review + create → Create**. Provisioning takes 5 to 10 minutes. If the
region reports a vCPU quota error, choose **Standard_D2s_v5** or request more
quota.

### Option B: Azure CLI

```bash
az login
az account set --subscription "<your subscription>"

az group create -n $RG -l $LOCATION
az aks create -g $RG -n $CLUSTER -l $LOCATION \
  --node-count 1 --node-vm-size Standard_D2as_v5 \
  --network-plugin azure --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 --network-dataplane cilium \
  --generate-ssh-keys
```

---

## 2. Connect kubectl

Put the cluster's credentials in their own kubeconfig file. Everything in this
shell then targets AKS, and any other cluster in your `~/.kube/config` stays
out of reach of a mistyped command.

```bash
export KUBECONFIG=$HOME/.kube/mailnite-aks
az aks get-credentials -g $RG -n $CLUSTER --file $KUBECONFIG
kubectl get nodes                 # one node, Ready
```

---

## 3. Reserve a static public IP

MX records, DNS, and your address's reputation with other mail servers all
depend on an IP that doesn't change. Create it in the cluster's **node
resource group**; AKS can use it there without any extra role assignment.

```bash
NODE_RG=$(az aks show -g $RG -n $CLUSTER --query nodeResourceGroup -o tsv)
az network public-ip create -g $NODE_RG -n mailnite-ip --sku Standard --allocation-method Static
IP=$(az network public-ip show -g $NODE_RG -n mailnite-ip --query ipAddress -o tsv)
echo $IP
```

> An IP in the node resource group is deleted with the cluster. To keep it
> across cluster rebuilds, see [production notes](#production-notes).

---

## 4. Install Mailnite

Write the values file:

```bash
cat > mailnite-azure.yaml <<EOF
domain: $DOMAIN
exposure: loadBalancer          # Mailnite gets its own public IP
platform: aks
mail:
  loadBalancer:
    ip: $IP
    annotations:
      # Azure closes idle TCP connections after 4 minutes by default; IMAP
      # IDLE (push mail in Apple Mail, Thunderbird, …) needs longer. Chart
      # 0.2.1+ sets this for platform: aks; it's kept here for older charts.
      service.beta.kubernetes.io/azure-load-balancer-tcp-idle-timeout: "30"
persistence:
  size: 32Gi                    # the whole mail store lives on this disk
EOF
```

Install the chart:

```bash
helm install mailnite oci://ghcr.io/mailnite/charts/mailnite \
  -n mail --create-namespace -f mailnite-azure.yaml
```

Helm prints the next steps; they match this guide. Watch the pod start:

```bash
kubectl -n mail get pods -w
```

The pod goes `Init:0/1` → `Running 0/1`. **`0/1` is expected** until the setup
wizard's first phase is done: readiness means "configured and serving mail".
Until then the load balancer has no backend, and connections to the IP time
out.

On its first start the pod creates the **Server Master Key**, which
encrypts the mail store, in the Secret `mailnite-keys`. **Save a copy
somewhere other than this cluster, now.** Without it, nobody can open the
mailboxes.

```bash
kubectl -n mail get secret mailnite-keys -o jsonpath='{.data.MAIL_SMK}' | base64 -d; echo
```

Check that the load balancer took your IP:

```bash
kubectl -n mail get svc mailnite-mail      # EXTERNAL-IP = $IP
```

---

## 5. Point DNS at the IP

At your DNS provider, create:

| Name | Type | Value |
|---|---|---|
| `mail.<your-domain>` | A | the static IP (`$IP`) |
| `<your-domain>` | MX | `10 mail.<your-domain>` |

On **Cloudflare**, set the A record to **DNS only** (grey cloud). Cloudflare's
proxy carries web traffic, not SMTP or IMAP.

```bash
dig +short mail.$DOMAIN            # must print $IP before the TLS step below
```

The setup wizard lists the remaining records (SPF, DKIM, DMARC, MTA-STS). It
can publish them for you if your DNS is on Cloudflare or Route 53.

---

## 6. Finish setup in the browser

```bash
kubectl -n mail port-forward statefulset/mailnite 8480:8480
```

Open **http://localhost:8480/mail/**. Keep the port-forward running while you
work in the wizard.

1. **Console password and license.** The password protects this setup
   console.
2. **Storage.** Leave the key field empty. Its hint shows `env://MAIL_SMK`:
   the master key from step 4, which the chart supplies to the server.
   Storage limits: defaults are fine.
3. **Organization and admin account.** This is your first mailbox and
   administrator.
4. **Admin console setup:**
   - **Host.** Set **Primary IP address** to your static IP (`$IP`). Auto-detection
     would find the cluster's *outbound* address, which differs from the load
     balancer's, and the DNS checklist would then be wrong.
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
     turns to "issued" within a minute or two. If you press Next before
     that, the wizard warns once; the next press continues, and issuance
     keeps retrying in the background.

---

## 7. Check that it works

```bash
kubectl -n mail get pods                   # mailnite-0 is now 1/1 Ready

# the server's own Let's Encrypt certificate, served through the Azure load balancer
openssl s_client -connect mail.$DOMAIN:443 -servername mail.$DOMAIN </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

- **Webmail:** open `https://mail.<your-domain>/mail` and sign in with the
  admin account.
- **Receiving:** send a message from Gmail or Outlook.com to your new mailbox.
  In webmail, open the message's original headers. The top `Received:` line
  must name the sender's public IP, not a `10.x` or `192.168.x` address. That
  shows the load balancer passes the real client address through, which SPF,
  DNSBL checks and rate limits depend on.
- **Mail clients:** add the account in any IMAP client, with
  `mail.<your-domain>`, IMAP on 993 (SSL/TLS) and SMTP on 587 (STARTTLS) or
  465 (SSL/TLS).

> Your home or office network may block outbound port 25 itself, so
> `telnet mail.<your-domain> 25` from your laptop can fail while the server is
> fine. Test receiving with a real message instead.

---

## 8. Sending mail from Azure

Unless your subscription is Enterprise Agreement or MCA-E, Azure drops
connections to port 25 on other servers, so delivery times out.
**Infrastructure → Outbound → Test outbound SMTP** shows it. Send through a
smarthost instead:

1. Sign up with a mail relay service (for example Azure Communication Services
   Email, SendGrid, Mailgun, Postmark or Amazon SES) and verify your domain
   there.
2. In **Infrastructure → Outbound**, set **Smarthost** to its `host:587`, and
   enter the user name and password it gives you under **Smarthost login**.
3. Add the service to your SPF record, as its documentation shows
   (typically an `include:`).

With an Enterprise Agreement or MCA-E subscription you can send directly
instead. Mail then leaves from the cluster's outbound IP, not from `$IP`, so
SPF must list that address.

---

## 9. Running it

**Back up the master key** (step 4) and keep it off the cluster. For the mail
data itself, turn on **Administration → Storage → Automatic backups**. They
can also copy to S3-compatible object storage.

**AKS node upgrades** drain the node. The chart's PodDisruptionBudget stops
the only Mailnite pod from being evicted, so with a single node a drain waits
and the upgrade can fail. For an upgrade window, relax the budget (mail pauses
briefly while the pod moves; sending servers retry):

```bash
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-azure.yaml \
  --set podDisruptionBudget.enabled=false
```

---

## 10. Updating

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
does; sending servers retry, and the load balancer keeps its IP.

### Update the chart, or change your values

Keep `mailnite-azure.yaml`: Helm applies the whole file on every upgrade. Edit it
to change a setting, then run the upgrade. The same command also brings in a
newer chart version:

```bash
helm show chart oci://ghcr.io/mailnite/charts/mailnite | grep -E '^(version|appVersion):'   # the newest chart
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-azure.yaml
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-azure.yaml --version 0.2.1   # one specific chart version
```

An upgrade never moves the server backwards: the chart reads the image the
updater has installed and keeps it whenever it is newer than the chart's own
default. What each chart version changes is on the
[chart's releases page](https://github.com/mailnite/mailnite-chart/releases).

### Pin a version, or go back one

Turn the automatic updates off and name the version, in `mailnite-azure.yaml` so
the next upgrade keeps it:

```yaml
autoUpdate:
  enabled: false
image:
  tag: "1.4.9"
```

```bash
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-azure.yaml
```

That is also how you return to the previous release when a new one
misbehaves. Read the release notes first: a version that changed the mail
store's format cannot be undone this way. To follow the channel again, remove
the two values and upgrade once more.

### Undo a chart or values change

Fix `mailnite-azure.yaml` and run `helm upgrade` again. Prefer that over `helm
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
| `EXTERNAL-IP` stays `<pending>` | Run `kubectl -n mail describe svc mailnite-mail`. The error *user supplied IP address was not found* means the IP is in another resource group or region. Create it in the node resource group (step 3), or see [production notes](#production-notes). |
| Ports 25/443 time out right after install | The pod isn't Ready yet: finish the wizard's first phase. `kubectl -n mail get pods` must show `1/1`. |
| Pod stuck in `Init` | Read `kubectl -n mail logs mailnite-0 -c keys`. If the mail store exists but the `mailnite-keys` Secret is missing, the pod refuses to create a new key, which could never open that store. Restore your saved key: `kubectl -n mail create secret generic mailnite-keys --from-literal=MAIL_SMK='<saved key>'` |
| No certificate is issued | `mail.<your-domain>` doesn't resolve to `$IP` yet, or it is proxied (Cloudflare's orange cloud). Fix DNS, then retry on the TLS page. |
| Mail apps disconnect every few minutes | The idle-timeout annotation from step 4 is missing. |
| Outgoing mail times out | Azure blocks port 25 on your subscription: see [step 8](#8-sending-mail-from-azure). |
| `localhost:8480` doesn't answer | The `kubectl port-forward` from step 6 must be running. |

---

## Remove everything

```bash
helm uninstall mailnite -n mail            # keeps the disk and the key Secret, on purpose
kubectl delete namespace mail              # deletes them
az group delete -n $RG --yes --no-wait     # the cluster, its node resource group, the IP, the disk
rm $HOME/.kube/mailnite-aks; unset KUBECONFIG
```

Then delete the DNS records from step 5.

---

## Production notes

- **Keep the IP beyond the cluster.** Create it in a resource group of its own,
  and let the cluster use it:

  ```bash
  az group create -n mailnite-ip -l $LOCATION
  az network public-ip create -g mailnite-ip -n mailnite --sku Standard --allocation-method Static
  az role assignment create --role "Network Contributor" \
    --assignee-object-id $(az aks show -g $RG -n $CLUSTER --query identity.principalId -o tsv) \
    --assignee-principal-type ServicePrincipal \
    --scope $(az group show -n mailnite-ip --query id -o tsv)
  ```

  Then add `service.beta.kubernetes.io/azure-load-balancer-resource-group: mailnite-ip`
  under `mail.loadBalancer.annotations`.
- **Two or more nodes** let a node upgrade move the pod instead of stalling.
  Azure managed disks are zonal. If you spread nodes over availability zones,
  use a zone-redundant (ZRS) storage class so the disk can follow the pod.
- **Premium SSD:** `persistence.storageClass: managed-csi-premium`. It needs a VM
  size that supports premium storage, the `s` in `D2as_v5`.
- **Standard pricing tier** for the AKS uptime SLA, and **Microsoft Entra ID**
  authentication instead of local accounts.
- More values, other clouds and other network setups (behind Cloudflare, a
  home lab behind NAT): the [chart's README](https://github.com/mailnite/mailnite-chart).

---

[mailnite.com](https://mailnite.com) · [Helm chart](https://github.com/mailnite/mailnite-chart) · [Releases](../../releases)

© Karagatan LLC. Mailnite is a product name of Karagatan LLC.
