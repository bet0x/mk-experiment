# Only the kernel is fake

mk-experiment/kerf-panel. A control panel for `kerf`, the multikernel
management tool at github.com/multikernel/kerf. Two pieces: `kerfd`, an
HTTP API that holds the privilege, and a static web UI that holds none.
There is no multikernel host yet, so the API ships with a second
implementation of its own host port — a userspace emulator that fakes
the privileged syscalls and the console device, and nothing else.

This is a spec for work not yet started. Nothing is built, no repo is
created. Read it and say where the shape is wrong.

Every claim below about kerf comes from reading the source of
`github.com/multikernel/kerf` at commit `49fd944` (v0.2.0, Apache 2.0),
not from the vendor documentation. Where the two disagree, section 1 says so.

## 1. What kerf actually is

`kerf` is a `click` command group, about 17,000 lines of Python, with
eleven subcommands: `init`, `create`, `update`, `load`, `exec`, `kill`,
`unload`, `delete`, `show`, `dump`, `console`.

It is not the source of truth. The kernel is. kerf writes device tree
blobs into a kernfs filesystem and calls syscalls; the kernel keeps
`/sys/fs/multikernel/device_tree` merged and up to date, and generates
`/sys/fs/multikernel/instances/*` from it.

**It reads:**

```
/sys/fs/multikernel/device_tree              baseline: pool resources
/sys/fs/multikernel/overlays/tx_N/{id,status,dtbo,instance}
/sys/fs/multikernel/instances/NAME/{id,status,device_tree}
/proc/kimage                                 loaded kernels, by MK_ID
/proc/iomem                                  "Multikernel Memory Pool"
/proc/cpuinfo                                logical CPU -> APIC id
/sys/devices/system/node                     NUMA nodes and cpulists
/var/lib/kerf/instances/NAME.json            load provenance kerf records
```

**It writes, four different ways:**

| operation | mechanism |
|---|---|
| `init` | `open("/sys/fs/multikernel/device_tree","wb")`, a whole DTB |
| `create`/`update`/`delete` | DTBO into `overlays/new`; kernel creates `tx_N/` |
| rollback | `rmdir /sys/fs/multikernel/overlays/tx_N` |
| `load` | `kexec_file_load(2)`, flags `KEXEC_MULTIKERNEL(0x10) \| KEXEC_MK_ID(id)` |
| `exec` | `reboot(2)`, cmd `0x4D4B4C49` |
| `kill` | `reboot(2)`, cmd `0x4D4B4C48`, or `0x4D4B4C46` with `--force` |
| `console` | `open("/dev/mktty", O_RDWR)`, then write `"ID\n"` to select |

**The instance state machine** is the kernel's `mk_instance_state`, read
as text from `instances/NAME/status`: `empty`, `ready`, `loaded`,
`active`, `failed`. `exec` refuses anything but `loaded`. `kill` accepts
`active`, and `loaded` too when forced.

**Nothing reports that an instance died, and `status` is not health.**
Of the five states, only `ready`, `loaded` and `active` are ever
assigned — every `mk_instance_set_state()` call in the kernel tree sets
one of those three. `MK_STATE_FAILED` appears twice, in the string table
and as the return of a parse error, and no transition reaches it.
`MK_SYS_HEARTBEAT` is defined at `include/linux/multikernel.h:213` and
is sent and received nowhere. So a spawn kernel that panics leaves its
instance reading `active` for as long as the host stays up. The panel
must label that field kernel state, not health, and must not show a
`failed` badge it can never earn.

**And `status` does not notify its readers.** `mk_instance_set_state()`
carries a TODO at `kernel/multikernel/core.c:324` saying `kernfs_notify()`
should be called on the status file's node and is not. Nothing in
`kernel/multikernel/` calls it. A `poll()` on that file will therefore
never wake, so `/events` reads on an interval — one second, configurable
— and the fake adapter does the same rather than pushing instantly, so
the UI is built against the latency the real host has.

**Devices are lent per PCI function, and the spawn drives them
natively.** `kerf init --devices` takes host names or raw PCI addresses,
classifies four PCI classes — network, storage, display, serial — from
the class code in sysfs, and nothing else about them. The kernel marks a
lent node `status = "reserved"` in the host's live tree, and the spawn
boots with `multikernel,pci-host-bridge` nodes plus an ECAM window, so
`drivers/pci/probe.c` gates probing on `mk_pci_should_probe()` and the
spawn binds its own driver. There is no hypervisor in the path and no
VFIO indirection. Three consequences the panel has to carry: lending is
per `devfn`, so a multifunction card is several separate devices with
nothing grouping them; kerf emits no `driver` property, so it cannot ask
the kernel to detach a host driver even though `mk_do_device_add` can;
and a device returned by a dead instance is not reset, because nothing
in `kernel/multikernel/` calls a reset.

**Three corrections to multikernel.io/getting-started.html**, which is
the page this started from:

