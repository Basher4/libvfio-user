# Message IRQs, single-socket TCP, and optional TCP twin sockets

Status: design revision and **S1 specification awaiting user confirmation**.
No feature code is implemented. This supersedes the earlier required-twin-socket
plan. [vfio-user.rst](vfio-user.rst) is the normative specification;
[tcp-protocol.md](tcp-protocol.md) explains the proposed choices;
[tcp-progress.md](tcp-progress.md) records approvals and actual execution.

## Outcome and sequencing

1. Extend the protocol with negotiated message-based IRQ delivery, then implement
   and test it on the existing SOCK transport. It must work without twin sockets.
2. Implement TCP with one socket for both directions, behind `tran-tcp=false` by
   default, and add a dedicated TCP client/server example.
3. Add optional TCP twin-socket negotiation in a separate patch/stage, retaining
   single-socket fallback. Extend the same examples with an optional twin mode.
4. Run the full regression matrix and finish documentation. Migration remains v2.

Do not require twin-socket capability for TCP or IRQ-message negotiation. Twin
sockets are an optimization, not the foundation of correctness. No QEMU code
changes are included; document the integration contract and distinguish sample
validation from actual guest/VMM validation. Existing sample programs stay intact.

## Specification-first workflow and versions

For EVERY stage, first record a spec-impact assessment. When the stage changes
wire format, negotiation, or observable protocol semantics, update
`docs/vfio-user.rst`, present the diff, and wait for the user's explicit approval
before ANY implementation code for that stage. Revisions restart this approval
gate. An independent agent may review but cannot approve the spec for the user.
After implementation, stop again for code review. User-approved spec changes do
not automatically approve code or authorize the next stage.

Upstream 0.2 is the baseline, independent of any checkout or commit. Propose
message IRQs in 0.3. Single-socket TCP needs no new wire version: the base protocol
already contemplates AF_INET and bidirectional messages. Recommend 0.4 for the
later TCP twin-socket extension, which adds endpoint discovery/association on the
wire. Each addition must say "added in version X.Y" in the spec. Capabilities
remain mandatory even at the corresponding version; implementation patch count
is not a reason to bump protocol minor versions.

## S1: message IRQ specification and existing-transport implementation

The current RST draft contains the first protocol amendment; stop for user
review of it before implementing these items.

- Add minor 0.3 and explicit `irq_message.supported` agreement; persist the
  negotiated minor and capabilities, clear them on disconnect. Neither version
  nor capability alone enables the extension or configures a vector.
- Keep SET_IRQS client -> server. Add DATA_MESSAGE configuration with optional
  DATA_BOOL selection: 1 configures that vector, 0 leaves it unchanged. Preserve
  no-FD DATA_EVENTFD deassignment and all legacy control semantics. Validation
  must be atomic across the range and use overflow-safe bounds.
- Add a no-reply IRQ_TRIGGER event with existing IRQ addressing and NONE/BOOL
  trigger forms. It works on one socket. The draft deliberately places IRQ
  events on the MAIN socket even when DMA uses a twin channel; serialize events
  with configuration/reset replies. Review that routing choice as part of S1.
- GET_IRQ_INFO's MESSAGE bit indicates support in the negotiated session, never
  configured state. Preserve actual counts and separate disabled from masked.
  Message delivery is suitable for INTx/MSI/MSI-X/ERR/REQ if advertised; it does
  not emulate missing device interrupt sources or PCI vector-table logic.
- Track per-index/vector delivery mode and mask/INTx automask state. Set INTx
  automask before publishing an event; suppress repeats while masked. Unmask
  clears it and invokes the device callback to re-evaluate the actual level.
  Do not replay a source already cleared. Apply the spec's reply-before-new-event
  rule to callbacks that trigger synchronously during configuration/unmask.
- Preserve `vfu_irq_trigger(ctx, subindex)`. Track the active PCI IRQ index;
  add `vfu_irq_trigger_index(ctx, type, subindex)` for explicit index addressing,
  including ERR/REQ. Do not silently guess the type from vector number. Preserve
  eventfd behavior with regression coverage, including owned-FD cleanup.
- Add an internal no-reply transport send operation independent of the DMA
  channel. A successful send is not an acknowledgement of guest handling.
- Reset, disable, mode replacement, and disconnect must obey the spec's IRQ
  ordering and cleanup rules. Client main-channel dispatch must process events
  and control replies in stream order, maintaining the proper vector association.

S1 acceptance: old/new version and capability combinations; with/without Unix
twin socket; support bits unchanged by configure/disable; subset configuration
and zero-bit preservation; bad sizes/flags/ranges/directions; no-FD deassignment;
no IRQ replies; explicit masks, INTx suppression/EOI/reassert, MSI/MSI-X addressing,
reset/re-enable ordering; existing IRQ regression tests. Test IRQ interleaving
while a client waits for another reply. No TCP code is needed to prove this step.

## S2: one-socket TCP backend and dedicated examples

