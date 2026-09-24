# Protocol design notes: message IRQs and TCP

This file is **non-normative rationale and review discussion**.
[vfio-user.rst](vfio-user.rst) is the sole normative protocol document. The
current RST amendments are the S1 message-IRQ proposal and await user approval;
there is no implementation yet. [tcp-plan.md](tcp-plan.md) defines stages and
[tcp-progress.md](tcp-progress.md) records approvals and execution evidence.

## Versioning recommendation

Use upstream 0.2 as the baseline, message IRQs as 0.3, and the later optional TCP
channel-pairing extension as 0.4. Do not attach a version to a particular local
checkout: a commit series can introduce and then implement one new version.

Single-socket TCP itself does not require 0.4. The base spec already anticipates
AF_INET and commands/replies sharing one connection. New IRQ wire semantics
justify 0.3; new twin-channel discovery/association would justify 0.4. Explicit
capability agreement is required in addition to versions. Numeric allocations
in the RST amendment are proposals until reviewed/assigned; implementation
milestones alone do not reserve numbers or require a version bump.

## Why keep delivery configuration explicit?

Capability negotiation says both peers understand message IRQs; it does not say
that the client has established guest routing for any particular vector. An
unmasked vector can still be disabled/unregistered. Automatically sending events
for every unmasked vector would lose that distinction, make disable semantics
unclear, and complicate coexistence with eventfds on SOCK.

DATA_MESSAGE performs the same registration role that associating an eventfd
performs today. It is not a second mask. Delivery can then be enabled, masked,
unmasked, replaced, or disabled without conflating peer support and vector state.
No-FD DATA_EVENTFD must continue to deassign, never silently select messages.

The previous draft's exclusive-selector design could configure discontiguous
vectors only through multiple requests. The revised RST allows DATA_MESSAGE
alone for a full range or DATA_MESSAGE | DATA_BOOL for a subset, both with
ACTION_TRIGGER. Boolean 1 selects; 0 leaves the existing registration unchanged.
This is an explicit extension to the old mutually exclusive DATA rules, gated
by minor/capability. It does not combine message delivery with DATA_EVENTFD.

## IRQ information is support, not status

The user is correct: GET_IRQ_INFO attributes describe capabilities. The proposed
MESSAGE bit means that this IRQ index supports message delivery in the negotiated
session. It stays set whether zero, some, or all vectors are currently configured
for it. It does not query delivery mode or mask state. EVENTFD and MESSAGE can
both be advertised on an FD-capable transport.

There is no general class of devices for which message IRQs intrinsically cannot
work. INTx, MSI, MSI-X, ERR, and REQ can be represented if their device/client
supports the corresponding semantics. A client using eventfds on SOCK does not
need message delivery; a TCP session that never needs interrupts does not need
it either. A TCP session that needs interrupts requires a supported in-band
mechanism. This is why "optional for polling-only devices" was misleading:
it describes the use of interrupts, not a device-class restriction.

## Twin sockets are optional and come later

Recommend implementing single-socket TCP first and adding twin support in a
separate patch/stage. This proves the required fallback and full-duplex progress
before adding listener pairing, session tokens, deadlines, and multi-FD polling.
The cost is implementing demultiplexing now, but that work is required regardless
if TCP is to work when twin capability is absent. The tests must exercise this
path permanently; examples default to one socket and later offer twin mode.

One-port context creation requests one listener. Two-port syntax, introduced
only with optional twin support, explicitly requests both listeners; creation
fails if either cannot bind/listen. Local configuration precedes peer negotiation:
a second listener can be available even when a peer declines twin mode.

The earlier phrase "try the next address" meant another resolved IP, not another
port, but it could still hide a failed intended bind. The revised plan chooses
one resolved address and fails on bind/listen error. It never silently substitutes
a port or another resolved address. Explicit port 0 requests an ephemeral port;
only the application decides to request that behavior.

## Ordering: why SOCK/eventfd is not the same identity problem

Unix twin sockets do not create global ordering across streams either. Existing
IRQs usually travel through a separate eventfd object, not an IRQ-index message
on the twin stream. The descriptor registration identifies the delivery lifetime.
A properly synchronized disable unregisters the notifier and releases that
registration; a later new eventfd is a different kernel object. An old signal
does not automatically become an event on a newly created notifier merely
because the IRQ index or descriptor number is reused.

For example, the adjacent QEMU VFIO PCI code's INTx disable path disables the
IRQ, clears pending state/deasserts it, removes the FD handler, and cleans up
the notifier. This is evidence of lifetime management, not proof that every
vfio-user client has no race. Existing SOCK transport itself has no general
cross-stream drain barrier, and its twin negotiation solves command/reply
separation, not all device-state synchronization.

A message carrying only index/start/count lacks the eventfd object's registration
identity. Sending those events on secondary while configure/disable replies
travel on primary creates a new stale-association ambiguity.

**Proposed S1 solution for user review:** always send IRQ_TRIGGER on primary,
even with twin mode. Serialize events and successful configuration/reset replies
and process them in receive order. Old-registration events precede the control
reply; newly enabled events follow it. DMA alone moves to the twin channel.
This preserves the requested IRQ event layout and avoids new generation fields,
a new barrier command, or any per-interrupt acknowledgement. The tradeoff is
that a client must handle unsolicited IRQs on primary even in twin mode, and
IRQ delivery shares primary-channel bandwidth. This is explicitly specified
as an exception to generic twin routing, not assumed existing behavior.

If the user instead requires IRQs on secondary, revise S1 before coding to
specify delivery generations or a configuration-time drain barrier. Do not claim
that separate sockets themselves solve ordering. The old unresolved barrier
requirement has been replaced by this concrete primary-stream proposal, which
is not approved until the user confirms the amended specification.

## Specification review gates

Every stage begins with a recorded spec-impact assessment. Any protocol change
is first written in RST and presented for user confirmation. Only that user's
explicit approval authorizes implementing that revision; code review is a
separate gate. Keep design alternatives here rather than duplicating normative
wire tables. S3's pairing protocol and V2's IRQ restore contract are future
spec-review work, not silently committed by this discussion.