1. It shows four subcommands. There are eleven.
2. It says kerf allocates pool memory "through the `/dev/lazy_cma`
   interface". kerf contains no reference to `lazy_cma` at all — zero
   hits across the source. The baseline DTB write is the whole
   interface; the kernel calls the module. The panel therefore never
   touches lazy_cma either, and reports it only as a loaded module.
3. `--initrd` is described as prose but it is a real `load` flag, and
   `load` carries eleven more the page never mentions, including the
   whole static network configuration (`--ip`, `--gateway`, `--netmask`,
   `--nic`, `--hostname`).

**There is no machine-readable output.** `kerf init --format=json`
emits a validation report and that is all. `kerf show` prints to a
terminal. So this project does not wrap an API that exists; it is the
JSON surface kerf lacks.

## 2. The seam is verbs, not paths

The obvious way to build without hardware is to point everything at a
fake directory tree. It does not work here: `load`, `exec` and `kill`
are syscalls, and `console` is a character device. No amount of path
redirection reaches them.

So the abstraction is a port of **operations**, and there are two
implementations:

```
kerfd/host/ports.py     HostPort — the verbs, and the read models
kerfd/host/live.py      LiveHost — real kerf, real syscalls, needs root
kerfd/host/fake.py      FakeHost — fake sysfs tree, fake syscalls
```

**The rule the whole project follows: fake the kernel, and nothing
above it.** Everything in `FakeHost` that is not the kernel is real. Real `pylibfdt`
blobs, real DTBO generation by kerf's own `OverlayGenerator`, real
validation by kerf's own `MultikernelValidator`. A fake `create` that
asks for a CPU another instance holds fails with kerf's genuine error
text, because it is kerf's validator that refuses it.

This buys a UI developed against authentic device tree parsing and
authentic error paths, and it buys the failure cases — a pool that
fills, an `exec` on an unloaded instance, a `tx` that fails to apply —
which are exactly the cases you cannot rehearse on real hardware
without wrecking a host.

## 3. How `LiveHost` drives kerf

Reads go through kerf's library layer, which is already injectable and
needs no adaptation:

```python
DeviceTreeManager(baseline_path=..., overlays_dir=...)
BaselineManager(path)
resources.get_pool_allocated_bytes(iomem_path=...)
resources.get_pool_chunks_from_iomem(iomem_path=...)
topology.cpu_numa_nodes(node_root=..., cpuinfo_path=...)
topology.logical_to_apic(cpuinfo_path=...)
```

Four readers hardcode their paths — `utils.get_instance_id_from_name`,
`utils.get_instance_status`, `runtime.read_instance_dtb`, and
`metadata.KERF_INSTANCES_DIR`. They are three lines each, so `kerfd`
resolves those four itself against a configured root rather than
patching kerf.

Mutations that carry real logic are **not reimplemented**. `create`'s
NUMA-aware CPU auto-allocation, its device alias resolution, `load`'s
bzImage extraction and DAXFS image build are several hundred lines of
policy in the command bodies. Upstream is at v0.2.0 and moving;
duplicating its allocation policy would rot inside a month.

**So kerfd runs each mutation in a forked child, and this is the one
design decision worth stating twice.** The child is not `kerf` the
subprocess. It is a fork of kerfd that imports kerf, calls the command
callback, and writes the result back over a pipe as a structure. That
buys three things a subprocess cannot:

- `ValidationResult` survives. A CLI proxy leaves you an exit code and
  text to parse, and the `errors`/`warnings`/`suggestions` triple that
  section 7 is built on does not survive the pipe.
- A wedged operation cannot take the API with it. `kexec_file_load` on
  a bad image, a `SystemExit` inside a click callback, a segfault in
  `pylibfdt` — the child dies and `/jobs/{id}` records how.
- The child's stdout and stderr *are* the job log, streamed as they are
  written rather than collected at the end.

Reads stay in the parent. `/events` polls once a second (section 1), and
a fork per poll would be absurd.

The two syscall wrappers and the console protocol are module-level
functions with no click in them, so they are imported directly:
`load.main.kexec_file_load`, `exec.main.boot_multikernel`,
`kill.main.halt_multikernel`.

## 4. `FakeHost`, the emulator

`FakeHost` stands in for the kernel's side of every interface kerf
touches, and that is more than the syscalls: three syscalls
(`kexec_file_load` and the two `reboot` commands), two character devices
(`/dev/mktty` and `/dev/dma_heap/multikernel`), three procfs files, the
NUMA sysfs tree, and the whole kernfs filesystem with its overlay
transactions and its instance state machine. The title of this document
is a claim about the other direction: nothing above the kernel is faked.

A directory under `$XDG_STATE_HOME/kerfd/fake/`, shaped exactly like
the real interface, plus the state machine of what the kernel would
have done:

- a DTB written to `device_tree` becomes the pool, parsed back by
  `BaselineManager` so a malformed one fails the way the kernel's would
- a DTBO written to `overlays/new` is validated, assigned the next
  `tx_N`, and written out as `tx_N/{id,status,dtbo,instance}`; the
  instance is then materialized under `instances/NAME/`
