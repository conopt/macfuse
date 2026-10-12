# NTFS throughput investigation and benchmark plan

Date: 2026-10-12. Status: source review complete; experiments not executed.

## Established result and objective

The user rigorously tested a single large file on the same storage connection:

- Built-in macOS NTFS reading: **more than 900 MB/s**.
- macFUSE and FUSE-T: **less than 350 MB/s** each.

Accept these results as established. The investigation must locate and quantify
the additional cost in the third-party paths. It must not revisit the validity of
the measurements or prescribe upgrades, paid drivers, or unmeasured tuning.

At reference rates of 350 and 900 MB/s, the slower path takes approximately
1.75 additional seconds per decimal GB. Explain that budget through measured
processing, device-I/O, transport, and scheduling costs. The exact deficit depends
on the user's measured rates.

Read performance is the first target. Treat write performance as a separate
experiment because allocation, metadata updates, and durability introduce costs
that native read performance does not measure.

## Source revisions and evidence boundary

This checkout was cloned recursively from `https://github.com/macfuse/macfuse.git`.
Its upstream base is macFUSE 5.4.0:

| Component | Inspected revision |
| --- | --- |
| macFUSE umbrella | `4852a23cf56e2e9ace90ff669c68b5f860dc1d82` |
| Library-2 | `fe59bf43f1a2f249ed8d82c7d28739121ee703a2` |
| Library-3, available in checkout | `00e9044ac2b6349eab5cd9dc8792e1a0544e9e54` |
| Mount | `68f082dc7b6b51aefdf99fafdb8ed1c74928b79a` |
| Framework, available in checkout | `dd962a4388cf2ebd32ff5bd6df4ed75be950608a` |
| NTFS-3G 2026.9.28 | `7f0f841fc52cf719106c5c93bafe465004e36816` |
| FUSE-T libfuse | `0daf8ea8799b82457548673fc8f2511f9b528b90` |
| FUSE-T NTFS-3G fork | `f0e5cb0274e30334dd03aefb4fd5a14c239e6756` |
| NTFSKit architectural comparison | `b7153a8dd51b895d0a87345c6ad8e95bda963ed3` |

The detailed macFUSE read-path review concerns Library-2. Before instrumenting,
record the exact installed NTFS executable, loaded libraries, compiler/build
configuration, macOS version, backend, and effective mount options. Match source
to those binaries; inspect any differences from these revisions. If the tested
filesystem implementation differs from NTFS-3G, trace its equivalent callbacks.

The current macFUSE kernel/FSKit engines are not included in this public checkout.
The inspected FUSE-T public repository does not contain its backend server
implementation. Public library and transport code establishes mechanisms;
runtime observations must establish their cost and costs inside unavailable
components. Do not substitute historical kernel source for current implementation
evidence or infer missing server defaults from documentation.

## Source-backed suspects

### Repeated attribute and extent-map setup

For ordinary files, `ntfs_fuse_read()` parses the path, resolves the inode, opens
the NTFS data attribute, calls `ntfs_attr_pread()`, conditionally updates access
time, and closes the attribute/inode. It does not retain the ordinary data
attribute in the open file handle.

`ntfs_attr_open()` allocates a new attribute. `ntfs_attr_find_vcn()` maps an absent
runlist through `ntfs_attr_map_runlist()` and mapping-pair decompression.
`ntfs_attr_close()` frees that runlist. Repeated extent decoding is therefore a
specific candidate even for sequential reads of one file.

Pathname and inode caches exist. Reopening an inode does **not** establish that
each read performs physical MFT I/O. Measure cache hits and metadata device I/O
separately. At the inspected revisions, upstream and FUSE-T fork
`src/ntfs-3g.c` compare byte-identical.

