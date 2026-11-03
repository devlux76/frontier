# BEAM application code, native boundaries and watchdog

Status: design specification; implementation pending.

## Application and native-code policy

Application code uses Erlang, Elixir or another BEAM-native language.
Native faults require containment, and an external watchdog recovers the
node if it fails. Keep native engines behind an explicit sandbox
boundary, including transitive dependencies and linked-in drivers. An Elixir
API, Rust wrapper or dirty scheduler alone does not create that boundary.

An in-process Rust WASM host remains under evaluation. Native engine
bindings and sandboxed WASM guests have different containment properties:

| Integration | Proposed treatment |
|---|---|
| Native Ladybug NIF, including Rust calling native C++ | Excluded from the main node; use a separate OS process/helper node |
| Ladybug compiled to WASM under a Rust runtime NIF | Evaluation candidate: guest is sandboxed, native runtime/binding/imports remain trusted in-process code |
| Managed BEAM WASM implementation | Evaluation candidate with optional NIFs disabled or separately isolated; prove required features and filesystem semantics |
| Native engine or WASM host in another OS process | Contains direct native address-space faults; also enforce resource and filesystem limits |

WASM guest traps normally become host errors. This is a materially stronger
boundary around the database than a native C++ NIF. It is not a sandbox around
the runtime NIF itself. Rust's memory safety does not guarantee process
survival: its default allocation-error handling can abort, and Wasmtime has
had a patched guest-triggered host-abort bug. That historical bug does not
establish that the current release has the same defect.
[Rust allocation failures](https://doc.rust-lang.org/alloc/alloc/fn.handle_alloc_error.html),
[patched Wasmtime advisory](https://github.com/bytecodealliance/wasmtime/security/advisories/GHSA-q8hx-mm92-4wvg).

Inventory the pinned release's native components, host imports, allocation
limits, cancellation and scheduler/signal integration. Guest memory limits
do not alone bound host compilation, result buffers or BEAM term allocation.
BEAM/ERTS/JIT and the OS remain trusted substrate in every option. Do not
promise that a finite state space or a memory-safe language makes a runtime
incapable of failing. In-process host risk has not been accepted merely by
keeping WASM on the shortlist. Compare boundaries under `frontier:89a29ee`;
details are in [Ladybug isolation](ladybug-isolation.md).

## External watchdog and recovery

An Erlang supervisor cannot restart a dead VM. On Linux, propose an OS service
manager outside the watched BEAM, with failure restart, startup readiness,
bounded stop/restart timeouts and restart rate limits. A systemd service can
use `Restart=on-failure` and `WatchdogSec=`; pin and verify the actual unit's
semantics before deployment. Respect intentional service stops and persistent
explicit holds. A task-level STOP is distinct from stopping the whole service.
[Service/watchdog contract](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html).

Emit a heartbeat only after a fresh, bounded responsiveness check through
the critical control path. A blind timer or live PID can falsely report a
wedged application as healthy. Notification must not introduce an application
NIF; use a BEAM-capable notification path or a bounded external helper whose
attribution and lifetime are tested. The external manager enforces the
deadline even when every Erlang process stops running.

Separate node responsiveness, dependency readiness and work progress. A
database outage or long but progressing inference call does not justify
restarting a responsive control node. Stop effect admission, expose degraded
state and recover the affected dependency. A wedged critical control path
must not keep issuing healthy heartbeats. Diagnose and bound restart storms
rather than repeatedly cycling against corrupt state.

After node restart, keep effect admission closed until store/schema identity,
runtime capabilities and resource ownership are checked and unfinished work
and effect receipts are reconciled. Rebuild ETS from durable state. A
watchdog restores service availability; it does not preserve lost in-memory
state or resolve an uncertain committed write. Never resurrect stale backlog
or clear parked/cancelled work merely because the node restarted.

## Acceptance before deployment

Use disposable nodes and data to prove abnormal exit, SIGKILL and a hung or
SIGSTOP'd VM trigger bounded recovery; intentional service stop stays stopped;
restart storms become visible failures; and a worker-only crash leaves the
main node responsive. Verify unhealthy control cannot fake a heartbeat,
dependency failure is distinguished from node failure, no duplicate database
owner appears, and interrupted effects are reconciled before admission.
Measure detection/recovery time rather than claiming instant or infallible
restarts. Existing desktop power/watchdog policies and live services are
outside this documentation change.