- `rmdir tx_N` genuinely un-materializes it, so rollback is testable
- fake `kexec_file_load` moves `ready` to `loaded` and appends a row to
  a fake `/proc/kimage` in the real column format `show` parses
- fake `reboot(MULTIKERNEL)` moves `loaded` to `active` and starts a
  synthetic console; fake `MULTIKERNEL_HALT` moves `active` back to
  `ready`
- memory is really tracked, and fake `/proc/iomem` renders pool chunks
  with their child allocations, so the pool genuinely fills up
- a fault-injection setting makes a `tx` fail to apply, or a faked
  syscall return an error
- that setting can also mark an instance `failed`, which the real kernel
  never does (section 1). It exists to prove the UI degrades safely, not
  because production reaches that state

Fixtures describe one plausible machine — two sockets, four NUMA nodes,
an NVMe and two NICs — and ship as **real DTBs** built by kerf's own
writers, not as JSON stubs. One config value switches the adapter;
the API above the port does not know which one it has.

## 5. The API

`/api/v1`, FastAPI. Reads:

```
GET  /host                         kernel, modules, adapter mode, /proc/kimage
GET  /pool                         cpus{total,host_reserved,pool,free},
                                   memory{chunks,allocated,available},
                                   numa[], devices{}
GET  /topology                     cpu -> {numa,socket,core,thread}, APIC map
GET  /instances                    name, id, status, cpus, memory, devices,
                                   kernel and rootfs provenance
GET  /instances/{name}
GET  /instances/{name}/devicetree  ?format=dtb|dts   (byte-identical to dump)
GET  /pool/devicetree              ?format=dtb|dts
GET  /transactions                 id, status, target
GET  /events                       SSE: state deltas and job progress
GET  /jobs/{id}                    status and log
```

Mutations. Every one returns a job, every one takes `dry_run`, and
every one that has a `kerf` flag exposes it — the full `create` surface
including `cpu_affinity`, `numa_nodes`, `memory_policy`, `memory_base`
and the four `io_uring` settings, and the full `load` surface including
the static network configuration:

```
POST   /pool                       init, or replay an uploaded baseline DTB
DELETE /pool                       init --cpus=none --memory=none
POST   /instances                  create, or replay an instance DTB
PATCH  /instances/{name}           update
DELETE /instances/{name}           delete
POST   /instances/{name}/kernel    load
DELETE /instances/{name}/kernel    unload
POST   /instances/{name}/boot      exec
POST   /instances/{name}/shutdown  kill  {force}
DELETE /transactions/{id}          rollback
WS     /instances/{name}/console   /dev/mktty
```

**Everything mutating is a job.** `load --image nginx:latest` pulls
from a Docker daemon and builds a DAXFS image; that is not an HTTP
round trip. One mechanism for all of them, with a log the UI tails over
SSE, rather than two shapes for fast and slow operations.

**`PATCH` resizes a kernel that is running.** This is easy to miss and
it is the most useful verb in the set. `kerf update` carries no state
check, and the kernel hot-adds to a live instance: the host sends
`MK_RES_MEM_ADD` or `MK_RES_CPU_ADD`, and the spawn calls
`add_memory_driver_managed()` with `MHP_MEMMAP_ON_MEMORY` and onlines the
blocks movable, or calls `add_cpu()`. So the instance detail page can
offer a resize on an `active` instance, in steps of one memory block —
128 MB on x86_64. Growth stays inside the pool chunk the instance
already sits in, because kerf describes one contiguous range per
instance while the kernel tracks a list of them.

**`kerf dump` makes snapshot and replay nearly free**, so the panel
takes them: `GET /pool/devicetree` captures a host, `POST /pool` with
the DTB back replays it. This is the only supported way to move a
configuration between hosts — kerf's `--input` accepts a dumped DTB and
refuses hand-written DTS — so the panel should not invent a second one.

## 6. One writer, and it respects the terminal

kerf serializes with `/var/run/kerf.lock`. Its lock is a file it
touches and unlinks, retried ten times at 100 ms — not `flock`. It
protects against a concurrent `kerf`, but a crash leaves it behind and
nothing cleans it up.

Worse, the unlink is in a `finally`. `SIGKILL` never reaches it, so a
killed operation leaves the lock behind for good.

`kerfd` therefore does four things: serializes every mutation through one
worker so the API cannot race itself, takes kerf's lock anyway so a
`sudo kerf create` in someone's terminal cannot interleave with a panel
operation, surfaces a stale lock as a host warning carrying the holder's
age rather than hanging for one second and failing with kerf's message,
and clears the lock when it reaps a child that held it. The parent knows
the child died, which is exactly the knowledge kerf's own `finally`
lacks. The fork model pays for itself here.

## 7. Errors are the surface, not an afterthought

kerf's `ValidationResult` carries `errors`, `warnings` **and**
`suggestions`, and its validation is mandatory and fail-fast by design.
A panel that reduced that to a red toast would throw away the best part
of the tool.

