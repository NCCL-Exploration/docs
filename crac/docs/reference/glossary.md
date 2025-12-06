# Glossary

This document defines key terms used throughout CRAC documentation.

## A

### Application
The user program being checkpointed and restarted. In CRAC context, typically a CUDA application.

### Area
Memory region in process address space. Used by DMTCP's `ProcSelfMaps` to represent memory mappings.

## C

### Checkpoint
The process of saving the complete state of a running application to disk, including CPU memory, GPU memory, and system resources.

### Checkpoint Image
The file(s) created during checkpoint containing the saved application state. In DMTCP, typically `ckpt_*.dmtcp` files.

### Checkpoint/Restart
The general technique of saving application state at one point in time and restoring it later.

### Context (CUDA)
CUDA execution context that contains device configuration, memory allocations, and other state. Similar to CPU process context but for GPU.

### Coordinator (DMTCP)
Central process that manages checkpoint/restart operations across multiple processes. Handles coordination and timing.

### CRAC
Checkpoint-Restart Architecture for CUDA. The DMTCP plugin that enables CUDA checkpointing using native NVIDIA APIs.

### CUDA
Compute Unified Device Architecture. NVIDIA's parallel computing platform and programming model.

### CUDA Driver API
Low-level CUDA API for fine-grained control over GPU operations. Used by CRAC for checkpoint operations.

### CUDA Runtime API
High-level CUDA API that provides convenience functions for common operations.

## D

### Device Memory
Memory allocated on the GPU for use by CUDA kernels. Must be checkpointed and restored.

### DMTCP
Distributed MultiThreaded Checkpointing. User-space transparent checkpointing tool for Linux applications.

### DMTCP Plugin
Shared library that extends DMTCP functionality. CRAC is implemented as a DMTCP plugin.

## E

### Event Hook
Function called by DMTCP at specific points during checkpoint/restart process. Used by plugins to perform custom actions.

### Eventfd
Linux system call for event notification. Used by CRAC for file descriptor wrapping.

## F

### File Descriptor (FD)
Integer handle for open files, pipes, sockets, and other I/O resources in Unix/Linux.

## G

### GPU
Graphics Processing Unit. In CUDA context, the parallel processor that executes CUDA kernels.

### GPU State
Complete state of the GPU including memory contents, contexts, streams, and other resources.

## I

### IPC
Inter-Process Communication. Mechanisms for data exchange between processes. Pipes are a common form of IPC.

## J

### JASSERT
DMTCP's assertion macro for error checking and debugging. Used throughout CRAC code.

## K

### Kernel (CUDA)
Function executed on the GPU. Parallel computation unit in CUDA programming.

## M

### Memory Mapping
Association of virtual memory addresses with physical memory or files. Represented in `/proc/self/maps`.

### MTCP
Minimal Thread Checkpointing. The low-level checkpointing library used by DMTCP.

## P

### Pipe
Unix IPC mechanism for one-way data flow between processes. CRAC preserves pipes across checkpoint/restart.

### Plugin
See DMTCP Plugin.

### Presuspend
DMTCP event that occurs before user threads are suspended for checkpointing. Critical for CRAC's GPU checkpointing.

### Process
Instance of a running program. CRAC operates at the process level.

### Process State
Current state of CUDA process (RUNNING, LOCKED, CHECKPOINTED, FAILED).

## R

### Restart
The process of restoring an application from a previously saved checkpoint image.

### Resume
Continuing execution after a checkpoint without killing the process (as opposed to restart from disk).

## S

### Shared Memory
Memory region accessible by multiple processes. Can be System V shared memory or POSIX shared memory.

### Stream (CUDA)
Sequence of operations that execute in order on the GPU. Used for asynchronous execution.

### Suspend
Temporarily stopping execution of user threads during checkpointing.

## T

### Trampoline
Function wrapper that intercepts system calls. Used by CRAC to handle special cases during checkpoint/restart.

### Transparent Checkpointing
Checkpointing that requires no modifications to the application code.

## U

### UVM
Unified Virtual Memory. CUDA feature that unifies host and device memory addressing. Not supported by current checkpoint APIs.

## V

### Vector
In CUDA context, one-dimensional array of data processed in parallel.

## W

### Wrapper
Function that intercepts and potentially modifies behavior of system calls or library functions.

## Acronyms

| Acronym | Full Name | Description |
|---------|------------|-------------|
| API | Application Programming Interface | Set of functions for interacting with a system |
| CUDA | Compute Unified Device Architecture | NVIDIA's parallel computing platform |
| DMTCP | Distributed MultiThreaded Checkpointing | User-space checkpointing tool |
| FD | File Descriptor | Handle for open files/resources |
| GPU | Graphics Processing Unit | Parallel processor |
| IPC | Inter-Process Communication | Data exchange between processes |
| PID | Process ID | Unique identifier for a process |
| UVM | Unified Virtual Memory | Unified host/device memory |

## CUDA-Specific Terms

### Block
Group of threads that execute together on a GPU multiprocessor.

### Grid
Collection of blocks that execute a kernel.

### Thread
Basic execution unit on the GPU.

### Warp
Group of 32 threads that execute together on NVIDIA GPUs.

## DMTCP-Specific Terms

### Checkpoint Interval
Time between automatic checkpoints when using `--interval` option.

### Coordinator Port
Network port used by DMTCP coordinator for communication.

### Plugin API Version
Version of DMTCP plugin interface that plugin implements.

### Restart Script
Shell script generated by DMTCP to restart checkpointed processes.

## System-Specific Terms

### /proc/self/maps
Virtual file showing memory mappings of current process.

### /proc/self/fd
Directory containing symbolic links to open file descriptors.

### mmap
System call for memory mapping.

### munmap
System call for unmapping memory regions.

## Error-Related Terms

### Assertion
Programming construct that verifies expected conditions. JASSERT is DMTCP's version.

### Error Code
Numeric or enumerated value indicating specific error condition.

### Return Value
Value returned by function indicating success or failure.

## Performance-Related Terms

### Checkpoint Time
Time taken to save application state to disk.

### Restart Time
Time taken to restore application state from disk.

### Overhead
Additional time or resources required by checkpointing system.

## See Also

- [Architecture Overview](../architecture/overview.md) - System architecture
- [DMTCP Plugin API](dmtcp-plugin-api.md) - DMTCP terminology
- [CUDA Checkpoint API](cuda-checkpoint-api.md) - CUDA terminology
- [Getting Started](../getting-started/) - Basic concepts in context