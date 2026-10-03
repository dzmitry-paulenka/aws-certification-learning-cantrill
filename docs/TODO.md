# TODO

Background/reasoning for all of the below: [01-aws-accounts.md](01-aws-accounts.md)

## Decided: skip Cantrill's standalone PRODUCTION account

`[DOITYOURSELF] Creating the Production Account` asks for a second **standalone** account,
to be *invited* into the Organization later. Skipping it. Continuing single-account
(GENERAL) until the **AWS Organizations** section, then creating member accounts from
inside the org.

Nothing of substance is lost — DEVELOPMENT gets created in-org later anyway, so still
covered: creating an Organization, member accounts, cross-account role switching, SCP demos.
The only unique content is the invite handshake, which is mundane. The one fact to retain:

> Accounts **created inside** an Organization get `OrganizationAccountAccessRole`
> automatically. Accounts **invited into** one do not — you build the role yourself.

## Open items

- [ ] !!! Secure ROOT account properly and then **almost never** log into it (see 01-aws-accounts.md for details)
- read [Root user best practices for your AWS account - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)
- read [AWS account root user - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html#root-user-tasks)
- [ ] Verify account plan: `Billing → Account plan` (free plan vs paid) — determines whether
      additional accounts are even possible
- [ ] Verify free-tier state: `Billing → Free tier` + creation date in `Account settings`.
      If the allowance expired unused on this legacy account, consider opening a fresh
      account on a dedicated email and **closing this one** — clean modern account, no
      retail-identity entanglement, free-plan hard-stop billing
- [ ] Confirm `Billing → Bills` reads $0.00 (⇒ nothing running in any region)
- [ ] Register a 2nd (and 3rd) root MFA device — up to 8 allowed. Makes the phone-based
      lost-MFA recovery path irrelevant, which resolves the SMS/roaming worry
- [ ] Delete any root access keys
- [ ] Budget alarm at $5
- [ ] Record contact details exactly as entered + the 12-digit account ID, stored outside
      1Password
- [ ] Check Aegis encrypted export isn't backing up to the same cloud account as the
      root-password archive
- [ ] Paper backup of the archive passphrase (memorized-only + used twice a year = it will rot)

## At the Organizations section

- [ ] Create the Organization in GENERAL — **all features**, not just consolidated billing
- [ ] Create member accounts: email + account name only, plus-addressed. No root password,
      no MFA setup needed (org-created accounts start with no root credentials)
- [ ] Enable **centralized root access management** so member root credentials stay deleted
- [ ] Set up IAM Identity Center + SuperAdmin permission set; stop using root
- [ ] Attach SCPs: region lockdown (with the `NotAction` global-services carve-out) and
      expensive-service deny