So `422` returns all three, structured and field-addressed, and
`dry_run` runs the same path. The create form preflights as it is
typed and renders the suggestion next to the field that caused it.

One trap the UI has to carry: `init --cpus` and `create --cpus` take
**physical APIC ids**, not logical CPU numbers. kerf ships
`topology.logical_to_apic` for exactly this. Every CPU the panel shows
is labelled with both, and the forms take APIC ids and say so.

## 8. The web UI

Astro, static build, islands only where they earn it. Served by
`kerfd` itself — the host may have no nginx, and one deployable beats
two.

### Mockup — Overview

```
┌─ kerfd ──────────────────────────────── host mk-lab-01 · live ─┐
│  Pool        CPUs 28 of 32            Memory 16.0 GB           │
│              ████████████████░░░░     ██████████░░░░░░         │
│              22 lent · 6 free         11.5 GB lent · 4.5 free  │
│  Modules     multikernel ✓  lazy_cma ✓  daxfs ✓  mktty ✓       │
├────────────────────────────────────────────────────────────────┤
│  Instances                                          + New      │
│  ● web-server   active   id 1   4 cpu   2.0 GB  nginx:latest   │
│  ● database     active   id 2   8 cpu   8.0 GB  /boot/vmlinuz  │
│  ○ compute      loaded   id 3   8 cpu   1.0 GB  /boot/vmlinuz  │
│  ◌ staging      ready    id 4   2 cpu   0.5 GB  no kernel      │
└────────────────────────────────────────────────────────────────┘
```

### Mockup — Pool

```
  Node 0                              Node 1
  ┌────────────────────────────┐      ┌────────────────────────────┐
  │ 128 129 130 131 132 133 134│      │ 160 161 162 163 164 165 166│
  │  ██  ██  ██  ██  ░░  ░░  ▒▒│      │  ██  ██  ░░  ░░  ░░  ▒▒  ▒▒│
  └────────────────────────────┘      └────────────────────────────┘
  ██ web-server  ██ database  ░░ free in pool  ▒▒ host reserved
  Node 2, Node 3 — host reserved, no pool CPUs

  Memory pool
  0x280000000 ┃██ web ┃██████ database ┃█ cmp ┃▌stg┃░░░░ 4.5G free ┃
  Devices     nvme1 → web-server     enp9s0 → free     nvme0 → host
```

### Mockup — New instance

```
┌─ New instance ─────────────────────────────────────────────────┐
│  Name     [ analytics                  ]                       │
│  CPUs     ( ) explicit APIC ids  [              ]              │
│           (•) count  [ 4 ]   affinity [ compact ▾ ]            │
│           → would take APIC 136-139, node 0, 2 cores × 2 thr   │
│  Memory   [ 4GB ]  policy [ local ▾ ]  base [ auto         ]   │
│  Devices  [x] nvme1      [ ] enp9s0 — held by web-server       │
│  io_uring [ ] enable                                           │
├────────────────────────────────────────────────────────────────┤
│  Preflight (dry run)                                           │
│   ✓ 4 of the 6 free pool CPUs are on node 0                    │
│   ✓ 4.0 GB fits the 4.5 GB free in chunk 0x280000000           │
│   ! memory-policy local is redundant when CPUs are on one node │
│                                      [ Cancel ]  [ Create ]    │
└────────────────────────────────────────────────────────────────┘
```

### Mockup — the autoscaler, on the instance page

```
┌─ web-server   active · id 1 ───────────────────────────────────┐
│  CPUs    APIC 136-139, 4 of 8     Memory   1.125 GiB           │
│  Range   1.0 ───────●──────── 2.0 GiB    headroom 7 blk        │
│                                                                │
│  Autoscale   ● on        cooldown 41s                          │
│    memory PSI  0.23  ██████░░░░   grow above 0.20              │
│    cpu    PSI  0.06  █░░░░░░░░░   grow above 0.50              │
│    last        grew 1.00 → 1.125 GiB, 2 min ago                │
│    reporter    healthy, last seen 4s ago                       │
├────────────────────────────────────────────────────────────────┤
│                       [ Edit policy ]   [ Suspend ]            │
└────────────────────────────────────────────────────────────────┘
```

A policy with no reporter behind it shows `reporter  none` and the
autoscale row reads `unavailable`, because section 12.1 cannot invent a
signal the host is unable to see. A policy whose `max` the chunk cannot
reach is refused when it is written, so that state never renders.

Six pages: Overview, Pool (topology map, memory chunks, devices,
reshape form, device class, and the sibling functions of a
multifunction device with a warning when only some of them are pooled),
Instances (the wizard above), Instance detail (status, the autoscale
panel above,
resources, kernel and rootfs provenance, DTS viewer, its transactions),
Transactions (the log, with rollback), Console. The topology map and
the memory chunk bar are the two real visualizations and go through the
dataviz skill when they are built.

## 9. The console is the one thing not reused

