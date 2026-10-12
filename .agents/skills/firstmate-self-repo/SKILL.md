---
name: firstmate-self-repo
description: The firstmate-repository publication boundary. Load before dispatching, promoting, relaunching, or delivering any task whose project is the firstmate repository itself, when a firstmate bug or improvement idea surfaces in ordinary use, before granting or revoking upstream publication authority, and before deciding whether any push, pull request, issue, comment, or fork may reach firstmate's own upstream.
user-invocable: false
metadata:
  internal: true
---

# firstmate-self-repo

Every firstmate home clones the same public repository and runs from it, which makes firstmate's own upstream the one repository a running home can always reach.
Most homes belong to people who simply use firstmate, not to the people who maintain it, and a running agent that treats "I can see the repo" as "I may publish to it" is how a user's firstmate opens surprise pull requests upstream.
This skill owns the boundary between the two relationships.

## The default: user/operator

Absent any explicit grant, a home runs as firstmate **user/operator**.
It may install, run, update, reconfigure, and locally repair firstmate, and its ordinary project work flows through the registered delivery modes unchanged.
What it may never do on its own is publish to firstmate's upstream: no pushed branch, no pull request, no issue, no comment, no fork, no review.

Role is **never inferred**.
GitHub identity, forge permissions, the name or URL of `origin` or any other remote, repository ownership, authentication state, environment variables, and CI context all say nothing about role; the only inputs are the explicit local grants below.

## When a firstmate bug surfaces in ordinary use

A user/operator home that hits a firstmate bug does local work only:

1. Diagnose privately - reproduce, isolate, and root-cause inside the home and its task records.
2. Apply a local workaround when one exists - a config change, a pinned local edit, a different invocation.
3. Optionally prepare a sanitized local issue draft or a proposed local patch under the task's data directory, for the operator to review and submit by hand.
4. Report to the captain what happened and what was produced.

A local patch or workaround may touch the gitignored operational surface and a task's own worktree; nothing reaches upstream without the authority below.
This posture covers firstmate self-repair exactly the same way it covers a dispatched task: the boundary is about publication, not about which surface holds the fix.

## The two explicit grants

### Maintainer opt-in (standing)

The operator writes `config/firstmate-role` by hand - it is gitignored and never travels through git.
The file holds one directive line: `user`, or `maintainer <host>/<owner>/<repo>` naming the one upstream repository this home develops firstmate for.
A fork maintainer contributing to the parent repository uses one-task consent, not a second standing scope.
`docs/configuration.md` "Firstmate role" owns the operator-facing schema; `bin/fm-role-lib.sh` owns parsing, and `bin/fm-role.sh report` prints the resolved role (also surfaced in the session-start digest's HOME ROLE subsection).
A malformed, unreadable, or unverifiable file resolves to user, never to a guess.

### One-task contribution consent (bounded)

When the captain explicitly approves one specific upstream contribution, record it on that task only: `--upstream-consent <host>/<owner>/<repo>` on `bin/fm-brief.sh`, `bin/fm-spawn.sh`, and `bin/fm-promote.sh`.
Consent renders as an `upstream-consent=` token on the brief's `Delivery contract:` line, is recorded in `state/<id>.meta`, and a relaunch reuses the recorded value.
It changes the home's persistent role in no way; the next task is gated fresh.
The captain's approval must be a current explicit instruction naming the contribution - never an inference from the task resembling maintainer work.

## Where the boundary is enforced

The gate is code, not memory: `bin/fm-role-lib.sh`'s `fm_role_gate_self_publish` decides whether a task on firstmate's own repository may carry a publishing contract, by repository lineage (a shared git common dir, or a project holding a root commit of the code root's history, which every fork, the parent, and any clone of either share) rather than a freeform repo name or remote URL.
Lineage is computed offline; when it cannot be determined (for example a shallow clone sharing no root commit) the gate refuses.

- `bin/fm-brief.sh --project <dir>` refuses to scaffold a publishing ship brief the home cannot authorize.
- `bin/fm-spawn.sh` refuses to launch a ship task on this repository without a maintainer scope or matching task consent, and refuses a brief whose `upstream-consent=` disagrees with the flag.
- `bin/fm-promote.sh` applies the same gate to scout promotion so the in-place path grants no more than a fresh spawn.
- `local-only` publishes nothing, so it never needs authority and never takes consent.

An authorized contribution then flows through the task's normal delivery contract - usually no-mistakes - like any other project; the grant answers only "may this publish", never "how".

A gate refusal names the exact remedy: write `maintainer <scope>` into `config/firstmate-role`, or pass `--upstream-consent <scope>` for that one task with the captain's explicit approval.
Never work around a refusal by pointing the task at a second checkout, fork, or parent clone; `fm_role_self_repo` treats every repository sharing firstmate's lineage as firstmate's own.
A publishing contract also needs a determinable publication target (an upstream, github, or origin remote) matching the maintainer scope or the task consent; with no target it is refused even under consent.

## Updating is not contributing

`/updatefirstmate` and every guarded sync pull **from** upstream, which is consumption, not publication, and stays allowed for every role.
The boundary covers only traffic that writes to the upstream repository.

## Secondmates

A secondmate home inherits the primary's resolved role through the same primary-authoritative propagation as other inheritable config (`bin/fm-config-inherit-lib.sh`), local and remote routes alike; a primary file its own resolver rejects (malformed, symlinked, unverified) propagates as revocation, never as a grant.
The grant is honored only while `config/.fm-inherit-firstmate-role.generation` proves the exact bytes the primary last published, so a child never gains through supported propagation a grant the primary does not resolve, and a stale proof authorizes nothing: a removed or divergent receipt, a role file with no receipt, and a failed propagation all resolve to user.
The receipt proves consistency, not provenance - it prevents accidental publication through supported paths and is not a security boundary against a process that deliberately forges local records.
Revocation converges the same way - the primary deleting its role file propagates an `absent` receipt downstream.
