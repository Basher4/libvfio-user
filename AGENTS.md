# Working on TCP transport and message IRQs

For this feature, read these files before changing anything:

- `docs/vfio-user.rst`: normative protocol specification and proposed amendments.
- `docs/tcp-plan.md`: implementation scope, stages, and acceptance criteria.
- `docs/tcp-protocol.md`: non-normative rationale and discussion only.
- `docs/tcp-progress.md`: current status, approvals, evidence, and handoff.

Draft specifications and plans are not implemented functionality. Verify source
and working-tree state before relying on a previous handoff.

## Mandatory specification-first gates

At the start of EVERY stage, record whether a specification change is needed
and why. If yes, edit `docs/vfio-user.rst` first, explain the concrete diff, and
STOP for the user's explicit confirmation. The user may request revisions.
Only after the user approves that exact specification revision may code work
for that stage start. An agent reviewer cannot substitute for user approval.
If the design changes after approval, revise the spec and obtain confirmation
again before implementing the changed behavior. No elapsed-time assumption or
"ready for review" status counts as confirmation.

A no-spec-change assessment is also reviewed with the stage; it must not hide
new wire behavior, semantics, or compatibility policy in implementation notes.
The protocol specification is the source of truth; do not maintain a second
normative wire definition in Markdown.

Implement only the user-authorized stage. Stop at each code review checkpoint;
ready for review is not accepted. Only the user or an explicitly designated
reviewer accepts a code checkpoint. Delegation, when authorized, remains inside
the current stage and cannot bypass a specification or code-review gate.

Record actual test commands/results, failures/skips, reviewed commit ranges or
changed-file scope, and the next action in the progress file. Do not report
planned tests as run. Report architectural deviations before implementation.

Preserve unrelated edits and the user's TCP skeleton. Do not stage/commit
`.idea/` or other unrelated files. Do not edit the adjacent QEMU checkout unless
separately instructed. Keep migration (v2) out of the v1 implementation stages.