`kerf console` cannot be called from a daemon. It checks
`sys.stdin.isatty()`, puts the terminal in raw mode with `termios`, and
detaches on `Ctrl+]` then `.`. None of that exists in a web request.

What `kerfd` reuses is the protocol, which is small: open `/dev/mktty`
`O_RDWR`, write `"ID\n"` to select the instance, then read and write
bytes. That loop, without the tty handling, is the bridge. It adds the
liveness check kerf does — poll `status`, close when the instance stops
being `active`. `xterm.js` on the other end.

The fd belongs to `kerfd-exec` (section 11), because `/dev/mktty` is the
host's device and no instance can open a sibling's console. The API half
relays bytes and holds no fd.

`FakeHost` serves the same bridge from a pty pair fed by a synthetic
boot log, so the console page is built and tested with no hardware.

## 10. Running it: a socket, and where authority lives

A systemd unit, running as root, because kexec and a kernfs write admit
no alternative.

**It listens on a Unix socket and opens no port.** `/run/kerfd.sock`,
`root:kerf`, mode 0660, `uvicorn --uds`. A loopback bind is reachable by
every local process and by anything that can route to loopback, a
container with host networking included. A socket is reachable by
processes that can open a file, which is a smaller and more legible set.

It buys real authentication rather than a shared secret. `SO_PEERCRED`
on the accepted socket returns the caller's pid, uid and gid as the
kernel knows them — measured here, a 12-byte `ucred` from option 17, and
nothing in user space can forge it. No token in a config file, no token
in a log, no token to rotate.

The browser reaches it through the reverse proxy that is already in front
of everything else on the machine: `reverse_proxy unix//run/kerfd.sock`.
That is also where a browser identity belongs, so kerfd never grows a
login of its own.

**Reaching the socket is not authority.** Group membership gets a
connection and nothing else. `/etc/kerfd/config.toml` maps a uid to one
of three capabilities, and it is the only authority:

```toml
adapter = "live"                  # or "fake"

[capabilities]
read    = ["alberto", "prometheus"]   # every GET
operate = ["alberto"]                 # instance lifecycle, console, rollback
pool    = ["alberto"]                 # init and teardown: the whole machine

[load]
# kerfd runs as root and kexecs the path a request names. This is the
# list of directories it will read a kernel, an initrd or a rootfs from.
allow_dirs = ["/boot", "/var/lib/kerfd/images"]
```

The `[load]` table is not decoration. `POST /instances/{name}/kernel`
hands a path to `kexec_file_load` from a root process, so without an
allowlist any caller who reaches the socket can execute an arbitrary
kernel. No transport fixes that, and a socket mode of 0660 hides it. A
path outside `allow_dirs` is a `403` before any fork happens.

Adding someone to the `kerf` group must therefore not grant kexec. It
grants a connection whose every verb the config still has to name.

## 11. The panel runs in an instance, and the privilege does not

The panel is two programs, not one. That split is unconditional. Where
the second one runs is a deployment choice.

**`kerfd-exec`, on the host.** It does six things and refuses everything
else: write a baseline DTB, write a DTBO, `kexec_file_load`,
`reboot(MULTIKERNEL)` and `reboot(MULTIKERNEL_HALT)`, read the sysfs and
procfs surface, and bridge `/dev/mktty`. It renders no HTML, parses no
browser request, and never speaks to a Docker registry. Its whole surface
is a verb list a person can audit in an afternoon.

**`kerfd-api`, with the UI, in a multikernel instance named `panel`.**
FastAPI, the job queue, the log, the SSE fan-out, the Astro bundle, and
the Docker pull that `load --image` needs. It holds no privilege of any
kind. Compromise it and the attacker holds exactly the executor's verb
list, narrowed further by the capability table in section 10.

This is not decoration. A spawn kernel **cannot** manage the host, and
that is enforced by the kernel rather than by our code: the baseline
write applies to `mk_self`, so a kernel that writes one manages a pool
carved from its own resources. `usage.rst` states it plainly — "a spawn
kernel sees its own instance under `instances/` and can modify itself".
The panel instance can nest children inside itself and can reach nothing
of the host's. So the isolation is structural. It holds even if every
line of `kerfd-api` is wrong.

### 11.1 The channel

