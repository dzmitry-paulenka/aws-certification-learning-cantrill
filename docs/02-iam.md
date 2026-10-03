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