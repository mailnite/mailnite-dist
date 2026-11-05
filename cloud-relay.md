# Run the mail relay on a cloud machine the console makes

Mail needs a public address that the world can reach on port 25 and friends. A
home or office server has none, and renting a whole VDS for the relay means
installing and babysitting one more machine. Mailnite's console can make that
machine for you — at Google Cloud, AWS or Azure — and keep it running:

- **A reserved public address** that is yours for as long as you want it:
  `mail.<your-domain>` points at it, the reverse DNS (PTR) names it, and every
  machine under it is disposable. The address is the reputation; the machine
  is a commodity.
- **A Spot machine by default** — the cloud's spare capacity at a steep
  discount — with a Standard machine as the fallback when no Spot capacity is
  there, and a way back to Spot when it returns.
- **Recovery without you**: a machine that is stopped is started again; one
  that is taken, gone or unreachable is replaced by a successor in the same
  region, paired, and the address moved onto it — the relay on the machine
  tells the console before the cloud takes it, so the successor boots while
  the old one still serves.
- **The honest bill**: the console shows the month as it stands — the
  machine line, the address line, the disk — and proposes a cheaper machine of
  the same size in the same region when one exists. Your address never moves
  for a saving; a change of region or cloud is a decision you make yourself.

Your server stays where it is. It dials out to the relay over mutual TLS; the
relay is a byte pump with no storage and never sees plaintext mail (STARTTLS
and implicit TLS run end to end between the remote peer and your server).

> **One fact about outbound port 25 before you choose a cloud.** Google Cloud
> blocks it on every machine. Azure blocks it on pay-as-you-go subscriptions
> (Enterprise Agreement and MCA-E subscriptions can ask Microsoft to open it).
> AWS throttles it until you ask once to have the limit removed (the *Request
> to remove email sending limitations* form, usually answered within a day).
> Receiving mail works everywhere. On a cloud that blocks it, your server sends
> through a smarthost (Administration → Outbound) and the relay carries inbound
> mail only; the console refuses to make the machine otherwise, and says why.

---

## Before you start

- **A Mailnite server** past its first-run setup, with a primary domain.
  Connecting the domain's DNS provider (Infrastructure → DNS) lets the console
  publish the `mail.<your-domain>` A record itself; without it, the console
  tells you what to publish and waits.
- **An account at the cloud** you pick, prepared as the next section says.
- **Where the relay should live.** The console lists the regions nearest to
  you first (from your pin, or from the browser's time zone) and prices every
  machine type **all-in**: the machine in its model, the address line, a 10 GB
  disk.

The secrets go in once and never come back to the page.

## What each cloud needs

### Google Cloud

The account is a **service account's JSON key** in the project the machine
goes to.

1. **IAM & Admin → Service accounts → Create** `mailnite-console` with the
   role **Compute Admin** on the project → *Keys → Add key → Create new key →
   JSON*.
2. Enable the **Compute Engine API**, and the **Cloud Billing API** for the
   prices (without it the machine is still made; the page shows no price).
3. The project's `default` VPC network, which every new project has; if yours
   was deleted, make a network named `default` (automatic subnets) once.
4. **Quotas** per region (*IAM & Admin → Quotas*): *CPUs* (the relay takes 2;
   Spot counts there unless the project has a *Preemptible CPUs* quota),
   *Static IP addresses* and *In-use IP addresses* — one each.

The console makes the firewall rule `mailnite-relay` on the `default` network,
a regional static external address, and a Debian 12 machine. The PTR is set on
the machine's access config once `mail.<your-domain>` resolves to the address.

### AWS

The account is an **access key of an IAM user** made for the console alone.
The *Home region* on the form is where the key's Test runs; the machine goes to
whatever region you pick.

1. **IAM → Users → Create user** `mailnite-console` (no console access) →
   *Attach policies directly* → *Create policy* → JSON, the policy below.
   Then *Security credentials → Create access key → Application running
   outside AWS*.
2. **A default VPC** in the region you use (accounts made since 2013 have one;
   otherwise `aws ec2 create-default-vpc --region <region>` once).
3. **Quotas** in vCPUs per region (Service Quotas → Amazon EC2), the relay
   takes 2: *Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) instances*,
   *All Standard (A, C, D, H, I, M, R, T, Z) Spot Instance Requests*, and
   *EC2-VPC Elastic IPs* (a migration holds a second one for a day). New
   accounts start low; the key's Test shows the two vCPU quotas.
