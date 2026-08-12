# AWS Accounts

## Fundamentals

- AWS Account is a **container** for identities and resources.
- Every AWS Account has a **ACCOUNT ROOT USER**. Has full access to all resources, can't be restricted.
  - **Exception — SCPs in AWS Organizations.** In a *standalone* account root is truly
    unrestrictable. In a **member** account of an Organization, **SCPs do apply to the
    root user** — deny `s3:*` via SCP and even root can't touch S3, with no workaround
    from inside the account.
  - Asymmetry: SCPs **never** restrict the **management** account (incl. its root).
    Hence: keep the management account empty, do real work in member accounts.
  - Joining an org keeps root's password/MFA, but root loses supremacy: SCPs bind it,
    billing moves to the payer, and with **centralized root access management** the
    management account can delete member root credentials entirely.
- Any other IAM identity (i.e. user, groups, roles) starts with **no permissions**. You have to explicitly grant permissions to them.

---

## Initial setup — what to put in place right away

The shape to aim for:

```
Management account          ← empty. no resources, ever. SCPs don't reach it.
├── root user               ← MFA'd, then never used
├── AWS Organizations       ← all features
├── IAM Identity Center     ← SuperAdmin  ← the actual daily driver
├── SCPs                    ← region lockdown + deny expensive services
└── Member accounts         ← where everything actually gets built
    ├── proj-networking
    ├── proj-serverless
    └── proj-scratch        ← nuke and recreate freely
```

Order of operations:

1. **Secure the management account root** — long unique password, **2+ MFA devices**,
   delete any root access keys. Then stop using it (see below).
2. **Budget alarm at ~$5.** Before building anything. Highest-value five minutes.
3. **Create the Organization** — **all features**, not just consolidated billing (SCPs
   require it).
4. **Enable IAM Identity Center**, create a user for yourself, and assign it the
   `AdministratorAccess` permission set. **This is what you log in as from now on.** Pick the
   home region carefully — it's annoying to change later.
5. **Attach SCPs** at the org root: region lockdown, deny expensive services.
6. **Create member accounts per project.** Email + account name only. Unique email required
   (plus-addressing is fine: `you+proj-networking@gmail.com`); phone and card are reused.
7. **Enable centralized root access management** so member accounts never get root
   credentials at all.
8. **Never mint long-lived access keys.** This is the most common breach vector; Identity
   Center's short-lived credentials remove it entirely.

Why the management account stays empty: SCPs don't restrict it, so it's the one place your
guardrails don't apply. Nothing you'd want protected should live there.

### IAM is per-account

There is no single global IAM. **Every AWS account has its own independent IAM** with its own
users, groups, roles and policies. An IAM user in the management account has *zero* access to
member accounts.

```
Organization
│
├── Identity Center identity store        ← "dzmitry" lives HERE
│     org-level — not inside any account's IAM
│
├── Management account
│     ├── root user
│     └── IAM ────────── users, groups, roles   ← "iamadmin" lives HERE
│
└── Member account (proj-scratch)
      ├── root user (or none at all)
      └── IAM ────────── users, groups, roles
            └── AWSReservedSSO_AdministratorAccess_abc123
                  ↑ provisioned automatically by Identity Center
```

|  | IAM user | Identity Center user |
|---|---|---|
| Lives in | one specific account's IAM | Identity Center's identity store (org-level) |
| Scope | that account only | assigned across many accounts |
| Sign-in | console with **account ID + username** | **access portal** URL |
| Credentials | permanent password / access keys | short-lived, per session |
| ARN | `arn:aws:iam::1234…:user/iamadmin` | not an IAM principal at all |

Two separate systems — creating `iamadmin` in IAM does not create an Identity Center user, and
vice versa. Same email in both would be coincidence, no link.

**But Identity Center doesn't bypass IAM.** An assignment provisions a real IAM **role** into
the target account, and the portal session assumes it. IAM roles underneath, centrally
managed, no IAM *users* involved.