`net/vmw_vsock/mk_transport.c` is already in the tree:
`CONFIG_MULTIKERNEL_VSOCKETS`, `AF_VSOCK` between kernel instances over
multikernel's IPI rings and shared memory. The mapping is `CID =
instance_id`, one to one, so the host is CID 0 and the panel dials it.
Two implementation facts the module's own header states: the transport is
a module named `mk_transport`, and a socket must set
`SO_VM_SOCKETS_TRANSPORT` to `VSOCK_TRANSPORT_MULTIKERNEL` explicitly.

`kerfd-exec` therefore listens on both: the Unix socket of section 10,
and vsock. The UDS is not vestigial — section 11.3 needs it.

### 11.2 Identity, again from the kernel

vsock hands the executor the caller's CID, and the CID *is* the instance
id. So the executor knows which instance is asking without being told,
the same shape as `SO_PEERCRED` on the Unix socket: the kernel supplies
identity, the config supplies authority.

That gives the one rule this design needs. **The executor refuses a
mutation whose target is the caller's own instance.** `kill panel` from
inside `panel` is the obvious way to lose the panel, and it is
unanswerable from inside — but from the executor it is one comparison.

### 11.3 Bootstrap, and why the UDS stays

The panel instance cannot exist before the pool does, and the pool comes
from a baseline write on the host. So the panel is a client of the
executor and never a prerequisite for it.

1. systemd starts `kerfd-exec`. It listens on `/run/kerfd.sock`.
2. The first pool is made through that socket, or with `kerf init`.
3. `kerfd-exec` creates the `panel` instance, loads the panel image, and
   starts it. This is an ordinary `create` plus `load` plus `exec`.
4. `kerfd-api` comes up inside it and dials CID 0 over vsock.

A host with no pool therefore still answers on the Unix socket, which is
what makes the machine recoverable when the panel instance is broken.

### 11.4 Where `load` splits

`load --image nginx:latest` does two jobs. It pulls from a registry and
unpacks a tarball, and it builds a DAXFS image through
`/dev/dma_heap/multikernel`. The first job is untrusted network traffic
and archive parsing. The second needs a host device.

So they separate along the seam that was already there: the panel
instance pulls and unpacks into a directory, and hands the executor a
path. The executor builds the DAXFS image and kexecs. The most
attackable part of the most privileged command stops being privileged.

### 11.5 What this costs

A vsock listener, a second image to build and ship, and a `panel`
instance that consumes pool resources to run the thing that manages the
pool. Roughly two CPUs and 1 GB.

It costs nothing in development. Both programs run on the host over the
Unix socket, which is the degenerate case of the same split, and
`FakeHost` sits under the executor either way. The instance deployment
changes one line of configuration and no code.

## 12. An autoscaler, and the number it must not trust

The mechanism for elastic resources is finished. The host sends
`MK_RES_MEM_ADD`, the spawn calls `add_memory_driver_managed()` with
`MHP_MEMMAP_ON_MEMORY` and onlines the blocks movable, and `add_cpu()`
does the same for a CPU. `kerf update` carries no state check, so it
already resizes a running kernel. What is absent is a range to declare
and something to decide. This section is the something.

### 12.1 The signal can only come from inside

The host cannot see a spawn's memory pressure. It is a separate kernel
with its own page tables and its own `/proc/meminfo`, and the host's view
stops at `/proc/iomem`, which records what was allocated and never what
is used. The resource protocol has `MK_RES_QUERY` and no "I need more" —
every flow is host-initiated.

So an agentless autoscaler is not possible here, and no amount of
host-side inference fixes it. The instance has to report.

`kerfd-report` is a small program in the instance's image. It posts
**pressure, not free bytes**: `some avg10` from `/proc/pressure/memory`
and `/proc/pressure/cpu`. Free bytes is close to meaningless once page
cache fills a kernel, and PSI measures the thing that actually hurts,
which is tasks stalled waiting for the resource.

An instance with no reporter is not scalable, and the panel says so on
its page rather than pretending.

### 12.2 The path is a relay, because siblings cannot talk

A spawn cannot reach a sibling. `multikernel_send_ipi_data()` resolves
its target with `mk_instance_find()` against the **sender's** instance
list, and the manifest gives a spawn the ring addresses "in both
directions" for its parent alone. A reporter therefore cannot dial the
panel instance.

It dials CID 0, and `kerfd-exec` relays to the panel. The reporter needs
no configuration and no knowledge of where the panel runs, which is also
what keeps it small enough to put in someone else's image.

### 12.3 The number it must not trust is the free pool

An instance can only grow into the pool chunk it already sits in. kerf
extends in place at `old_base + old_size` and refuses when the extension
leaves the chunk, in its own words: "the overlay names an existing range,
so an instance can only grow into the chunk it already sits in".

So the headroom of an instance is not the free memory in the pool. It is
the free space physically above that instance, inside its chunk. A
controller that reasons about the pool total will promise growth it
cannot deliver, and every one of those decisions ends as a 422.

The controller therefore computes `headroom(instance)` from the chunk map
and the neighbours, and that same number is what the instance page shows
beside the range. Steps are whole memory blocks, 128 MB on x86_64.

CPUs carry no such constraint. Pool `available_free` is a set, and
`add_cpu()` does not care which CPU it gets. The CPU half of a policy is
simple and the memory half is not.

### 12.4 The policy

Per instance, opt in, off by default. `kerfd` stores it at
`/var/lib/kerfd/policies/NAME.toml`, because the device tree binding has
no min or max property to hold it.

**A consequence worth stating: `kerf dump` does not carry the policy.**
A replayed instance returns without one. The panel's own snapshot
includes the policies, and the instance page marks a policy that a dump
would lose.

```toml
[memory]
min = "1GiB"            # never shrink below this, block aligned
max = "2GiB"            # never grow above this
grow_above = 0.20       # memory PSI, some avg10
shrink_below = 0.02
step = "128MiB"         # one memory block