4. **Outbound port 25**: send the limit-removal form before the first machine.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MailniteRelay",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeRegions", "ec2:DescribeAvailabilityZones", "ec2:DescribeInstanceTypes", "ec2:DescribeSpotPriceHistory",
        "ec2:DescribeImages", "ec2:DescribeVpcs", "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups", "ec2:CreateSecurityGroup", "ec2:AuthorizeSecurityGroupIngress", "ec2:RevokeSecurityGroupIngress",
        "ec2:DescribeAddresses", "ec2:AllocateAddress", "ec2:AssociateAddress", "ec2:DisassociateAddress", "ec2:ReleaseAddress",
        "ec2:ModifyAddressAttribute", "ec2:ResetAddressAttribute",
        "ec2:RunInstances", "ec2:DescribeInstances", "ec2:StartInstances", "ec2:StopInstances", "ec2:TerminateInstances",
        "ec2:GetConsoleOutput", "ec2:CreateTags",
        "ec2:GetSpotPlacementScores", "pricing:GetProducts", "servicequotas:GetServiceQuota"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SpotRoleOnce",
      "Effect": "Allow",
      "Action": "iam:CreateServiceLinkedRole",
      "Resource": "*",
      "Condition": { "StringEquals": { "iam:AWSServiceName": "spot.amazonaws.com" } }
    }
  ]
}
```

The second statement lets the first Spot machine create AWS's Spot
service-linked role when the account has none yet. `ec2:ModifyAddressAttribute`
is the reverse DNS record on the Elastic IP; `ec2:StartInstances` restarts a
stopped machine in place; `ec2:GetSpotPlacementScores` scores the
cheaper-machine proposals. The console makes the security group
`mailnite-relay` in the default VPC, an Elastic IP, and a Debian 12 machine
with a 10 GB gp3 disk.

### Azure

The account is a **service principal** with rights over one resource group,
`mailnite` unless you name another; everything the console makes lives there.

1. **Resource group** `mailnite` in any region (the console makes it when the
   service principal may create resource groups; otherwise make it once).
2. **Microsoft Entra ID → App registrations → New registration**
   `mailnite-console` → *Certificates & secrets → New client secret*. Note the
   tenant ID, the application (client) ID and the secret's value. The secret
   expires: renew it on the Cloud providers page before it does.
3. **Roles**: **Contributor** on the resource group, and **Reader** on the
   subscription (for the Test and the region list).
4. **Resource providers**: register **Microsoft.Compute** and
   **Microsoft.Network** if the subscription is new.
5. **Quotas** in vCPUs per region (*Usage + quotas*, Compute), the relay takes
   2: *Total Regional vCPUs*, the size's family, and for Spot *Total Regional
   Spot vCPUs* — **often 0 on a new subscription**: ask for 10 or more.

The console makes the network security group `mailnite-relay-<region>`, a
virtual network for the relays, a Standard static public IP whose reverse DNS
it sets, and a Debian 12 machine. A Spot machine is evicted at any price, with
at most 30 seconds' notice.

## 1. Add the cloud account

Infrastructure → **Cloud providers** → the cloud's kind, a name, the
credentials → **Save**, then **Test**. The form states that cloud's
outbound-25 fact beside the kind; Test answers with what the cloud says about
the key. The same page holds one SSH key pair the console uses for every
machine it makes; generate it here once, before the first machine.

## 2. Make the machine

Infrastructure → **Mail relay** → *How does the internet reach this server?*
→ **Through a relay on a cloud machine made for me**. Pick the account, a
region (nearest first), a machine type (the all-in price beside each) and the
model — **Spot preferred** or **Standard only** — then **Create**. The job
runs on the server and shows every step as it happens:

1. the address is reserved and recorded first (a crash leaves a known address,
   never an unknown bill);
2. the firewall is written: the mail ports, the web ports and the relay's
   control port, nothing else;
3. the machine boots with a startup script that only creates the console's
   SSH login — no relay material ever enters cloud metadata;
4. the console installs the relay over SSH with mutual-TLS material minted on
   your server, and your server pairs with it;
5. the A record of `mail.<your-domain>` is published through your DNS
   provider;
6. once the name resolves, the PTR is set at the cloud and checked.

When no zone of the region has Spot capacity, the console makes a Standard
machine instead and marks it as the fallback; it proposes Spot again when Spot
is back. Burstable types — AWS t3, t3a and t4g, Google's shared-core e2-micro,
e2-small and e2-medium, Azure's B-series — are the cheapest and are offered
with a note: they wait for CPU under load, and the console never switches to
one on its own.

## 3. Check that it works

- The **Address** card on the Mail relay page shows the address, when it was
  reserved, and the PTR state; the **Machine** card shows the provider, the
  region and zone, the type and model, the price and the uptime.
- Monitoring → **Servers** shows the relay connected, the machine's facts and
  the operations.
- From any machine: `dig -x <address>` answers `mail.<your-domain>`, and
  `dig mail.<your-domain>` answers the address.
- Point a mail client at `mail.<your-domain>`: IMAP on 993, submission on 587
  or 465, exactly as with a VDS you installed by hand.

## 4. What happens when the machine goes

Every thirty seconds the console looks at the machine, the tunnel and the
address, and at once when the relay says its machine is being taken (Google
preemption, AWS Spot interruption, Azure eviction):

| when | what the console does |
|---|---|
| the tunnel is down and the cloud says the machine is stopped | starts it again (three minutes to come back) |
| the machine is gone, stays stopped, is unreachable for five minutes, or its relay said it is leaving | makes a successor in the same region, installs and pairs it at its own address, **moves the reserved address onto it**, deletes the old machine |
| two replacements fail in a row | one notification, and the next try waits two minutes, doubling to thirty |

Every step is a line in the Servers card's history and a note in the bell.
**Replace machines automatically** (on the Servers card, on by default) is the
kill switch: off, the console only alerts, and *Restart in place* and
*Replace machine* by hand still work. **Migrate** moves the relay to another
region or cloud — a new address, the old one kept a day and then released;
that is a decision about reputation, so it is yours alone.

## 5. Cheaper machines

Every hour the console compares the machine with its region's catalog. Every
candidate keeps the address — another type of about the same size, Spot or
Standard — so only the machine line of the bill is compared. Under *Spot
preferred* a Standard machine is proposed back to the cheapest Spot type as
soon as one exists; under *Standard only* Spot is never proposed. Proposals
appear on Monitoring → Servers with **Apply** and **Dismiss**; Cloud providers
→ **Cheaper machines** holds the thresholds (a dollar and fifteen percent of
the machine line by default), whether Arm types count, and *Apply
automatically in the quiet hours* (off by default).

## 6. What it costs

The address line is the same in every option within a region, and the machine
line is where Spot's discount is real. As an example at AWS list prices when
this guide was written, the smallest AWS type, t4g.nano as a Spot machine in
us-west-2, came to about $5.40 a month all-in: roughly a dollar for the
machine, $3.65 for the public IPv4 address ($0.005 an hour, which AWS charges
for every public IPv4 address), and $0.80 for the 10 GB disk. The Cloud pane
shows the prices for every type and region as you choose, and the Servers card
shows the month as it stands afterwards.

A kept address bills its line while it is reserved, machine or not.

## 7. Firewall and SSH

The machine's firewall holds exactly one list (Cloud providers → **Firewall
ports**): TCP 25, 465, 587, 143, 993, 995, 80, 443 and the relay's control
port, to anywhere, plus any rows you add. The console pushes the list before
every machine and restores it hourly if it is edited at the cloud. SSH is
**just in time**: before an operation the console opens port 22 from its own
address alone and closes it after. *Always open* is an option for operators
who log in themselves. The private half of the SSH key pair can be downloaded
once from the Cloud providers page; the download is logged.

## 8. Remove everything

Infrastructure → Mail relay → **Forget**. The machine is deleted and the relay
unpaired; the address stays reserved for the next machine unless you tick
*release the address too*. A kept address shows as its own card with
**Release**. While a machine stands, the account that holds it cannot be
deleted and the SSH key pair cannot be replaced.

## Running on a cloud machine you made yourself

Nothing here requires the console to make the machine: the relay installer
from the [README](README.md) works on any Linux VDS, cloud or not. What the
console adds is the reserved address that outlives the machine, the recovery
ladder, and the price comparison.