> Cantrill still has you create `iamadmin` as an IAM user — correctly. The course teaches IAM
> fundamentals first, and the exam tests users/groups/roles/policies heavily. Identity Center
> is the better *operational* pattern for multiple accounts; it doesn't replace learning IAM.
> Expect to end up with both: `iamadmin` in GENERAL for the IAM lessons, an Identity Center
> user for actually moving between accounts.

### IAM Identity Center is a service, not an account

You don't log in "as Identity Center" or "to an account" — you log in as a **user in its
identity store**, then pick which account to enter. Three distinct things:

| Thing | What it is |
|---|---|
| **User** | your identity — lives in Identity Center, not in any account |
| **Permission set** | a policy template, e.g. `AdministratorAccess` |
| **Assignment** | user × permission set × target account |

An assignment makes Identity Center **provision a real IAM role into the target account**,
visible there as `AWSReservedSSO_<PermissionSetName>_<hash>`. So underneath it's still plain
IAM role assumption — Identity Center just manages those roles across every account.

- Sign-in is at a **dedicated AWS access portal** URL (`https://d-xxxxxxxxxx.awsapps.com/start`),
  not the normal console page. No 12-digit account ID needed.
- The portal shows a grid of accounts × roles; click one for a console session with
  short-lived credentials, or copy CLI credentials for the same.
- **One login, then pick the account** — exactly what an IAM user can't do, since an IAM user
  lives permanently inside one account.
- MFA for the Identity Center user is configured **separately** from root/IAM MFA.

**One instance per Organization.** Enabled from the management account, pinned to a **home
region** chosen at enable time (awkward to change later). Administration can be *delegated* to
a member account, though a few operations stay management-account-only. A standalone account
not in an org can enable it for itself; once it joins an org, the org instance governs.