Sources: [callback](https://github.com/tuxera/ntfs-3g/blob/7f0f841fc52cf719106c5c93bafe465004e36816/src/ntfs-3g.c#L1556),
[attribute mapping/lifetime](https://github.com/tuxera/ntfs-3g/blob/7f0f841fc52cf719106c5c93bafe465004e36816/libntfs-3g/attrib.c#L410),
[inode cache](https://github.com/tuxera/ntfs-3g/blob/7f0f841fc52cf719106c5c93bafe465004e36816/libntfs-3g/inode.c#L465).

### Serialized processing and synchronous disk access

NTFS-3G calls `fuse_loop()`. Both inspected libraries dispatch through a loop
that receives a request and processes it before receiving the next. Ordinary
nonresident file data is read synchronously per extent through `ntfs_pread()`
and ultimately device `pread()`; this inspected path contains no asynchronous
data-I/O pipeline. Kernel read-ahead and queued requests do not themselves make
the NTFS callback or its device calls concurrent.

Sources: [local request loop](Library-2/lib/fuse_loop.c),
[NTFS attribute reads](https://github.com/tuxera/ntfs-3g/blob/7f0f841fc52cf719106c5c93bafe465004e36816/libntfs-3g/attrib.c#L989),
[device reads](https://github.com/tuxera/ntfs-3g/blob/7f0f841fc52cf719106c5c93bafe465004e36816/libntfs-3g/unix_io.c#L298).

### Allocation and transport paths

- macFUSE `fuse_fs_read_buf()` can borrow transport-provided reply memory for an
  ordinary high-level read; otherwise it allocates the data buffer. Reply copying
  skips an actual memory copy when source and destination pointers match.
  Verify negotiated capabilities and the branch actually taken.
- macFUSE's `DeviceTransport.nextMessage()` allocates `0x2001000` bytes
  (32 MiB + 4 KiB) per incoming message, freed with the message. Measure allocation,
  VM, and page-fault costs; do not count this as 32 MiB of touched/copied bytes.
- FUSE-T's ordinary high-level read allocates a payload buffer. Its channel frames
  messages using separate header/body `recv()` calls, serial dispatch queues for
  receiving/sending, and `writev()` replies. Its mount code forwards `rwsize` to
  the server; the resulting request size must be observed.

Sources: [local read-buffer selection](Library-2/lib/fuse.c),
[local reply handling](Library-2/lib/fuse_lowlevel.c),
[local copy implementation](Library-2/lib/buffer.c),
[local receive allocation](Mount/Mount/Channel/Device/DeviceTransport.swift),
[FUSE-T channel](https://github.com/macos-fuse-t/libfuse/blob/0daf8ea8799b82457548673fc8f2511f9b528b90/lib/fuse_kern_chan.c),
[FUSE-T read buffers](https://github.com/macos-fuse-t/libfuse/blob/0daf8ea8799b82457548673fc8f2511f9b528b90/lib/fuse.c#L2124).

## Phase 1: instrument the existing slow path

Keep the user's storage, file, consumer, and OS as the reference environment.
Capture the active backend rather than inferring it from installed packages.

Collect aggregate counters and sampled timing with no per-request terminal
logging. Compare instrumented and uninstrumented execution to quantify observer
overhead. Use a monotonic clock and correlate request identifiers/offsets across
observable boundaries.

| Measurement | Purpose |
| --- | --- |
| Actual READ size histogram, offsets, bytes, and requests/s | Detect splitting and establish fixed work per request |
| LOOKUP/GETATTR/SETATTR/WRITE/FSYNC counts during reads | Identify ancillary traffic and read-induced writes |
| Path lookup, attribute open/close, mapping-decode counts and exclusive timings | Quantify repeated setup and cache behavior |
| Device I/O sizes, offsets, counts, latency, and data/metadata classification | Separate file transfer from metadata and small I/O |
| Callback entry/exit, reply enqueue, and consumer completion | Bound engine versus bridge/scheduling time |
| Per-thread CPU, blocked time, and actual device-I/O overlap | Separate CPU ceilings from serialized waits |
| Allocations, allocated sizes, page faults, and sampled copy work | Quantify buffering cost without assuming copies |
| Reply-buffer capability and branch counters | Verify whether the shared-buffer path is active |

Attribute timings exclusively: do not add an inclusive `ntfs_attr_pread()` time
to the device reads and extent decoding nested inside it. A reply enqueue is not
consumer completion; report unobserved backend time as a residual, supplemented
by process/system profiling. Retain distributions and outliers as well as means.

For illustration only, 64 KiB requests at 350 MB/s imply about 187 microseconds
per request, compared with 73 microseconds at 900 MB/s. Substitute the measured
request distribution before assigning a latency budget.

## Phase 2: causal controls

Use the same consumer and matched buffers, offsets, byte counts, and effective
request sizes where the interfaces permit. Verify which bytes reached the
callback/device; report cache-served throughput separately. These experiments
locate the ceiling, rather than retest the accepted native-read result.

| Experiment | Removed cost | Decision informed |
| --- | --- | --- |
| Memory-backed file through each bridge | NTFS processing and disk I/O | Whether dispatch/transport/buffering alone reproduces the ceiling |
| Minimal FUSE callback reading the same file's immutable extents, mapped once | Repeated NTFS metadata and mapping work | Whether the bridge plus this device path can exceed the ceiling |
| Direct `libntfs-3g` file reads without FUSE | Bridge and FUSE dispatch | Whether the NTFS/device path independently reproduces the ceiling |
| Direct NTFS reads with attribute retained versus reopened per request | Only attribute lifetime changes | Cost of allocation and extent-map reconstruction |
| Read-only versus read/write mount with access-time and metadata-write counters | Conditional mutation during reads | Whether dirty metadata contributes materially |

For the extent control, preserve NTFS logical-to-physical mapping, initialized
length, holes, and EOF semantics. Use an immutable test file/volume so saved
extents cannot become stale; do not raw-write a mounted filesystem. Treat sparse,
compressed, encrypted, and resident files as separate cases rather than silently
using the ordinary-file fast path for them.

## Phase 3: request size and overlap

After identifying the expensive layer, sweep:

- Effective request sizes: 64, 128, 256, and 1,024 KiB.
- Client queue depths: 1, 4, and 16.
- The currently used backends first, followed by macFUSE kernel/FSKit and FUSE-T
  backend variants where supported by the test environment.

Backend options and application buffer sizes are inputs, not measurements.
Record actual request splitting, negotiated limits, and concurrent device calls.
More application threads may only queue more work at the same serial callback.
Do not run a full combinatorial matrix until an earlier result justifies it.

Use repeated steady measurement windows; retain raw run-level results, medians,
spread, and tracing configuration. Verify data correctness outside timed regions
so checksum computation does not become the throughput ceiling.

## Conditional improvement experiments

These are angles to test after attribution, not recommended fixes. Each prototype
must change one mechanism and preserve the production reference run.

| Observed dominant cost | Experimental change | Required evidence |
| --- | --- | --- |
| Attribute setup/extent decoding | Retain per-open attribute and extent map | Reduced setup count/time and correct lifetime/invalidation |
| Fixed overhead per request | Larger effective requests or coalescing | Lower requests/GB; unchanged data correctness and bounded memory |
| Device waiting without overlap | Bounded read-ahead or overlapping immutable data reads | Increased actual device overlap and reduced wall time |
| Allocation/VM activity | Reusable receive/payload buffers | Fewer allocations/faults and measured end-to-end gain |
| Reply transport | Shared-buffer use or batching | Observed path/copy change and measured gain |
| Payload crossing dominates | Native FSKit extent-offload prototype | Fewer payload upcalls and preserved metadata/data semantics |

Do not replace the dispatcher with `fuse_loop_mt()` blindly. Shared volume,
inode-cache, attribute, and mutation state need a concurrency design. A possible
experiment keeps mutable metadata serialized while overlapping bounded data I/O
using stable extent snapshots.

For the offload angle, source exists in
[NTFSKit's blockmap implementation](https://github.com/whereteam/ntfskit/blob/b7153a8dd51b895d0a87345c6ad8e95bda963ed3/NTFSModule/NTFSVolume.swift#L820)
and its
[C mapper](https://github.com/whereteam/ntfskit/blob/b7153a8dd51b895d0a87345c6ad8e95bda963ed3/NTFSModule/bridge/ntfs_bridge.c#L540).
It packs extents for kernel data I/O while serializing engine calls. It still
opens/closes attributes per mapping. This is an architectural reference, not
evidence of its performance or suitability for the user's disk.

## Separate write matrix

Use a disposable test volume for mutations. Measure:

1. Overwrite of a fully initialized file of fixed size.
2. Sequential growth of a new file.
3. Each case through transfer completion, then through close and explicit flush.

Count allocation/runlist changes, initialized-size updates, metadata writes,
data writes, and full-device sync calls. The inspected NTFS-3G `fsync` callback
syncs the device; its macOS device backend uses `F_FULLFSYNC` with a fallback.
Preserve durability while comparing variants, and distinguish buffered acceptance
from completed device writes. Test Windows readability and content correctness
after each experimental variant.

## Userspace installation path and its relevance

The [local backend selector](Library-2/lib/mount_darwin.c) parses
`backend=fskit` and routes that choice through `fuse_mount_ext()`; otherwise it
selects `fuse_mount_core()`. The
[local mount API](Mount/Mount/Mounter.swift) represents FSKit through an XPC
endpoint and distinguishes it from the kernel backend.

Apple's kernel-extension policy explains the Recovery/Reduced Security and
restart workflow: third-party kernel code is included in the boot-time Auxiliary
Kernel Collection, and changing that collection requires approval and restart.
The NTFS filesystem process already runs in userspace; the traditional macFUSE
bridge is the part that requires kernel permission.

On supported current macOS, macFUSE's FSKit backend provides a userspace route
without that kernel-extension security change. The selector is
`-o backend=fskit`; this is an option for the filesystem's mount invocation, not
a standalone mount command. The filesystem must use a compatible macFUSE
library and propagate the option. A wrapper may continue to select the kernel
path by default. FSKit extension enablement is separate from Recovery approval.

FUSE-T also offers a kernel-extension-free route through its userspace bridge.
The user's FUSE-T throughput result is already established; avoiding a kernel
extension does not itself eliminate NTFS processing or transport overhead.

Installation-policy sources, used for policy rather than performance evidence:
[Apple kernel-extension policy](https://support.apple.com/guide/security/securely-extending-the-kernel-sec8e454101b/web),
[macFUSE installation](https://github.com/macfuse/macfuse/wiki/Getting-Started).

## Deliverables and execution boundary

- A manifest of binaries, source revisions, backend, OS, and mount options.
- A stage-cost budget and request/device-I/O histograms explaining the deficit.
- Results from the isolation controls and only the justified parameter sweeps.
- A ranked list of improvement angles tied to measured costs.
- For any subsequent authorized prototype: before/after throughput, correctness,
  memory/CPU costs, and write-durability results where applicable.

This document authorizes no live-machine changes. Benchmark harnesses and
instrumented builds remain future implementation work; manage their dependencies
through a project Nix development environment. The source review and this plan
do not require building or installing macFUSE on the Linux host.
