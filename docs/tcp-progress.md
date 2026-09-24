# Feature progress and agent handoff

## Current state and next action

- Active stage: **S1 specification review — awaiting explicit user confirmation**.
- This task revises design/specification/handoff documents only. No feature code
  has been implemented and no implementation stage is authorized by this status.
- Next action: user reviews the message-IRQ amendment in
  [vfio-user.rst](vfio-user.rst), requests changes or explicitly approves it.
  Only then may an agent implement S1. Do not jump to TCP or twin sockets.
- Specification approval: **not received**. The primary-channel IRQ ordering
  choice and DATA_MESSAGE/BOOL combination are proposals included for review.
- Baseline HEAD: `803924493a7c787b2ba358751f55f07d5dba64b2`.
- Pre-existing user changes: `lib/meson.build`, staged/modified skeletons
  `lib/tran_tcp.c`, `lib/tran_tcp.h`, and untracked `.idea/`. Preserve them.
- User supplied `docs/vfio-user.rst`; it is now the normative working spec.
  Original downloaded file SHA-256:
  `cd05c1570caddb18bfa6a39406d92f7d0eb124cd99ca2ab4f7ff9ace0ac38ff0`.
  Original text was copied to `/tmp/libvfio-user-spec-before-message-irq.rst`
  for this review's diff; that temporary copy is not durable handoff storage.
  Preserve/review the specification baseline in version control when committing.

## Superseding decisions

- Message IRQ protocol extension is first, tested on SOCK before TCP.
- TCP must work with a single connection and no twin capability.
- Optional TCP twin support is a separate later patch; examples default to
  one socket and later gain an explicit twin option.
- `tran-tcp` remains opt-in, default false. IRQ support is independent of it.
- One-port creation creates one listener; optional two-port creation must acquire
  both listeners or fail. No fallback to another address/port after bind failure.
  Explicit zero is an application request for ephemeral allocation.
- Recommend 0.3 for message IRQs, no extra bump for single TCP, and 0.4 for the
  later new TCP twin wire negotiation. Version and capability gates both apply.
- DATA_MESSAGE configures delivery, optionally with DATA_BOOL subset selection;
  zeros leave vectors unchanged. GET_IRQ_INFO MESSAGE indicates support only.
- Proposed IRQ events always use primary, ordered with configure/disable/reset
  replies; optional secondary carries DMA. This replaces the previous secondary
  IRQ/barrier proposal and must be explicitly reviewed in S1.
- RST is normative; tcp-protocol.md is rationale/discussion, not a second spec.
- Every stage records whether the spec needs changes. If yes, amend it and wait
  for the USER to approve before any stage code changes. Reviewer acceptance is
  not a substitute for this confirmation.
- QEMU implementation and real guest tests remain outside this repository scope.
- Migration stays v2, with its own spec-impact/IRQ restore gate.

## Stages and approval ledger

| Stage | Spec assessment/status | Code status |
| --- | --- | --- |
| S1: message IRQs | Required; RST 0.3 draft awaiting user approval | Not started |
| S2: single-socket TCP/examples | Assess/clarify FD-free and duplex wording before code; no new wire version expected | Not started |
| S3: optional TCP twins | Required; proposed 0.4 negotiation/association draft not yet written | Not started |
| S4: integration/docs | Assessment required; amendments, if any, need user approval | Not started |
| V2: migration | Deferred; assess restore contract and any wire changes | Not started |

For each approval record the approving user message and exact reviewed revision
(commit/hash or preserved diff), not just "approved". Spec revision after approval
requires renewed confirmation. Code checkpoints independently record reviewer
findings, fixes, and acceptance. No prior checkpoint was accepted automatically.

## Implementation findings retained for later agents

- `lib/tran.c` computes but does not persist the negotiated minor.
- Existing IRQ code stores eventfds and forwards mask/unmask callbacks, but has
  no internal INTx automask state.
- Existing IRQ storage shares INTx/MSI/MSI-X vectors; index-aware message delivery
  must not infer type solely from vector number.
- Existing socket DMA reply handling warns about simultaneous commands without
  twin sockets; single-channel TCP needs real demultiplexing/progress, not a copy
  of that assumption. Queued work also needs poll-readiness signaling.
- Existing eventfd cleanup provides registration lifetime boundaries; it is not
  evidence that two streams are globally ordered.

## Validation and review evidence

- Previous document-only checkpoint checked local links/whitespace; no builds.
- Current task inspected the supplied RST, library negotiation/IRQ code and the
  adjacent QEMU VFIO PCI notifier teardown. QEMU source was read only.
- `rst2pseudoxml --halt=warning docs/vfio-user.rst /tmp/libvfio-user-message-irq-spec.xml`
  passed after correcting a new section's underline length.
- `rst2html5 --halt=warning docs/vfio-user.rst /tmp/libvfio-user-message-irq-spec.html`
  passed; generated an HTML review copy without parser warnings.
- Python checks of all four Markdown documents' local links, whitespace and
  final newlines passed. Checks that the RST contains the S1 IRQ amendment and
  no prematurely added TCP pairing command passed.
- `git diff --check` passed; new untracked documents checked separately above.
- Reviewed the amendment against the preserved downloaded baseline. Current
  RST SHA-256 (awaiting approval):
  `bdb00e6ce3afc3cbf248f0fbd30ade5c5236016dcf3990275b7b7322efd7a7d0`.
- No source changes or commits; no feature tests/builds claimed.

## Reusable prompts

Specification stage:

> Read AGENTS.md and all four feature documents. Assess stage S<N>'s spec impact.
> If it changes protocol behavior, update docs/vfio-user.rst, present the exact
> changes and stop for my confirmation. Do not implement code until I approve
> this specification revision. Record approval status and open questions.

Implementation stage (only after the required specification confirmation):

> Verify the recorded spec approval and current tree. Implement only stage S<N>
> and its acceptance tests. You may delegate bounded subtasks within this stage.
> Preserve unrelated edits. Stop for code review, recording actual commands,
> results and handoff. If you discover a necessary spec change, revise the spec
> and wait for my confirmation before implementing the affected behavior.

Code review:

> Review stage S<N> against the approved RST revision and plan. Independently
> inspect code/tests and report actionable findings. Record the outcome; do not
> start another stage or approve a protocol change on my behalf.
