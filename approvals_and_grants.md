# Approvals and Grants

**Applies to:** VaultKit 0.9.0-alpha

Nobody in VaultKit holds standing access to data. Every request is checked against
your policies, and if it's allowed, VaultKit issues a **grant**: permission to run that
one query, for a limited time. When a policy says a person must sign off first, the
request waits for an **approval**.

## What happens to a request

1. **Checked.** The control plane applies the row limit, confines tenant-scoped
   datasets to the requester's tenant, and evaluates your policies for every field
   and table the query touches.
2. **Decided.** The strongest matching action wins: deny, then require approval, then
   mask, then allow.
   - **Deny:** the request is refused, with the policy's reason.
   - **Allow or mask:** a grant is issued straight away. Masked fields are listed.
   - **Require approval:** the request is queued for the approver groups the policies
     name.
3. **Run.** The requester fetches the rows with the grant. Asking the same question
   again while the grant is valid reuses it.

Every step is written to the audit log.

## Approvals

An approval policy names a group in `approver_role`:

```yaml
action:
  require_approval: true
  approver_role: payments_reviewer
  reason: Payment records need sign-off from a payments reviewer.
  ttl: 1h
```

**Who can approve:**

- **Group members:** people in the named approval group. Admins add them in the
  console under **Team** (Edit member, Approval groups).
- **Admins and approve-any users:** members who are set to "Can approve any request"
  can sign off for any group.
- **Never the requester:** nobody can approve or deny their own request, admins
  included.

**Several groups:** when the matching policies name more than one group, every group
must sign off, each by a different person. An admin can't satisfy all of them alone.
The shortest TTL among those policies applies.

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

**How long it lasts:** the matching policy's `ttl`, or one hour when the policy sets
none. The `starter` pack gives allowed requests four hours.

**What stays with it:** masking. A field masked by policy stays masked after
approval: an approval lifts the approval requirement, not the masks.

**Checked every time it's used.** A grant stops working when:

- it expires;
- an admin revokes it with `vkit grant:revoke --grant REF`, or removes the requester
  from the team, which revokes all their grants;
- the active bundle changes, or the bundle it was issued under is revoked;
- a condition it was issued under changes. Conditions under a policy's
  `match.context` (such as the requester's role or region) and the tenant are checked
  again on each use.

## In the audit log

Requests record the full query, the requester and their role, clearance, region and
tenant, the environment and the decision. Events include `request.received`,
`policy.evaluated`, `request.denied`, `request.queued_for_approval`,
`approval.signed_off`, `approval.approved`, `approval.denied`, `grant.issued`,
`grant.reused`, `grant.fetch.executed` and `grant.revoked`.