(There's also a narrower **account instance** — single-account scope, can only back AWS managed
applications, *not* multi-account access. Not relevant here, just don't be confused by the term.)

---

## Root user

- **Must have MFA.** AWS has been phasing root MFA into *mandatory* since 2024.
- **Register 2+ MFA devices — up to 8 allowed.** Removes the lockout risk, and makes the
  phone-based lost-MFA recovery path irrelevant.
- **Delete root access keys.** There is no good reason for them to exist.
- **Then stop using root.** Daily work goes through Identity Center. Root is for billing,
  account closure, and a short list of account-level tasks — a few times a year.
- **It's fine for root to be inconvenient.** Inconvenience is a feature for an identity you
  touch twice a year. AWS agrees: root sign-ins from a new device also trigger an
  **additional verification** email code (separate from MFA, not disableable), and the
  account's **phone number is mandatory** — it's not an MFA device, it exists for lost-MFA
  recovery and support identity verification.
- Root recovery ultimately runs through **email + phone**, so hardening the email account
  matters at least as much as the AWS password. Store the **12-digit account ID** somewhere
  you can reach even if locked out of your password manager.

> Note: AWS and amazon.com **share the email namespace but not the password store** — same
> email, separate dedicated passwords. Resetting one doesn't touch the other.

---

## Free tier

- Tracked **per account**, by creation date and usage. Consolidated across an Organization —
  a fresh member account gives a clean slate, **not** a fresh free tier.
- Two plans since mid-2025: **free plan** (credits, and it *hard-stops* instead of billing
  you — best spend protection, but can't create additional accounts) and **paid plan**
  (pay-as-you-go; legacy accounts are on this). Check `Billing → Account plan`.
- **You can't farm it with `+alias` emails and virtual cards.** It's prohibited by the AWS
  Customer Agreement, and it doesn't work: `name+x@gmail.com → name@gmail.com` is trivially
  normalized, and virtual cards resolve to the same name/address/BIN pool. Accounts get
  flagged weeks later and suspended, and the flag follows your real identity.

`+alias` emails do work fine *as addresses* — they just don't disguise you as a different
customer. Wrong tool for evasion, right tool for org member accounts, where being linked to
one owner is the point.

---

## Organizations hierarchy — four distinct types

The tree has no official AWS name — AWS names the *entity* ("organization") and just calls the
structure "the organization's hierarchy." Informally, **organization tree**.

It groups **accounts**, for policy and consolidated billing. It is **not** an identity hierarchy —
identities live inside each account and aren't part of the tree.

The ID prefixes give the game away. These are four different kinds of thing, not variations
on one:

| Thing                 | ID looks like | Is it an account? | Has a root user? |
|-----------------------|---|---|---|
| **Organization**      | `o-a1b2c3d4e5` | no | no |
| **Root**              | `r-a1b2` | no | no |
| **Organization Unit** | `ou-a1b2-x1y2z3` | no | no |
| **Account**           | `111122223333` | yes | **yes, exactly one** |

**Only accounts have root users.** Everything else is a folder or a label.
**Root is conceptually a top-level OU.** and OUs basically just an account hierarchy folders.

```
Organization  o-a1b2c3d4e5           ← the entity itself. NOT a node in the tree.
│   • has exactly one Root
│   • has exactly one management account
│
└── Root  r-a1b2                      ← the mandatory top-level OU. A folder.
    │
    ├── acct  111122223333  Management     ← just an account that sits here
    │
    ├── OU  ou-a1b2-sandbox           ← a folder.
    │   └── acct  222233334444  proj-scratch
    │
    └── OU  ou-a1b2-workloads         ← a folder.
        ├── acct  333344445555  proj-networking
        └── OU  ou-a1b2-inner         ← OUs nest, up to 5 deep
            └── acct  444455556666  proj-serverless
```

### Where the root user fits: one level below the tree

The root user is **not part of the Organizations hierarchy**. The tree is about grouping and
policy; identities are a separate layer, inside each account leaf:

```
Root  r-a1b2                              ← org hierarchy ends at accounts
└── acct  111122223333
      │
      │   ┄┄ zoom in: identities live in here ┄┄
      │
      ├── root user                       arn:aws:iam::111122223333:root
      │     • the identity created *with* the account
      │     • its "username" is the signup email address
      │     • NOT an IAM user — never appears in the IAM users list
      │     • has no IAM policy attached, and can't be given one
      │     • unrestricted inside this account (unless SCPs bind it)
      │     • can't be deleted while the account exists
      │
      └── IAM
            ├── users    (iamadmin, …)
            ├── groups
            └── roles    (incl. AWSReservedSSO_… from Identity Center)
```

The root user is the account's **original owner identity** — it exists because the account
exists, is addressed by the signup email, and sits **outside IAM's user list**, which is why you
can't attach a policy to it or delete it.

Pure word reuse, three unrelated things:

| Phrase | What it actually is |
|---|---|
| **Root** (`r-a1b2`) | the top-level OU — a folder in the org tree |
| **root user** | an identity inside a single account |
| ~~"root account"~~ | **not an AWS term.** People saying it usually mean the *management account* |

That last row causes most of the confusion — "root account" gets thrown around constantly and
means nothing precise.

One more overload: `arn:aws:iam::111122223333:root` in a **resource** policy (e.g. an S3 bucket
policy) means **"any principal in account 111122223333"** — the whole account, not literally the
root user. Same string, two meanings depending on context.

### The relations

- **Organization ↔ OU** — the Organization *has* a tree of OUs. The Organization is the overall
  entity (grouping + consolidated billing + the policy engine); OUs are nodes inside its tree.
  The Organization is not itself a node.
- **Organization ↔ Account** — every account is a *member of* the organization and belongs to
  **exactly one**. One is the **management account** (the one that created the org); the rest are
  member accounts. Accounts are the leaves.
- **OU ↔ Account** — an OU *contains* accounts and other OUs. An account sits in exactly one OU
  (or directly in Root) at a time; moving it changes which SCPs apply immediately. Grouping only
  — so policies can hit a set of accounts at once.
  - An OU may be **completely empty**, or contain only other OUs. All valid.
  - So the normal workflow is **pre-staging**: create the OU, attach its SCPs while empty, then
    move accounts in — the guardrail applies the moment an account arrives.
  - **Deleting an OU requires it to be empty** (no accounts, no child OUs).

### What "unrestricted" actually means

Two **independent** constraints. Root users escape one of them always; only one cell escapes both:

| | bound by IAM policy? | bound by SCPs? | net |
|---|---|---|---|
| **Management account root user** | no | no | **unrestricted, absolutely** |
| Member account root user | no | **yes** | bounded by SCPs |
| IAM user/role in mgmt account | yes | no | bounded by IAM only |
| IAM user/role in member account | yes | yes | IAM ∩ SCP |

- **SCPs don't apply to the management account at all** — not just its root user, but *every*
  principal in it. Its IAM users are still bounded by their IAM policies though; root has no IAM
  policy to bound it. That combination is what makes it absolute.
- **Member root users aren't special for escaping IAM** — every root user does. What makes them
  restrictable is the SCP ceiling arriving from above.

### Traps

- **An OU has no account and no root user.** Nothing owns an OU; it's a pure folder.
- **Organization ≠ Root.** The Organization *contains* the Root.
- **Nothing is "Root's account."** The management account is just an account sitting in the tree
  (usually directly in Root), distinguished by having created the Organization — a property of the
  *Organization*, not of the Root node.
- **Root is just the top-level OU** — same kind of node, only it always exists, has no parent, and
  can't be created, deleted, moved, or given siblings. Like `/` in a filesystem: `/` *is* a
  directory, just the special one at the top. The API confirms it —
  `ListOrganizationalUnitsForParent(parent=…)` accepts an `r-` or an `ou-` ID interchangeably.
- **Root users are not singular** — every account has one. What's singular is the *unrestrictable*
  one: only the **management account's** root user escapes SCPs.
- An SCP on the **Root** applies to every account in the org except the management account. SCPs
  inherit downward through OUs.

---

## Member accounts: created-in-org vs invited

| | Created inside the org | Invited into the org |
|---|---|---|
| `OrganizationAccountAccessRole` | created automatically | **not created** — build it by hand |
| Root credentials | **none set at all** | already exist, owner keeps them |
| Signup flow | none — email + name only | full signup: phone verification, card |

**What an invited account loses.** It keeps its root password and MFA, but root stops being
supreme: SCPs now bind it, billing control transfers to the payer, and centralized root
access management lets the management account delete its root credentials. It can leave only
if **standalone-capable** (own payment method, full signup info), and can be ejected
unilaterally. So accepting an invitation is a real transfer of governance — the M&A scenario.

**Management account access into a member.** Org membership grants **no implicit resource
access**. Two ways in:

- **IAM Identity Center** — assign a permission set and it provisions the role into the
  member account itself, via Organizations trusted access. **No cooperation needed from the
  member.** This is the practical answer to "as the payer, can I administer it?"
- **Manual cross-account role** — member admin creates a role trusting your account ID;
  you `sts:AssumeRole`. One-time cooperation.

---

## SCPs (Service Control Policies)

An Organizations feature. Not `scp` the file-copy command.

- **Sets the ceiling; never grants.** Effective permissions = IAM allows ∩ SCP allows.
- Can't be escaped from inside the account — it isn't part of the permission system, it's a
  boundary around it. Even `AdministratorAccess`, even member root, is bound.
- Attach to org root, an OU, or a single account; inherits downward.
- **Requires all-features Organizations.**
- **Does not restrict the management account.**
- **SCPs prevent, they don't remediate.** `Deny ec2:*` blocks new API calls; running
  instances keep running *and keep billing*. No SCP deletes a resource.

### Region lockdown

Attackers fan out across all ~30 regions; you'd never notice instances in `ap-southeast-3`.

```json
{
  "Effect": "Deny",
  "NotAction": [
    "iam:*", "organizations:*", "sts:*", "cloudfront:*",
    "route53:*", "support:*", "budgets:*", "ce:*"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["eu-central-1", "us-east-1"]
    }
  }
}
```

The `NotAction` list is **essential** — global services are pinned to `us-east-1` and
omitting them locks you out of IAM.

### Deny expensive services

Flat deny on things never needed while learning: `sagemaker:*`, `emr:*`, `redshift:*`,
`bedrock:*`. This is where five-figure surprise bills come from.

### Emergency brake + cleanup

```
Brake (instant, org-wide, binds member root too):
  SCP { "Effect": "Deny", "Action": "*", "Resource": "*" } on the OU
  → nothing new can be created anywhere

Then detach it and clean up (needs admin via Identity Center or a role):
  assume admin → terminate instances per region
```

Ordering matters: the deny-all also blocks `ec2:TerminateInstances`, so brake to stop
growth, then release to terminate.

Gotchas when sweeping:
- **Delete Auto Scaling groups first** — an ASG relaunches terminated instances.
- **Termination protection** blocks `terminate-instances`; disable per instance first.

---

## Cost control and blast radius

### What does NOT protect you

- **A spend limit on the linked card.** AWS usage is a **debt, not a prepayment**. A
  declining card doesn't stop the meter — AWS retries, dunning, suspends the account, can
  send it to collections. Same as capping your card to limit a phone bill.
- **AWS Budgets.** They send email. That is all.
  - Exception: **Budget Actions** can apply a deny IAM policy / SCP or stop EC2/RDS at a
    threshold. Real, but fires on billing data that lags **hours to a day** — a compromised
    key running GPU instances in every region does five figures before the first alert.

### What does

- No long-lived access keys — Identity Center short-lived credentials instead.
- Region-lockdown SCP + expensive-service deny SCP.
- **Leave service quotas alone.** New accounts have low vCPU limits; that's protective.
- MFA on root, zero root access keys.
- Budget alarm at ~$5 from day one.
- GuardDuty (few $/month) detects mining and credential abuse.
- Being on the **free plan**, which genuinely hard-stops.

### If compromised

- **AWS blocking speed is unreliable** — hours to days. Don't plan around it.
- One fast exception: a key published publicly (GitHub, npm, pastebin) is usually caught by
  AWS's scanners in minutes and auto-quarantined. Useless for a key stolen off a laptop.
- **AWS routinely waives fraud charges for first-time victims** who open a support case
  promptly and rotate credentials. Not guaranteed, not policy — but the usual outcome for
  hobby accounts. **Open the case immediately**, don't wait for the invoice.

### Cost traps that hide in forgotten regions

| Resource | Cost |
|---|---|
| NAT Gateway | ~$32/mo each — the classic |
| Public IPv4 address | ~$3.60/mo each, charged for **all** of them since Feb 2024, even attached |
| Unattached EBS volumes, old snapshots | per GB |
| RDS instances, Redshift clusters | substantial |
| Route 53 hosted zone | $0.50/mo each |

Free: VPCs, subnets, route tables, security groups, internet gateways.

---

## Resetting / cleaning up an account

**There is no factory reset.** AWS has no "revert account to day one."

- **Member accounts are the unit of reset.** Close and recreate one — that *is* a factory
  reset. (Doesn't restore free tier.)
- **Write infra as code** (Terraform/CDK). `destroy` is the undo button and the state file is
  the record of "what did I tweak." Console clicking is what creates the drift.
- **Bulk erasers** for an already-messy account: `aws-nuke` (ekristen), `cloud-nuke`
  (Gruntwork). Genuinely destructive — throwaway accounts only.
- **CloudTrail** keeps 90 days of management events with no setup: `CloudTrail → Event
  history`. Every setting toggled is in there.

### Settings that don't unwind cleanly

Avoid casually enabling: **IAM Identity Center** (painful to unwind), **Organizations**
itself, **Cost Explorer** (can't be turned off, harmless), **opt-in regions**, and
**AWS Config / GuardDuty / Security Hub** — these three bill continuously and quietly.

### Finding stragglers

1. **`Billing → Bills`**, expand by service, last few months. **$0.00 ⇒ nothing is running
   anywhere, full stop.** Thirty seconds, and it's the definitive answer.
2. **Resource Explorer** (free, one-time enable) indexes across all regions.
3. CLI sweep:

```bash
for r in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo "== $r"
  aws ec2 describe-instances --region "$r" \
    --query 'Reservations[].Instances[].[InstanceId,State.Name,InstanceType]' --output text
done
```

Swap in `describe-nat-gateways` / `describe-addresses` for the expensive ones.