[cpu]
min = 2
max = 8
grow_above = 0.50       # cpu PSI, some avg10
shrink_below = 0.05
step = 1

[limits]
cooldown = "60s"
pool_reserve = "2GiB"   # free pool the controller will not consume
```

`PUT /instances/{name}/policy` validates a policy against the live
headroom and refuses with the reason, so a `max` the chunk cannot reach
is rejected when it is written rather than discovered at 03:00.

### 12.5 The loop

One loop, in `kerfd-api`. The controller holds no privilege and is a
client of the executor exactly as the UI is.

Each tick, for each instance that has a policy:

1. Read the last report. Nothing within two intervals is `unknown`, and
   `unknown` never scales in either direction. A dead reporter must not
   read as a quiet instance.
2. Compare PSI against the thresholds. They are two different numbers,
   never one, so an instance sitting on the boundary does not flap.
3. Clamp the request to `min` and `max`, to `headroom(instance)`, to
   `pool_reserve`, and to one `step` for this tick.
4. Send `PATCH /instances/{name}` with `dry_run` first. A refusal is an
   ordinary outcome. Record it and wait.
5. Apply. It enters the same job queue and takes the same kerf lock as a
   human's resize. The controller gets no fast path.

### 12.6 The operator's hand wins

A manual `PATCH` suspends that instance's policy for one cooldown, and
the page says so. A person resizing an instance while a loop argues with
them is the failure this rule exists to prevent.

`DELETE /instances/{name}/policy` stops it. Deleting an instance deletes
its policy. And the controller cannot scale the panel: section 11.2
already refuses a mutation whose target is the caller's own instance, and
that refusal lives in the executor where the CID proves it.

### 12.7 A shrink is a request, not a command

`offline_and_remove_memory()` fails when anything pins a page in a
movable block. So a shrink is best effort: one attempt, the failure
recorded on the instance page, and no retry before the next cooldown. A
failed shrink is never a reason to try harder.

### 12.8 Every decision is a job

The controller writes the same job a person writes, with its reasoning in
the log:

```
grew web-server 1.00 -> 1.125 GiB
  memory PSI some avg10 = 0.23 (grow_above 0.20)
  headroom 8 blocks, pool reserve intact
