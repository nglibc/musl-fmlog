# musl-fmlog: opt-in, lightweight C library logging of kernel FIEMAP extent data, to assist undelete

# RFC

This proposal is for a lightweight (~160 significant lines of new code) logging facility for kernel FIEMAP data, that may assist undelete when files are deleted through musl. The opt-in is via configuration of MUSL_FIEMAP_PATH. For static programs not using musl delete functions, there is a weak symbol check at startup (at most six instructions on x86_64), and otherwise zero size or speed overhead.
Deletion proceeds so that storage is reclaimed normally. No file contents are retained. Recovery remains possible only until the released file data blocks are overwritten. Logging is done on a best effort basis only. Failure to capture or log metadata never changes the deletion result or errno.  This preserves compatibility with existing applications, as logging is not exposed to the caller.

A reference implementation (available as a patch to musl 1.2.6) is intended to make the proposed semantics, cost, and limitations concrete. No log data durability guarantee is made - this is for users and administrators to decide.

## Motivation and System Layering

FIEMAP is a powerful kernel primitive for exposing filesystem extent mappings. For undelete purposes, however, the mapping must be captured before a deleted inode is finally released and its blocks become available for reuse.
The kernel and filesystem correctly provide the extent-mapping primitive without prescribing a recovery policy, logging destination, or persistent record format. These choices belong in a userspace layer built on the filesystem primitives.
Implementing capture independently in applications would provide inconsistent coverage and require applications to concern themselves with low-level recovery. Many command-line tools, services, and low-level utilities simply call unlink() and have no file recovery support.

The system's C library occupies a practical intermediate layer in both the static and shared forms:
-  It observes a broad class of deletion requests at the point where the inode can still be held long enough to obtain its extent mapping-  It requires no kernel, filesystem, or application-specific changes.-  It permits compiled applications to produce a common record format, giving recovery tools one format to target, rather than a multitude of application-specific formats.
Coverage remains deliberately incomplete. Direct syscalls, non-musl programs, io_uring, and other paths can bypass the facility. Applications that intentionally operate at the raw syscall layer retain the corresponding bare kernel semantics, including responsibility for any recovery metadata policy.

## Relationship to Trash

This low level facility complements Trash and is not intended to replace it.

Trash supports reliable recovery by retaining and managing file contents. It is consequently a heavier-weight operation that delays storage reclamation.  The proposed facility instead frees storage normally and retains only bounded metadata that may assist last-resort recovery. It fills the gap between preserving the complete file and retaining no recovery information.
Trash should remain the recommended mechanism whenever recovery from retained file contents is required, particularly in the desktop environments. However, Trash is not consistently used by command-line tools, services, low-level utilities, or applications that invoke deletion functions directly.
Automated development tools - including build systems, cleanup scripts, package managers, and AI coding agents - can execute command-line deletion outside desktop Trash workflows. As such automation becomes more common, a low-overhead, best-effort recovery log provides an additional safety net against unintended deletion without retaining full file contents.
## Proposed Semantics

MUSL_FIEMAP_PATH is read at process startup. Runtime changes have no effect, and secure-execution processes ignore it.

For a successful deletion, musl holds a supported target inode across the deletion, obtains bounded inode and FIEMAP metadata, appends it to the configured log file, and then releases the inode. Failure to log metadata or FIEMAP data is ignored. The requested deletion’s result and errno remain authoritative.

The log write destination is selected entirely by the user or administrator.

## Cost and Implementation Size

The reference implementation consists of approximately 160 lines of core implementation code (1.2Kb of object code), and 3 lines of call-site code in each deletion wrapper function, and one conditional weak initialization call during startup. The implementation is therefore largely isolated from existing deletion and startup logic.

For static applications that do not use the affected deletion functions, the implementation, environment scan, and MUSL_FIEMAP_PATH string are not linked, resulting in effectively zero size or performance impact.

When the implementation is linked but disabled, the only startup cost is one environment scan. No log file is opened and no additional filesystem operations are performed. Each deletion function call then performs an integer state check before proceeding directly to the deletion syscall.

When enabled, its main work is holding and identifying the target, issuing bounded FIEMAP queries, and performing one log write for each bounded page of extent metadata. The destination may be persistent storage or volatile storage such as tmpfs, depending on the desired performance and retention characteristics. No fsync, durability acknowledgement, or forced FIEMAP synchronization is performed.

## Reference Implementation Scope

The reference implementation is contained in the patch "0001-unistd-add-opt-in-FIEMAP-logging-on-deletion.patch".

It covers
- unistd: unlink/unlinkat
- stdio: remove
- Regular files
- Absolute paths and paths relative to dirfd
- Successful deletions only
- Bounded paginated FIEMAP capture

It performs no allocation, stdio operation, durability synchronization, or FIEMAP_FLAG_SYNC request.

The log file is a compact native binary format. Each record contains inode metadata and path, and up to 32 FIEMAP extents. Capture is limited to 256 pages per deletion, or 8192 extents. Truncated mappings are marked explicitly.
The log path can define the administrative recovery domain. For example, an administrator can select a different log path for each boot and retain corresponding filesystem and mount information outside libc.

## Cached Log File Descriptor

The log file is opened once during process startup and retained as a close-on-exec descriptor.

This follows the descriptor-management pattern already used by musl’s syslog implementation, which also retains an internal descriptor for a logging endpoint. As with that existing mechanism, application code that indiscriminately closes or replaces unknown descriptors can interfere with libc-managed logging state.

## Limitations

The facility is explicitly best effort only, as it is intended to complement (and not replace) Trash.

Capture may be missed or incomplete because of:
- Direct deletion syscalls - applications may bypass libc wrappers and perform bare unlink  
- Non-musl programs
- Released blocks being overwritten before recovery
- Log write failure or partial writes
- Rename replacement (but the same logging mechanism can be used if coverage is desired)
- Filesystems without useful FIEMAP support (eg. logging not done for tmpfs)
- Files that are deletable but cannot be opened for FIEMAP capture- Delayed allocation or otherwise incomplete FIEMAP results
- Pathname replacement races
- io_uring

Device and inode numbers are intended for correlation within the administrator-selected recovery domain. They are not presented as persistent filesystem identities across reboots or device reconfiguration.

## Questions for Review

The practical question is whether the recovery value and broad coverage justify implementing this capability at the libc layer. The libc layer permits a simple implementation of approximately 160 significant lines of self-contained code, requiring no kernel, filesystem, or application changes. For static applications that do not perform deletion operations, the overhead is effectively zero (a single weak-symbol check during startup).

The alternatives are generally more complex or less beneficial: interception mechanisms in other layers, application-specific support that is inherently incomplete and inconsistent, or foregoing last-resort undelete metadata entirely.

Feedback is particularly welcome on:
- The overall cost-benefit tradeoff
- The held-inode approach
- Capture bounds, currently set to 8192 extents & if this should be remain a compile time policy or become configurable through the environment in a future revision- Extent paging size (higher EXT_MAX reduces paging, but increases stack memory use)
- Binary log file format- Other meta data for logging (eg. using statx instead of stat for the birth time stamp)- Whether to use FIEMAP_FLAG_SYNC- Use of a cached log file descriptor - has there been any learning with this approach in syslog
