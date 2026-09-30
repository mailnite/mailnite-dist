# Install Mailnite on your own Kubernetes cluster

This guide is for a cluster you run yourself (bare metal, a home lab, a
cluster behind Cloudflare) that already serves websites through a Gateway
API gateway. In about 30 minutes you get:

- **Mailnite** in its own namespace, as one stateful pod on your storage.
- **Webmail** at `https://<your-domain>/mail`, routed through your existing
  Gateway. Your site keeps `/`.
- **The mail ports** (25, 465, 587, 993, 143) and `mail.<your-domain>` on a
  small VDS running **mailrelay**. The cluster needs no inbound port:
  Mailnite dials out to the relay. The VDS stores no mail and no keys, so
  losing it loses nothing.
- **Automatic updates** for the server and the relay (see
  [Updating](#9-updating)), plus a master key that is generated inside your
  cluster and never leaves it.

Why a relay: Cloudflare's proxy carries web traffic only, not SMTP or IMAP,
and a NAT or a home connection gives you no public port 25.

| Your cluster | The internet reaches mail through | Setup |
|---|---|---|
| Behind Cloudflare or NAT, websites through a Gateway | a mailrelay on a small VDS | **this guide** |
| A home lab behind NAT, no gateway | the same relay, for mail and webmail | [below](#no-gateway-a-home-lab-behind-nat) |
| A router with its own static public IP | router port-forwards to node ports | [below](#a-router-with-its-own-public-ip) |
| Public addresses on the cluster (MetalLB, k3s ServiceLB) | the cluster's own load balancer | [below](#public-addresses-on-the-cluster) |
| GKE, AKS or EKS | a cloud load balancer | [Google Cloud](gcp.md) · [Azure](azure.md) · [AWS](aws.md) |

---

## Before you start

- **The cluster:** kubectl access, Helm 3.8 or later (for OCI charts), and a
  storage class (`kubectl get storageclass`; the default one is used unless
  you name another).
- **A Gateway API gateway** that already serves your site over HTTPS for
  `<your-domain>` (Envoy Gateway, Cilium, Istio, NGINX Gateway Fabric,
  Traefik, …).
- **A VDS:** any small Linux server with a static public IPv4 address, and
  SSH as root or as a user with sudo. 1 vCPU and 1 GB of RAM are plenty. Its
  provider must allow port 25 (many block it on new accounts until you ask)
  and let you set reverse DNS.
- **A domain whose DNS you control.**

Set a few names used throughout this guide:

```bash
NS=mail                          # the namespace: one Mailnite per namespace
DOMAIN=example.com               # your mail domain: the server becomes mail.$DOMAIN
GATEWAY=external-https           # your Gateway (kubectl get gateway -A) …
GATEWAY_NS=gateway-infra         # … and its namespace
VDS_IP=203.0.113.10              # the VDS's public IPv4 address
```

---

## 1. Check the Gateway

Mailnite's route lives in its own namespace, and attaches to your Gateway
from there. List the Gateway's listeners:

```bash
kubectl -n $GATEWAY_NS get gateway $GATEWAY -o jsonpath='{range .spec.listeners[*]}{.name}{"  "}{.protocol}/{.port}{"  host="}{.hostname}{"  routes from="}{.allowedRoutes.namespaces.from}{"\n"}{end}'
```

You need an HTTPS listener that covers `<your-domain>` and takes routes from
other namespaces:

- **`host=`** empty or `<your-domain>`. A wildcard such as `*.example.com`
  doesn't cover `example.com` itself.
- **`routes from=All`**, or `Selector` with a matching label on the `$NS`
  namespace. Empty or `Same` means only routes in the Gateway's own
  namespace attach. Change the listener to:

  ```yaml
  allowedRoutes:
    namespaces:
      from: All
  ```

---

## 2. Prepare the VDS

The wizard installs the relay for you in step 5. Before that, the VDS needs:

- **Open ports** in the provider's firewall, and in ufw or firewalld if
  the VDS runs one:
  - TCP 25, 465, 587, 993, 143 (mail);
  - TCP 80 and 443 (certificate checks and webmail);
  - TCP 8443 (the relay's control port; add UDP 8443 for the QUIC
    transport);
  - SSH, reachable from your cluster.
- **Outbound port 25.** Mail leaves from the VDS's address. If the provider
  blocks it, ask them to lift the block, or use a smarthost later.
- **Reverse DNS (PTR)** for `$VDS_IP`: `mail.<your-domain>`. Set it in the
  provider's panel.

---

## 3. Install Mailnite

Write the values file:

```bash
cat > mailnite-values.yaml <<EOF
domain: $DOMAIN
exposure: relay                 # mail rides a mailrelay on the VDS
gateway:
  name: $GATEWAY                # webmail at https://$DOMAIN/mail through it
  namespace: $GATEWAY_NS
  # sectionName: https          # only to pin one listener
persistence:
  size: 32Gi                    # the whole mail store lives on this volume
  # storageClass: local-path    # omit for the cluster's default class
EOF
```

Install the chart:

```bash
helm install mailnite oci://ghcr.io/mailnite/charts/mailnite \
  -n $NS --create-namespace -f mailnite-values.yaml
```

Helm prints the next steps; they match this guide. Watch the pod start:

```bash
kubectl -n $NS get pods -w
```

The pod goes `Init:0/1` → `Running 0/1`. **`0/1` is expected** until the setup
wizard's first phase is done: readiness means "configured and serving mail".
Until then `https://<your-domain>/mail` answers with the gateway's error
page (often 503).

On its first start the pod creates the **Server Master Key**, which
encrypts the mail store, in the Secret `mailnite-keys`. **Save a copy
somewhere other than this cluster, now.** Without it, nobody can open the
mailboxes.

```bash
kubectl -n $NS get secret mailnite-keys -o jsonpath='{.data.MAIL_SMK}' | base64 -d; echo
```

Check that the Gateway accepted the route:

```bash
kubectl -n $NS get httproute mailnite-mail-path \
  -o jsonpath='{range .status.parents[*].conditions[*]}{.type}={.status} {end}{"\n"}'
# Accepted=True ResolvedRefs=True
```

---

## 4. Point DNS at the VDS

At your DNS provider, create:

| Name | Type | Value |
|---|---|---|
| `mail.<your-domain>` | A | the VDS's IP (`$VDS_IP`) |
| `<your-domain>` | MX | `10 mail.<your-domain>` |

On **Cloudflare**, set the `mail` record to **DNS only** (grey cloud).
Cloudflare's proxy carries web traffic, not SMTP or IMAP. `<your-domain>`
itself keeps pointing at your site, proxied or not.

The relay's tunnel dials `mail.<your-domain>`, so the record must resolve
before step 5:

```bash
dig +short mail.$DOMAIN            # must print $VDS_IP
```

The setup wizard lists the remaining records (SPF, DKIM, DMARC, MTA-STS). It
can publish them for you if your DNS is on Cloudflare or Route 53.

---

## 5. Finish setup in the browser

```bash
kubectl -n $NS port-forward statefulset/mailnite 8480:8480
```

Open **http://localhost:8480/mail/**. Keep the port-forward running while you
work in the wizard.

1. **Console password and license.** The password protects this setup
   console.
2. **Storage.** Leave the key field empty. Its hint shows `env://MAIL_SMK`:
   the master key from step 3, which the chart supplies to the server.
   Storage limits: defaults are fine.
3. **Organization and admin account.** This is your first mailbox and
   administrator.
4. **Admin console setup:**
   - **Host.** Set **Primary IP address** to the VDS's IP (`$VDS_IP`). With a
     relay, that is your public address, for mail in and out.
   - **DNS / Domains.** Publish or copy the listed records.
   - **Mail relay.** Choose **Relay: automatic setup**. Enter the **VDS
     host** (`$VDS_IP`), the **SSH port** and **SSH user** (root, or a user
     with sudo), and a password or an SSH key. Choose **Test connection**,
     then **Install & pair**. The console installs mailrelay as a system
     service, pairs it with mutual TLS, and keeps it up to date.
   - **TLS.** Choose Let's Encrypt and tick **I accept the Let's Encrypt
     Subscriber Agreement**; Save stays disabled until you do. The check
     reaches the server through the relay.
   - **Ports.** Choose **Relay** for both groups, mail and web. The VDS then
     takes the standard ports and tunnels them here. Links in emails
     (password resets, invitations) point at `https://mail.<your-domain>/mail`,
     which only the relay serves.
   - **Servers.** Every server shows as bound on the relay's address, and
     the certificate line turns to "issued" within a minute or two.

---

## 6. Trust your gateway for client addresses

Requests to `https://<your-domain>/mail` arrive through Cloudflare and your
Gateway. Until you say whom to trust, Mailnite sees the gateway's address
on every request, and its rate limits and login log see one client. In the
admin console, open **Security → Reverse proxy client addresses**:

1. Turn on **Use a forwarding header from explicitly trusted proxies**.
2. Sign in once through `https://<your-domain>/mail`. **Recommended from
   observed traffic** then proposes your gateway's address, and recognizes
   Cloudflare. Check that the address is your gateway's, choose **Use
   detected chain**, and save.

   You can also fill the fields in yourself: **Trusted proxy IPs or CIDRs**
   is your gateway's own network, not the whole pod CIDR of the cluster.
   **Upstream proxy / CDN** is Cloudflare when it sits in front.

Traffic through the relay needs nothing here: the relay hands over each
client's address itself.

**With Cloudflare in front:**
- A rule that adds a `Content-Security-Policy` header (a Transform Rule or a
  Worker) must skip `/mail/`. Two policies intersect and break webmail.
- Keep WebSockets on (it is the default); webmail runs over one.
- Free and Pro plans cap an upload at 100 MB. For bigger imports use
  `https://mail.<your-domain>/mail`, which goes through the relay.

---

## 7. Check that it works

```bash
kubectl -n $NS get pods                    # mailnite-0 is now 1/1 Ready
curl -sI https://$DOMAIN/mail/ | head -1   # 200, through your Gateway

# the server's own Let's Encrypt certificate, served through the relay
openssl s_client -connect mail.$DOMAIN:443 -servername mail.$DOMAIN </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

- **Webmail:** sign in at `https://<your-domain>/mail`, and at
  `https://mail.<your-domain>/mail`.
- **Receiving:** send a message from Gmail or Outlook.com to your new mailbox.
  In webmail, open the message's original headers. The top `Received:` line
  must name the sender's public IP: the relay passes the real client address
  through the tunnel, and SPF, DNSBL checks and rate limits depend on it.
- **Sending:** **Infrastructure → Outbound → Test outbound SMTP**. With a
  relay active, mail leaves through it ("Deliver outbound mail through the
  relay's egress" is on by default), from `$VDS_IP`: the address your SPF
  and PTR records name. If the VDS provider blocks port 25, set a
  **Smarthost** on the same page.
- **Mail clients:** add the account in any IMAP client, with
  `mail.<your-domain>`, IMAP on 993 (SSL/TLS) and SMTP on 587 (STARTTLS) or
  465 (SSL/TLS).

---

## 8. Running it

**Back up the master key** (step 3) and keep it off the cluster. For the mail
data itself, turn on **Administration → Storage → Automatic backups**. They
can also copy to S3-compatible object storage.

**Losing the VDS loses nothing.** Rent a new one, point `mail.<your-domain>`
at it, set its reverse DNS, and run **Infrastructure → Mail relay → Relay:
automatic setup** again. One VDS serves one Mailnite install: a relay owns
its IP's port 25.

**Draining the node** that runs Mailnite waits on the chart's
PodDisruptionBudget. For maintenance, relax it with
`--set podDisruptionBudget.enabled=false` on a `helm upgrade`.

---

## 9. Updating

Three things change over time, each with its own command, and none of them
undoes another: **Mailnite** (the server image), **the chart**, and **your
values file**.

### What is running

```bash
helm list -n $NS                                    # the chart version; APP VERSION is the chart's default, not always what runs
kubectl -n $NS get statefulset mailnite \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'   # the server version actually running
kubectl -n $NS exec mailnite-0 -- /app/mailnite version   # the same, asked from inside
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
kubectl -n $NS get cronjob mailnite-update          # LAST SCHEDULE
kubectl -n $NS get jobs                             # the last run; failed runs are kept
kubectl -n $NS logs job/<job-name>                  # e.g. "up to date: 1.4.10 (channel v1.4.10)"
```

### Update Mailnite now

The same thing, without waiting for the hour:

```bash
kubectl -n $NS create job --from=cronjob/mailnite-update update-now
kubectl -n $NS logs -f job/update-now
kubectl -n $NS delete job update-now
```

The pod restarts once with the new image. Mail pauses for a moment while it
does; sending servers retry, and the relay reconnects on its own.

### Update the chart, or change your values

Keep `mailnite-values.yaml`: Helm applies the whole file on every upgrade. Edit it
to change a setting, then run the upgrade. The same command also brings in a
newer chart version:

```bash
helm show chart oci://ghcr.io/mailnite/charts/mailnite | grep -E '^(version|appVersion):'   # the newest chart
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n $NS -f mailnite-values.yaml
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n $NS -f mailnite-values.yaml --version 0.2.1   # one specific chart version
```

An upgrade never moves the server backwards: the chart reads the image the
updater has installed and keeps it whenever it is newer than the chart's own
default. What each chart version changes is on the
[chart's releases page](https://github.com/mailnite/mailnite-chart/releases).

### Pin a version, or go back one

Turn the automatic updates off and name the version, in `mailnite-values.yaml` so
the next upgrade keeps it:

```yaml
autoUpdate:
  enabled: false
image:
  tag: "1.4.9"
```

```bash
helm upgrade mailnite oci://ghcr.io/mailnite/charts/mailnite -n $NS -f mailnite-values.yaml
```

That is also how you return to the previous release when a new one
misbehaves. Read the release notes first: a version that changed the mail
store's format cannot be undone this way. To follow the channel again, remove
the two values and upgrade once more.

### Undo a chart or values change

Fix `mailnite-values.yaml` and run `helm upgrade` again. Prefer that over `helm
rollback`: a rollback re-applies an old revision exactly as it was rendered,
server version included, which may be older than what the updater has
installed since. If you do roll back, run the updater right after
(**Update Mailnite now**, above).

### An update was rolled back

```bash
kubectl -n $NS get statefulset mailnite -o jsonpath='{.metadata.annotations}'   # mailnite.com/update-failed: 1.4.11
kubectl -n $NS get jobs                                                           # the failed run's log says why
```

The updater skips that version and waits for the next release. To try it
again after fixing the cause (a full volume, a node with no room, …):

```bash
kubectl -n $NS annotate statefulset mailnite mailnite.com/update-failed-
kubectl -n $NS create job --from=cronjob/mailnite-update retry
```

### The relay

The relay on the VDS follows the same release channel on its own (the
automatic setup ticked **Keep the relay up to date automatically**).
**Monitoring → Servers** shows its version and status, and updates or
restarts it on demand. The tunnel reconnects by itself after a relay restart;
mail waits a few seconds and continues.

---

## Other setups

### No gateway: a home lab behind NAT

Leave the `gateway:` block out of the values file and skip steps 1 and 6.
Everything public then rides the relay: mail, and webmail at
`https://mail.<your-domain>/mail`. The router needs no port forward, because
Mailnite dials out.

### A router with its own public IP

The router forwards the mail ports to fixed node ports, and its port 443
already goes to your Gateway:

```yaml
domain: example.com
exposure: nodePort
mail:
  ports:
    https: { enabled: false }   # 443 belongs to the gateway here
gateway:
  name: external-https
  namespace: gateway-infra
```

- **Router:** forward 25→30025, 465→30465, 587→30587, 993→30993 and
  143→30143, to any node.
- **DNS:** `mail.<your-domain>` A → the router's public IP; ask the ISP for
  the reverse DNS. The Gateway serves `mail.<your-domain>` as well as
  `<your-domain>/mail`, so its listener and certificate must cover
  `mail.<your-domain>` (a wildcard does).
- **Wizard:** Mail relay **No relay — reachable directly**; Ports: keep the
  preset high port numbers. The certificate is checked over HTTP through the
  Gateway: `http://mail.<your-domain>/.well-known/acme-challenge/` must reach
  it (a redirect to https is fine).

A public IP has only one port 25. A second mail server behind the same
router needs the relay setup from this guide.

### Public addresses on the cluster

With MetalLB or k3s ServiceLB handing out public IPs, Mailnite can have its
own: `exposure: loadBalancer` and `mail.loadBalancer.ip: <a public IP from
the pool>`. From there the steps match the cloud guides (for example
[Azure, from step 5](azure.md#5-point-dns-at-the-ip)): DNS to that IP, and
**No relay — reachable directly** in the wizard.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| HTTPRoute `Accepted=False`, reason `NotAllowedByListeners` | The listener takes routes only from its own namespace: allow others (step 1). |
| HTTPRoute `Accepted=False`, reason `NoMatchingListenerHostname` | No listener covers `<your-domain>`. A `*.example.com` listener doesn't match `example.com`. |
| `https://<your-domain>/mail` answers 503 | The pod isn't Ready yet: finish the wizard's first phase. `kubectl -n $NS get pods` must show `1/1`. |
| Webmail stays blank; the browser console reports Content-Security-Policy errors | A Cloudflare rule adds its own policy: exclude `/mail/`. |
| Relay: **Test connection** fails | The cluster can't reach SSH on the VDS: check the VDS firewall. If SSH isn't on port 22, add its port to `networkPolicy.egress.relayPorts`. |
| The relay is installed but the tunnel stays down | `mail.<your-domain>` doesn't resolve to `$VDS_IP` yet, or port 8443 is closed on the VDS. |
| No certificate is issued | `mail.<your-domain>` is proxied (Cloudflare's orange cloud) or points elsewhere, or ports 80 and 443 are closed on the VDS. |
| Outgoing mail times out | The VDS provider blocks port 25: ask them to lift it, or set a smarthost in **Infrastructure → Outbound**. |
| Pod stuck in `Init` | Read `kubectl -n $NS logs mailnite-0 -c keys`. If the mail store exists but the `mailnite-keys` Secret is missing, the pod refuses to create a new key, which could never open that store. Restore your saved key: `kubectl -n $NS create secret generic mailnite-keys --from-literal=MAIL_SMK='<saved key>'` |
| `helm upgrade` fails with *conflicts with "Go-http-client"* (or *"mailnite-k8s"*) on the image fields | Helm 4 applies server-side, and the hourly updater had written the image before. Run the upgrade once with `--force-conflicts`. From Mailnite 1.5.2 the updater hands the fields back to Helm after every run, so it does not recur. |
| `localhost:8480` doesn't answer | The `kubectl port-forward` from step 5 must be running. |

---

## Remove everything

```bash
helm uninstall mailnite -n $NS              # keeps the volume and the key Secret, on purpose
kubectl delete namespace $NS                # deletes them; with a Delete reclaim policy, the mail too
```

On the VDS:

```bash
curl -fsSL https://get.mailnite.com/relay | sudo bash -s -- --uninstall --purge
```

A relay installed without root lives in `~/.mailrelay`: delete that folder
and its lines in `crontab -e`. Then delete the DNS records from step 4, and
undo the listener change from step 1 if you made one.

---

## Production notes

- **Storage.** Node-local volumes (local-path, OpenEBS hostpath) tie the pod
  to one node: if that node's disk dies, the mail survives only in your
  backups. Keep the automatic backups in S3-compatible storage, and prefer a
  storage class with `reclaimPolicy: Retain`.
- **NetworkPolicy.** The chart ships one; it takes effect only when the CNI
  enforces policies (Cilium, Calico, …). If you run SSH on the VDS on
  another port, or reach S3, LDAP or a smarthost on unusual ports, add them
  under `networkPolicy.egress`.
- More values and setups: the [chart's README](https://github.com/mailnite/mailnite-chart).

---

[mailnite.com](https://mailnite.com) · [Helm chart](https://github.com/mailnite/mailnite-chart) · [Releases](../../releases)

© Karagatan LLC. Mailnite is a product name of Karagatan LLC.