Spec assessment: no new command/version is needed. Before coding, add/review
normative clarification of TCP FD-free operation and single-channel dispatch in
RST if the base text leaves gaps. Record and present that assessment; changes
require explicit user confirmation just like S1.

### Build, endpoints, and lifecycle

- Add Meson `tran-tcp` boolean default false. Guard backend, TCP examples/tests.
  IRQ-message support is transport-independent and not behind this build flag.
- Append VFU_TRANS_TCP without changing existing enum values; keep the enum in
  disabled builds and return ENOTSUP from creation when the backend is absent.
- Keep `vfu_create_ctx()`'s signature. The one-socket endpoint is `host:port` or
  `[IPv6]:port`. Require an explicit host and numeric port. Resolve the hostname,
  choose the first AF_INET/AF_INET6 stream address returned by the resolver,
  and bind once. After a bind/listen failure, fail creation with the originating
  error; do not retry another address or substitute another port. Callers wanting
  a specific interface should pass its numeric address.
- Explicit port 0 requests an OS-assigned ephemeral port; it is never a fallback
  from failure to bind a fixed port. The application chooses fixed vs ephemeral.
  Until attach, the listening FD allows `getsockname()` to discover the selected
  port. Context creation succeeds only after its requested listener is listening.
- Preserve listen/attach/poll/detach/reconnect behavior. `vfu_get_poll_fd()` returns
  the listener before attachment and connection afterward. Never unlink a TCP
  endpoint. Failures release every resource allocated during creation/attachment.
- Preserve nonblocking request progress across partial headers/bodies; dispatch
  only complete frames. Incrementally negotiate on nonblocking attach. Document
  that DMA and backpressure-sensitive reply sends remain synchronous; this is not
  a promise that all APIs/callbacks are asynchronous.

### Framing and full-duplex progress

- Share internal FD-free framing/endpoint helpers with the new client; no new
  public framing ABI. Handle fragments, coalesced frames, EINTR, short writes,
  bounds before allocation, MSG_NOSIGNAL, EOF and malformed replies.
- Demultiplex commands/replies while awaiting DMA. Retain incoming client commands
  in arrival order until the current device callback/transaction completes; do
  not re-enter reset/unmap/device callbacks from the nested DMA wait. Match the
  DMA reply by type/ID/command, not by assuming the next frame is a reply.
- Queue request data with a finite byte budget (16 times SERVER_MAX_MSG_SIZE).
  On overflow terminate the session with a reported resource error rather than
  deadlock or execute reentrant control actions. Treat this as implementation
  resource policy, not a negotiated wire limit. Expose queued work through the
  poll interface: add an internal readiness event/epoll wrapper if needed, and
  keep draining queued work without requiring another network byte to arrive.
- The client must continue answering DMA commands while awaiting ordinary replies
  and IRQs. Neither peer may make answering a DMA request depend on completion
  of a competing client request. Test simultaneous traffic; do not limit the
  library to the simpler ordering used in the example.
- Advertise max_msg_fds=0; suppress BAR mmap/sparse-mmap capabilities, reject
  region-I/O-FD requests, and reject FD-bearing sends. Reuse message DMA maps and
  sgl read/write chunking. FD limit is unrelated to DMA-region count. TCP without
  IRQ-message agreement remains usable for operations needing no interrupts;
  unsupported IRQ configuration fails explicitly, never falls back to eventfds.

### Examples

- Add `tcp-server [-v] host:port` and `tcp-client host:port`, defaulting to a
  single connection. Negotiate message IRQs; use heap-backed guest memory and
  FD-free DMA. Keep existing Unix/PIPE examples unchanged.
- Use a shared BAR0 layout: little-endian guest address u64 at 0, length u32 at 8,
  command u32 at 12, status u32 at 16, IRQ acknowledgement u32 at 20. Command 1
  reads/inverts/writes guest memory; statuses idle=0, busy=1, complete=2, error=3.
  ACK=1 clears the device's interrupt source. Validate widths/ranges, reject
  concurrent work, and bound transfers to 1 MiB.
- Advertise one INTx vector. Configure DATA_MESSAGE. Acknowledge the command
  write, execute recorded DMA work in the event loop, update status, then send
  the completion IRQ. The client services DMA while awaiting replies/events;
  on IRQ read status, verify bytes, acknowledge the source, and send ACTION_UNMASK
  to model EOI. No status polling or counting expected DMA messages.
- Test a transfer larger than the negotiated transfer size. Bound sample waits,
  print actual listener endpoint for harness readiness, clean up child processes.
  GPIO transport selection remains optional future work, not a duplicate sample.

S2 acceptance: TCP on/off and PIPE on/off builds; parsing, occupied port fails
without fallback, port-zero discovery, deterministic resolver tests (no external
DNS), loopback IPv4/IPv6, fragmentation/short writes/EOF/SIGPIPE, queued-command
readiness, DMA failure and concurrency, IRQ completion and INTx EOI. Update
example/developer docs and stop for review.

## S3: optional TCP twin sockets, separate patch