```

So the instance timeline shows what decided and why, and a bad policy is
legible afterwards instead of mysterious. No silent background mutation.

### 12.9 The endpoints

```
GET    /instances/{name}/policy
PUT    /instances/{name}/policy     validated against live headroom
DELETE /instances/{name}/policy
GET    /instances/{name}/metrics    the last report, and its age
POST   /instances/{name}/metrics    the reporter's write, relayed by the executor
GET    /controller                  per instance: state, last decision, cooldown left
```

### 12.10 Testing it without pressure

`FakeHost` gains a synthetic reporter, and the tests drive the PSI series
directly. Hysteresis, the clamps, `unknown` handling, a refused grow
against a full chunk, a failed shrink, and the operator override are all
testable with no hardware and no real memory pressure. This is the part
of the panel that would otherwise be untestable, and the fake makes it
the best tested part instead.

## 13. Tests

The contract tests run the whole API against `FakeHost`: every read
shape, every mutation, every refusal, the tx log, rollback, a pool that
fills, a stale lock, and the console bridge. They are written against
the port, so the same bodies can later run against `LiveHost` on a real
host, marked `serial` and skipped when `/sys/fs/multikernel` is absent.

The controller gets the treatment of section 12.10, driven by a
synthetic PSI series.

The executor gets its own tests, and they are refusals rather than
happy paths: a uid with `read` denied every mutation, a `load` naming a
path outside `allow_dirs`, a caller whose CID equals the instance it
asks to kill, and a child killed mid-operation leaving a lock the parent
must clear. Identity is injectable in both listeners, so these run with
no privilege and no hardware.

Test-driven throughout, and the fixtures are generated by a committed
script so a fixture can be regenerated rather than hand-patched.

## 14. What this does not do

**It does not build the kernel.** Getting a multikernel host booting —
`CONFIG_MULTIKERNEL`, `CONFIG_MKTTY`, `lazy_cma.ko`, `daxfs.ko` — is
its own task on its own hardware. `FakeHost` exists so this one does
not wait for it.

It does, however, have to report a host built wrong, because one way is
easy to miss: `kernel/multikernel/dma_heap.c` is gated on
`CONFIG_DMABUF_HEAPS`, and `CONFIG_MULTIKERNEL` does not select it. A
kernel with multikernel enabled and that symbol off exposes no
`/dev/dma_heap/multikernel`, so `kerf load --image` has no heap to build
a DAXFS image from while everything else works. `GET /host` therefore
reports the heap's presence next to the modules, not just the modules.

**It does not manage the host's Docker.** `load --image` hands the
reference to kerf, which talks to the Docker daemon itself through
`/var/run/docker.sock`. The panel shows the pull progress in the job
log and nothing more.

**It does not decide a failover.** The kernel has the primitives and
they are real: a graceful shutdown over the message ring, a forcible one
that arms a host-owned marker in shared memory and NMIs the target's
CPUs, and a fence that runs in the interesting direction — a spawn can
force-halt the host, then confirm through the `parked[]` presence bytes
that every one of the host's CPUs actually reached the park loop, and a
CPU that never parks fails the fence so no later baseline can claim the
machine. `multikernel_allow_emergency_restart()` also refuses to reboot
a machine with a live instance on it. What does not exist anywhere in
these repos is the layer above: nothing probes liveness, nothing decides
a takeover, nothing promotes. The panel therefore reports state and
runs operations an operator chooses, and claims no availability
property. If that layer is wanted later it is a separate spec, and it
would live above this API rather than inside it.

**It does not make a GPU work.** A display controller is a PCI device
like any other here — kerf tags it `pci-display` and lends the function.
Nothing partitions it (no MIG, no vGPU), nothing resets it between
instances, nothing detaches the host's driver, and `MK_DEVICE_BIND_VFIO`
is defined in `include/linux/multikernel.h` and referenced nowhere in
the tree. The panel reports the class and the function siblings, and
says plainly when a pooled function's host driver is still bound. The
rest is upstream's to build.

**It does not place, and it does not restart.** No placement policy —
kerf's affinity and memory policies are exposed as they are, and the
operator chooses. No restart after a failure either, because nothing
reports a failure to restart from, as section 1 records. Section 12
resizes an instance that is running. It never decides where an instance
should have been, and it never brings a dead one back.

**It does not patch kerf.** Where kerf's paths are hardcoded, `kerfd`
resolves them itself. If upstream later takes a `--json` flag or a
library entry point, `LiveHost` gets simpler and nothing above it
changes.

## 15. Files

```
kerf-panel/
  exec/                      # kerfd-exec: the privileged half, on the host
    verbs.py                   the whole vocabulary, one function each
    host/{ports,live,fake}.py  the seam of section 2
    listen/{uds,vsock}.py      SO_PEERCRED and CID identity
    auth/capabilities.py       the config of section 10 is the authority
    fork.py                    one child per mutation
    mktty.py                   the host's console fd
    fixtures/{build.py,*.dtb,proc/*}
    main.py
  api/                       # kerfd-api: unprivileged, runs in the instance
    routers/{host,pool,topology,instances,transactions,jobs,console,events}.py
    schemas.py
    client.py                  dials the executor: UDS or vsock CID 0
    jobs/{queue,log,sse}.py
    pull.py                    docker pull and unpack, deliberately here
    control/{loop,policy,headroom,psi}.py   the autoscaler of section 12
    main.py
  report/                    # kerfd-report: ships inside an instance image
    main.py                    posts PSI to CID 0, needs no configuration
  web/
    src/pages/{index,pool,instances,instance,transactions,console}.astro
    src/islands/{TopologyMap,MemoryBar,CreateForm,ReshapeForm,Console}.tsx
  tests/{contract,unit}/
  packaging/
    kerfd-exec.service  config.example.toml
    panel-image/Dockerfile   # the image the `panel` instance boots
```

## 16. Order of work

1. `FakeHost` and the fixture builder, with the port defined first.
2. `kerfd-exec` over the Unix socket: the verb list, `SO_PEERCRED`, the
   capability config, the `[load]` allowlist.
3. `kerfd-api` and its client, both on the host. The read routes, and
   the contract tests that pin their shapes.
4. Mutations: the fork executor, the job queue, the lock discipline,
   `dry_run` preflight.
5. The UI, read-only: Overview, Pool, Instance detail.
6. The UI, writing: the wizard, the reshape form, rollback.
7. The console bridge, both adapters.
8. The vsock listener, and the refusal of a mutation aimed at the
   caller's own instance.
9. The panel image, and the bootstrap sequence of section 11.3.
10. `headroom()` and the policy store, with the `PUT` that refuses a
    `max` the chunk cannot reach.
11. `kerfd-report`, and the executor's relay of `POST /metrics`.
12. The control loop, against the synthetic PSI series first.
13. Packaging: the unit files, the config, the README, and the first run
    against real hardware when there is some.

## 17. Open questions

1. **Does `/dev/mktty` admit more than one reader per instance?** The
   WebSocket bridge is single-viewer if not, and the panel has to say
   so rather than silently stealing a terminal's console. Unknown
   without hardware; the fake supports both so the UI can be built
   either way.
2. **Should tearing the pool down require typing the host name?** It is
   the most destructive operation in the set and it is one `DELETE`.
3. **The fake's boot log.** Synthetic text is enough to build the
   console page. Replaying a captured real boot would be better and
   costs nothing once there is a host to capture one from.
