# Install Mailnite on Google Kubernetes Engine (GKE)

This guide takes you from an empty Google Cloud project to a working mail
server in about 30 minutes. You get:

- **Mailnite** on GKE, as one stateful pod with its mail store on a Compute
  Engine persistent disk.
- **Its own static public IP** on a Google Cloud passthrough network load
  balancer: `mail.<your-domain>` for SMTP (25), submissions (465), submission
  (587), IMAPS (993), IMAP (143) and HTTPS (443).
- **Webmail** at `https://mail.<your-domain>/mail`, with a Let's Encrypt
  certificate the server obtains and renews itself.
- **Hourly automatic updates** (see [Updating](#10-updating)), plus a master key
  that is generated inside your cluster and never leaves it.

No mail relay, no ingress controller and no cert-manager are needed: the
load balancer is the mail server's edge.

> **One Google Cloud fact before you start.** Outbound **port 25** may be
> blocked on your project: most projects can't send on it, some established
> ones are exempt (Google decides which; ask about yours). Receiving works
> either way. If yours is blocked, you send through a smarthost, a mail relay
> service on port 587. See [step 8](#8-sending-mail-from-google-cloud).

---

## Before you start

- **A Google Cloud project** with billing enabled, where you may create GKE
  clusters and IP addresses.
- **A domain whose DNS you control.** For a trial, use a subdomain: with
  `test.example.com` as the mail domain, the server lives at
  `mail.test.example.com`.
- **Tools:** the gcloud CLI with its GKE auth plugin, kubectl, and Helm 3.8
  or later (for OCI charts).

```bash
# gcloud CLI: https://cloud.google.com/sdk/docs/install
gcloud components install gke-gcloud-auth-plugin kubectl
# (gcloud installed with apt or dnf: install google-cloud-cli-gke-gcloud-auth-plugin and kubectl instead)
brew install helm                # Linux: https://helm.sh/docs/intro/install/
```

Set a few names used throughout this guide:

```bash
PROJECT=my-project               # your Google Cloud project ID
CLUSTER=mailnite                 # GKE cluster name
ZONE=europe-west1-b              # the cluster's zone
REGION=${ZONE%-*}                # its region (europe-west1): the IP must live there
DOMAIN=example.com               # your mail domain: the server becomes mail.$DOMAIN
```

---

## 1. Create the cluster

Use either the console or the CLI. The two produce the same cluster: a
**Standard** cluster in one zone, with one node.

### Option A: Google Cloud console

Go to **Kubernetes Engine → Clusters → Create**. The form opens on
Autopilot; switch to the **Standard** cluster.

| Section | Setting | Value |
|---|---|---|
| Cluster basics | Name | `mailnite` |
| | Location type | **Zonal**, in your zone (e.g. `europe-west1-b`) |
| | Release channel | default (Regular) |
| default-pool | Number of nodes | **1** |
| default-pool → Nodes | Machine type | **e2-standard-2** (2 vCPU, 8 GB) |
| Networking | Enable Dataplane V2 | **on** (enforces the chart's NetworkPolicy) |
| Everything else | | defaults |

Then choose **Create**. Provisioning takes 5 to 10 minutes.

### Option B: gcloud CLI

```bash
gcloud auth login
gcloud config set project $PROJECT
gcloud services enable container.googleapis.com

gcloud container clusters create $CLUSTER --zone $ZONE \
  --num-nodes 1 --machine-type e2-standard-2 \
  --enable-dataplane-v2
```

> **Autopilot works too:** `gcloud container clusters create-auto $CLUSTER
> --region $REGION`. Google runs the nodes and bills per pod. In the
> commands below, use `--region $REGION` wherever they say `--zone $ZONE`.

---

## 2. Connect kubectl

Put the cluster's credentials in their own kubeconfig file. Everything in this
shell then targets GKE, and any other cluster in your `~/.kube/config` stays
out of reach of a mistyped command.

```bash
export KUBECONFIG=$HOME/.kube/mailnite-gke
gcloud container clusters get-credentials $CLUSTER --zone $ZONE
kubectl get nodes                 # one node, Ready
```

---

## 3. Reserve a static public IP

MX records, DNS, and your address's reputation with other mail servers all
depend on an IP that doesn't change. Reserve a **regional** address in the
cluster's region. A global address doesn't work here: the mail load
balancer is regional.

```bash
gcloud compute addresses create mailnite-ip --region $REGION
IP=$(gcloud compute addresses describe mailnite-ip --region $REGION --format='value(address)')
echo $IP
```

The address belongs to the project, not to the cluster, so it survives
cluster rebuilds.

---

## 4. Install Mailnite

Write the values file:

```bash
cat > mailnite-gke.yaml <<EOF
domain: $DOMAIN
exposure: loadBalancer          # Mailnite gets its own public IP
platform: gke
mail:
  loadBalancer:
    ip: $IP
persistence:
  storageClass: standard-rwo    # balanced persistent disk (SSD-backed)
  size: 32Gi                    # the whole mail store lives on this disk
EOF
```

`platform: gke` asks for a passthrough load balancer that keeps each
client's real IP address. GKE opens the firewall for the load balancer's
ports itself; there is nothing to add.

Install the chart:

```bash
helm install mailnite oci://ghcr.io/mailnite/charts/mailnite \
  -n mail --create-namespace -f mailnite-gke.yaml
```

Helm prints the next steps; they match this guide. Watch the pod start:

```bash
kubectl -n mail get pods -w
```

The pod goes `Init:0/1` → `Running 0/1`. **`0/1` is expected** until the setup
wizard's first phase is done: readiness means "configured and serving mail".
Until then the load balancer has no healthy backend, and connections to the
IP time out.

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
     would find the node's address, which differs from the load balancer's,
     and the DNS checklist would then be wrong.
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

# the server's own Let's Encrypt certificate, served through the Google Cloud load balancer
openssl s_client -connect mail.$DOMAIN:443 -servername mail.$DOMAIN </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

- **Webmail:** open `https://mail.<your-domain>/mail` and sign in with the
  admin account.
- **Receiving:** send a message from Gmail or Outlook.com to your new mailbox.
  In webmail, open the message's original headers. The top `Received:` line
  must name the sender's public IP, not a `10.x` address. That shows the
  load balancer passes the real client address through, which SPF, DNSBL
  checks and rate limits depend on.
- **Mail clients:** add the account in any IMAP client, with
  `mail.<your-domain>`, IMAP on 993 (SSL/TLS) and SMTP on 587 (STARTTLS) or
  465 (SSL/TLS).

> Your home or office network may block outbound port 25 itself, so
> `telnet mail.<your-domain> 25` from your laptop can fail while the server is
> fine. Test receiving with a real message instead.

---

## 8. Sending mail from Google Cloud

Test first: **Infrastructure → Outbound → Test outbound SMTP** dials a real
mail server from inside the pod. If your project is one of the exempt ones
the test succeeds and you can send directly (mail then leaves from the
node's address, not from `$IP`, so your SPF record must list it too). For
most projects Google Cloud drops connections to port 25 and the test times
out; ports 587 and 465 are always open, so you send through a smarthost:

1. Sign up with a mail relay service and verify your domain there. Google
   suggests SendGrid, Mailgun and Mailjet; Amazon SES and Postmark work too.
2. In **Infrastructure → Outbound**, set **Smarthost** to its `host:587`, and
   enter the user name and password it gives you under **Smarthost login**.
3. Add the service to your SPF record, as its documentation shows
   (typically an `include:`).

---

## 9. Running it

**Back up the master key** (step 4) and keep it off the cluster. For the mail
data itself, turn on **Administration → Storage → Automatic backups**. They
can also copy to S3-compatible object storage.

**GKE upgrades nodes automatically.** The chart's PodDisruptionBudget holds
the only Mailnite pod in place for up to an hour; then GKE moves it anyway,
and mail pauses for a minute while it restarts (sending servers retry). Pick
a quiet hour for that with a maintenance window:

```bash
gcloud container clusters update $CLUSTER --zone $ZONE \
  --maintenance-window-start 2026-01-01T02:00:00Z \
  --maintenance-window-end 2026-01-01T06:00:00Z \
  --maintenance-window-recurrence FREQ=DAILY        # 02:00–06:00 UTC, every day
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

Keep `mailnite-gke.yaml`: Helm applies the whole file on every upgrade. Edit it
to change a setting, then run the upgrade. The same command also brings in a
newer chart version:

```bash
helm show chart oci://ghcr.io/mailnite/charts/mailnite | grep -E '^(version|appVersion):'   # the newest chart
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-gke.yaml
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-gke.yaml --version 0.2.1   # one specific chart version
```

An upgrade never moves the server backwards: the chart reads the image the
updater has installed and keeps it whenever it is newer than the chart's own
default. What each chart version changes is on the
[chart's releases page](https://github.com/mailnite/mailnite-chart/releases).

### Pin a version, or go back one

Turn the automatic updates off and name the version, in `mailnite-gke.yaml` so
the next upgrade keeps it:

```yaml
autoUpdate:
  enabled: false
image:
  tag: "1.4.9"
```

```bash
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n mail -f mailnite-gke.yaml
```

That is also how you return to the previous release when a new one
misbehaves. Read the release notes first: a version that changed the mail
store's format cannot be undone this way. To follow the channel again, remove
the two values and upgrade once more.

### Undo a chart or values change

Fix `mailnite-gke.yaml` and run `helm upgrade` again. Prefer that over `helm
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
| `EXTERNAL-IP` stays `<pending>` | Run `kubectl -n mail describe svc mailnite-mail`. The address must be **regional**, in the cluster's region (`$REGION`), not global. Create it as in step 3. |
| Ports 25/443 time out right after install | The pod isn't Ready yet: finish the wizard's first phase. `kubectl -n mail get pods` must show `1/1`. |
| Pod stuck in `Init` | Read `kubectl -n mail logs mailnite-0 -c keys`. If the mail store exists but the `mailnite-keys` Secret is missing, the pod refuses to create a new key, which could never open that store. Restore your saved key: `kubectl -n mail create secret generic mailnite-keys --from-literal=MAIL_SMK='<saved key>'` |
| No certificate is issued | `mail.<your-domain>` doesn't resolve to `$IP` yet, or it is proxied (Cloudflare's orange cloud). Fix DNS, then retry on the TLS page. |
| Outgoing mail times out | Port 25 is blocked on your project: see [step 8](#8-sending-mail-from-google-cloud). |
| kubectl: `gke-gcloud-auth-plugin` not found | Install the plugin (see [Before you start](#before-you-start)), then run `get-credentials` again. |
| `helm upgrade` fails with *conflicts with "Go-http-client"* (or *"mailnite-k8s"*) on the image fields | Helm 4 applies server-side, and the hourly updater had written the image before. Run the upgrade once with `--force-conflicts`. From Mailnite 1.5.2 the updater hands the fields back to Helm after every run, so it does not recur. |
| `localhost:8480` doesn't answer | The `kubectl port-forward` from step 6 must be running. |

---

## Remove everything

```bash
helm uninstall mailnite -n mail            # keeps the disk and the key Secret, on purpose
kubectl delete namespace mail              # deletes them
kubectl get pv                             # wait until the mailnite volume is gone
gcloud container clusters delete $CLUSTER --zone $ZONE --quiet
gcloud compute addresses delete mailnite-ip --region $REGION --quiet
rm $HOME/.kube/mailnite-gke; unset KUBECONFIG
```

Delete the namespace before the cluster. A cluster deleted first leaves the
persistent disk behind (`gcloud compute disks list` shows it). Then delete
the DNS records from step 5.

---

## Production notes

- **SSD persistent disk:** `persistence.storageClass: premium-rwo`.
- **Two or more nodes** let a node upgrade move the pod without waiting on
  the disruption budget. A persistent disk is zonal; keep the nodes in its
  zone (a zonal cluster does).
- **Webmail on your site's hostname** (`https://<your-domain>/mail`) through
  a GKE Gateway you already run: set `gateway.name` and `gateway.namespace`.
  The chart adds the GCPBackendPolicy (long downloads) and the
  HealthCheckPolicy the Google load balancer needs.
- More values, other clouds and other network setups (behind Cloudflare, a
  home lab behind NAT): the [chart's README](https://github.com/mailnite/mailnite-chart).

---

[mailnite.com](https://mailnite.com) · [Helm chart](https://github.com/mailnite/mailnite-chart) · [Releases](../../releases)

© Karagatan LLC. Mailnite is a product name of Karagatan LLC.
