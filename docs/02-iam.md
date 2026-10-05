# IAM

- On a high level, IAM has 3 main jobs
  - it's an identity provider (IdP), it let's you create, modify and delete identities, such as users, groups, and roles.
  - it also authenticates those identities
  - it then authorizes those identities to perform actions on AWS resources, based on the policies that are attached to those identities.
- Confers to a "least privilege" principle, granting only the necessary permissions to perform specific tasks.
- IAM, like an account Root user, **can do anything in the account**. There are some restrictions, that are generally around billing control and account closure. 
- IAM let's us create 3 types of things: users, groups, and roles.
- Users represent humans and applications, that needs access to your AWS account. 
- Groups are just collections of related users.
- Roles are identities that get **assumed**, *not* attached. You never attach a role to a user or group — you temporarily **become** the role. They carry permissions but **no credentials of their own**. Used to grant access to other accounts, external/federated identities, and AWS services.
- Another concept is **IAM Policies**. Which are essentially documents, which can be used to **Allow** or **Deny** access to AWS services and resources. They work **when and only when** they are attached to IAM users, groups or roles. They don't do anything on their own.
- IAM servicd is provided at no cost.
- IAM user can have **up to 2 access keys** — the second slot exists for **rotation**: create the new key, migrate everything to it, then delete the old one. No downtime window.
- Every role carries two policies - trust policy and permissions policy → see below.

---

## Roles and `sts:AssumeRole`

A role is an identity with permissions but **no credentials of its own** — no password, no
permanent access keys. It sits there until someone assumes it.

Every role carries **two** policies. This is the structural bit to memorize:

| Policy | Question it answers |
|---|---|
| **Trust policy** | *Who* may assume this role? (unique to roles) |
| **Permissions policy** | *What* can it do once assumed? |

`sts:AssumeRole` is the API call that does the becoming. **STS = Security Token Service.**

```
iamadmin (mgmt acct 1111)                     role in member acct 2222
      │                                        ┌─ trust policy: "any principal in 1111" ✓
      │  1. sts:AssumeRole(role_arn) ─────────► │
      │     caller's own IAM must allow it ✓   └─ permissions: AdministratorAccess
      │
      ◄── 2. temporary credentials ────────────
      │      AccessKeyId + SecretAccessKey + SessionToken
      │      expires in 1h (up to 12h, per role's MaxSessionDuration)
      │
      └─ 3. you are now:
           arn:aws:sts::222233334444:assumed-role/OrganizationAccountAccessRole/dzmitry
```

### Four things that trip people up

- **`SessionToken` is the tell.** Temporary credentials always have one; an IAM user's
  long-lived access keys never do. See a session token ⇒ you're in a role session.
- **Two gates, both must allow.** The trust policy must permit the caller, *and* the caller's
  own IAM policy must permit `sts:AssumeRole` on that role ARN. Most IAM needs one allow;
  cross-account role assumption needs both sides to agree.
- **Your permissions are replaced, not added.** After assuming, you no longer have `iamadmin`'s
  permissions. No union. Getting them back means ending the session.
- **The ARN is `sts::`, not `iam::`**, and it embeds the session name you passed — which is what
  appears in CloudTrail. That's why `--role-session-name` matters for audit.

### Where roles get used — all the same mechanism

- **cross-account access** — e.g. `OrganizationAccountAccessRole` from the management account
- **EC2 instance profiles / Lambda execution roles** — services get credentials with no
  embedded keys anywhere
- **federation / IAM Identity Center** — an external identity becomes a role session
- temporary privilege elevation

### The STS family

All return the same kind of temporary credentials:

| Call | Caller |
|---|---|
| `sts:AssumeRole` | an IAM principal |
| `sts:AssumeRoleWithSAML` | SAML federation |
| `sts:AssumeRoleWithWebIdentity` | OIDC — Cognito, GitHub Actions |
| `sts:AssumeRoot` | Organizations centralized root access (cannot be called *by* a root user) |

### Why AWS pushes roles over users everywhere

