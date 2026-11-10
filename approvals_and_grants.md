# Approvals and Grants

**Applies to:** VaultKit 0.12.0-alpha

Nobody in VaultKit holds standing access to data. Every request is checked against
your policies, and if it's allowed, VaultKit issues a **grant**: permission to run that
one query, for a limited time. When a policy says a person must sign off first, the
request waits for an **approval**.

## What happens to a request

1. **Checked.** The control plane checks that every dataset and field the query
   names is in the registry, and that the requester may see the dataset. It then
   applies the row limit, confines tenant-scoped datasets to the requester's tenant,
   and evaluates your policies for every field and table the query touches. An
   unknown field is refused with `422` and recorded as **Rejected**. An unknown or
   hidden dataset is answered `404` "Dataset not found", and recorded as Rejected
   or Denied.
2. **Decided.** The strongest matching action wins: deny, then require approval, then
   mask, then allow.
   - **Deny:** the request is refused, with the policy's reason.
   - **Allow or mask:** a grant is issued straight away. Masked fields are listed.
   - **Require approval:** the request is queued for the approver groups the policies
     name.
3. **Run.** The requester fetches the rows with the grant. Asking the same question
   again while the grant is valid reuses it. If an admin revoked the grant, asking
   again is evaluated afresh: a new grant, or a new approval.

Every step is written to the audit log.

## Approvals

An approval policy names a group in `approver_role`, or several in `approver_roles`
with an `approval_mode`:

```yaml
action:
  require_approval: true
  approver_role: payments_reviewer
  reason: Payment records need sign-off from a payments reviewer.
  ttl: 1h
```

```yaml
action:
  require_approval: true
  approver_roles: [security, finance_manager]
  approval_mode: any     # all (the default): every group signs off; any: one does
  reason: Agents reading financial rows need security or finance review.
```

**Who can approve:**

- **Group members:** people in the named approval group. Admins add them in the
  console under **Team** (Edit member, Approval groups).
- **Admins and approve-any users:** members who are set to "Can approve any request"
  can sign off for any group.
- **Never the requester:** nobody can approve or deny their own request, admins
  included.

**Several policies, several groups:**

- **Every matched policy's requirement must be met.** When several policies ask for
  approval, each keeps its own groups and mode, and the request needs them all.
- **One group per person.** Each sign-off comes from a different person and counts
  for one group only, even if that person is in several. An admin can't satisfy
  every group alone.
- **Which group a sign-off counts for:** a person in several groups signs for the
  one that meets the most open requirements. For example, with `financial_compliance`
  asking a `finance_manager` and `ai_safety` asking `security` or a `finance_manager`,
  one finance sign-off approves the request.
- **TTL:** the shortest among the matched policies applies.

**Progress** reads the same in the console's Approvals page, the approvals API
(`progress`), `GET /requests/:id` (`approval_progress`) and the agent's
`vaultkit_check_approval`: "Needs 2 approvals: finance_manager ✓ Ada, security
waiting". A set of groups any one of which will do is shown once ("security or
data_owner waiting").

**Deciding:**

- **Approve:** in the console under **Approvals**, or with `vkit approval:approve ID`.
  An approver can shorten the grant (`--ttl`), never lengthen it past the policy's
  TTL.
- **Deny:** any approver who could sign off can deny the whole request, with a reason.
- **After approval:** the requester runs the same request again and gets the rows.

**While it waits:**

- **Asking again** joins the pending approval rather than queueing another. An
  agent's free-text purpose doesn't count, so rewording it changes nothing.
- **A new policy bundle** deployed while it waits blocks the approval; the requester
  asks again under the new policies.

## Grants

A grant is bound to:

- **The exact query,** including its filters, and the tenant filter for tenant-scoped
  datasets. A different question needs its own grant, and its own approval if a
  policy requires one.
- **The requester:** only they can use it.
- **The active policy bundle:** deploying a new bundle ends existing grants.

**When its query fails to run:** if a run fails, for example because a column was
renamed in the database since the last scan, Activity shows the request as failed
("Approved · failed: column doesn't exist") and counts it under **Failed**. Admins see
the failed run's SQL. When the failure is about the schema (a missing column or
table), the grant can't run again as it stands. It's marked "Fails: column doesn't
exist" on the Agents page and in the approvals history; revoke it there. It isn't
revoked automatically, because such failures are often temporary.

**How long it lasts:** the matching policy's `ttl`, or one hour when the policy sets
none. The `starter` pack gives allowed requests four hours.

**What stays with it:** masking. A field masked by policy stays masked after
approval: an approval lifts the approval requirement, not the masks.

**Checked every time it's used.** A grant stops working when:

- it expires;
- an admin revokes it with `vkit grant:revoke --grant REF --reason TEXT` or from the
  console's Agents or Activity page, revokes the agent token it was issued to, or removes
  the requester from the team, which revokes all their grants;
- the active bundle changes, or the bundle it was issued under is revoked;
- a condition it was issued under changes. Conditions under a policy's
  `match.context` (such as the requester's role or region) and the tenant are checked
  again on each use.

## In the audit log

Requests record the full query, the requester and their role, clearance, region and
tenant, the environment and the decision. Events include `request.received`,
`policy.evaluated`, `request.rejected`, `request.denied`, `request.queued_for_approval`,
`approval.signed_off`, `approval.approved`, `approval.denied`, `grant.issued`,
`grant.reused`, `grant.fetch.executed` and `grant.revoked`.