Spec change required: draft the endpoint-discovery/association extension in RST,
proposed version 0.4, then wait for user confirmation BEFORE code. The following
is design direction, not a second normative wire specification:

- Accept `host:primary,secondary` (or bracketed IPv6) to request/support an
  optional second listener; one port continues to work. Both explicitly requested
  listeners must bind/listen or creation fails, cleaning up both. Do not choose a
  substitute port/address. Each explicit zero independently requests ephemeral
  allocation. Before S3 these two-port endpoints are rejected as unsupported.
- Listener count is a local configuration decision made before negotiation.
  Opening a second listener allows offering twin mode; it does not force the
  client to accept it. A peer declining/omitting twin capability uses primary
  only, even when the server has a second listener available.
- Propose a separate `tcp_twin_socket` capability so old Unix `twin_socket`
  fd_index rules remain valid. Server advertises the actual secondary port only
  on mutual agreement. Client connects both channels to the primary peer address.
- Associate secondary to primary using a fresh one-use random session token;
  do not pair by accept order/source IP. Specify the bind command, exact payload,
  error handling, handshake timeout, invalidation, and disconnect behavior in the
  S3 spec draft, not in code first. Do not preallocate its command number in S1.
- Route DMA and its replies over secondary once paired. Per the proposed S1
  contract, IRQ events remain on primary to order them with control replies.
  No twin agreement means ordinary one-socket attach and all traffic on primary.
  Once pairing is agreed, a failed pairing must fail attach, not silently fall back.
- Extend polling/readiness to both channels without losing queued work. Closing
  either member of an established pair invalidates the session. Document how
  applications obtain the two ephemeral bound ports when the poll FD aggregates
  multiple listeners; approve any required local API before implementing it.
- Extend the same examples: server accepts one/two ports, client opts in via
  `--twin-socket` and otherwise uses single mode. Functional tests cover one-port,
  offered-but-declined, and negotiated twin operation. No mandatory twin default.

S3 acceptance: absent/false capability fallback, old minor fallback, one-port
creation, pair bind atomic failure, fixed-port preservation, wrong/stale tokens,
partial association, timeout/reconnect, either-channel EOF, DMA/IRQ/control
concurrency, and unchanged S2 tests. Code review gate follows implementation.

## S4: integration and documentation

Spec assessment required; normally no new behavior/version. Any discovered spec
change returns to user review before its implementation. Run relevant unit,
Python, and functional checks with TCP on/off crossed with PIPE on/off. Check
both connection modes and SOCK message/eventfd compatibility. Publish CLI,
blocking/FD/IRQ limits, and the QEMU integration contract. Never claim guest/VMM
validation from sample tests. No source changes merely to mark a stage complete.

## Review gates and handoff

| Stage | Spec gate (before code) | Code review boundary |
| --- | --- | --- |
| S1 | 0.3 message IRQ amendment; awaiting user confirmation | IRQ negotiation, delivery, ordering and existing-transport tests |
| S2 | Assess/update TCP and duplex progress wording; user approves any amendment | Single-socket TCP, new examples, all S2 tests |
| S3 | 0.4 optional TCP twin negotiation/association; user approves draft first | Optional twin implementation and fallback tests |
| S4 | Assess any remaining changes; user approves amendments first | Regression matrix and documentation |
| V2 | Migration/IRQ restore assessment and spec amendment if needed | Separately reviewed migration implementation |

Keep small commits when requested and record baseline/head or exact file scope.
Every handoff records actual tests, skipped checks, review findings, and approval
state. A code reviewer cannot waive the user's specification gate. Stages are
not permission to implement the whole series in one task.

## V2: stop-and-copy migration extension

Extend the new client with `--migrate-to destination:port source:port`; start the
servers separately. Use existing migration messages on primary, independently
negotiate single/twin mode for each endpoint, and retain one client-owned guest
buffer. No inter-VMM RAM transfer is demonstrated.

1. Stop new guest work, drain DMA and IRQ events, then quiesce the source. Enter
   STOP_COPY and read a versioned fixed-width image to EOF, then enter STOP.
2. Include device registers, completion status, and asserted interrupt source;
   exclude sockets, tokens, descriptors and pointers. Capture client routing,
   injected/pending IRQ and EOI state separately as VMM state.
3. Enter destination RESUMING, stream the image respecting its negotiated chunk
   limit, validate version/length, finalize to STOP, and recreate DMA maps and
   delivery configuration while stopped.
4. Restore IRQ ownership without replay: already injected events remain client
   state; asserted-but-undelivered sources can emit only on resume/unmask. Specify
   automask restoration and the exact image format at V2's spec-first gate; assess
   any necessary API/wire changes explicitly, then obtain user confirmation.
5. Resume destination, compare state, perform more DMA and verify completion IRQ
   and EOI. Test pending INTx, differing chunk limits, malformed images, and
   channel loss. On failure after stopping source, leave it stopped and destination
   inactive; do not silently resume either. Pre-copy remains later work.