**Nothing long-lived to leak, automatic expiry, and every session is attributable in
CloudTrail.**

### Assuming a role in practice

Console — signed in as `iamadmin`:
> account menu (top-right) → **Switch role** → account = target 12-digit ID,
> role = `OrganizationAccountAccessRole`

CLI — a profile in `~/.aws/config` does the AssumeRole call for you:
```ini
[profile proj-scratch]
role_arn       = arn:aws:iam::222233334444:role/OrganizationAccountAccessRole
source_profile = iamadmin
region         = eu-central-1
```
```bash
# check who you are
aws sts get-caller-identity --profile proj-scratch
```
---

## Managed and inline policies

| | Managed policy | Inline policy |
|---|---|---|
| What it is | Its own IAM object, with its own ARN | Part of one user, group or role, no ARN |
| Reuse | Attach to many identities | Belongs to exactly one |
| Versions | Up to 5, roll back with `SetDefaultPolicyVersion` | None |
| In CloudFormation | `ManagedPolicyArns` on the role/user/group, or an `AWS::IAM::ManagedPolicy` resource | the `Policies` property |

Two kinds of managed policy. The ARN tells them apart:

- **AWS managed**: AWS writes and updates them. `arn:aws:iam::aws:policy/AdministratorAccess`
- **Customer managed**: you write them. `arn:aws:iam::111122223333:policy/MyPolicy`

Limit: 10 managed policies per user or role by default, raisable with a quota request.

### Edits apply everywhere, at once

- Editing a managed policy creates a new version and makes it the default. Every identity it's attached to gets it within seconds.
- That includes **active role sessions**. IAM checks permissions on every request.
- You can't pin a version per attachment.
- AWS edits its own managed policies too, mostly to add actions for new features.

Changing many identities at once is the reason managed policies exist: fix a mistake once, not in 200 inline copies. The risk sits with whoever can edit the policy.

### Breadth is the real risk

AWS editing `AmazonS3ReadOnlyAccess` adds little risk. You already trust AWS to run S3. The policy's breadth is the problem:

```json
// AmazonS3ReadOnlyAccess, roughly
{
  "Effect": "Allow",
  "Action": ["s3:Get*", "s3:List*", "s3-object-lambda:Get*", "s3-object-lambda:List*"],
  "Resource": "*"
}
```

- **`Resource: "*"`** covers every bucket in the account.
- **Wildcard actions grow on their own.** A new S3 `Get...` action falls under `s3:Get*` with no policy edit. An inline `s3:Get*` grows the same way.
- **"ReadOnly" includes data.** `s3:GetObject` reads every object in every bucket.

So the choice that matters is broad vs scoped, more than managed vs inline. A scoped policy lists explicit actions and ARNs, and nothing in it grows until you change it:

```json
{
  "Effect": "Allow",
  "Action": ["s3:ListBucket"],
  "Resource": "arn:aws:s3:::my-blog-images"
},
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::my-blog-images/*"
}
```

`ListBucket` acts on the bucket ARN, `GetObject` on the object ARN (`/*`). Mixing them up is a common reason for "access denied".

### Who can edit policies

Editing rights are **permissions on the policy**, not a trust relationship. A trust policy only controls who can assume a role. Actions to restrict:

- `iam:CreatePolicyVersion`, `iam:SetDefaultPolicyVersion`
- `iam:AttachRolePolicy`, `iam:AttachUserPolicy`, `iam:PutRolePolicy`

These are classic **privilege escalation** paths. A principal that can edit a policy attached to itself can grant itself anything.

Other controls:

- **Stage risky changes.** Attach a new policy to one role, test, then move the rest. Don't edit the shared policy in place.
- **Check before deploy.** IAM Access Analyzer's `CheckNoNewAccess` compares two policy versions and fails if the new one grants more.
- **Audit.** CloudTrail logs `CreatePolicyVersion` and `SetDefaultPolicyVersion`.
- **Cap it.** Permissions boundaries and SCPs limit what any policy can grant.
