# Code Walkthrough

This document provides a file-by-file explanation of the CRAC codebase, including key functions, control flow, and interactions between components.

## Repository Structure

```
crac/
├── crac.cpp              # Main plugin implementation
├── Makefile              # Build configuration
├── README.md             # Project overview
├── .gitignore           # Git ignore patterns
└── test/                # Test programs
    ├── Makefile          # Test build configuration
    ├── counter.cu        # Simple CUDA counter test
    ├── counter_mpi.cu    # MPI + CUDA counter test
    └── mpi_cuda.cu      # MPI + CUDA vector addition
```

## Core Plugin: `crac.cpp`

This is the heart of CRAC - the DMTCP plugin that bridges CUDA checkpoint APIs with DMTCP's checkpoint/restart infrastructure.

### File Overview
- **Lines**: 357
- **Purpose**: DMTCP plugin for CUDA checkpoint/restart
- **Language**: C++ with CUDA Driver API integration
- **Dependencies**: DMTCP, CUDA Driver API, system libraries

### Header Dependencies (Lines 1-21)

```cpp
#include <stdio.h>           // Standard I/O
#include <stdlib.h>          // Memory allocation
#include <dlfcn.h>           // Dynamic library loading
#include <unistd.h>          // System calls
#include <string.h>          // String operations
#include <dirent.h>          // Directory operations
#include <errno.h>           // Error codes
#include <fcntl.h>           // File control
#include <sys/types.h>       // System types
#include <sys/mman.h>        // Memory mapping
#include <sys/eventfd.h>      // Event file descriptors
#include <cuda.h>             // CUDA Driver API
#include <cuda_runtime_api.h> // CUDA Runtime API
#include <vector>             // STL vector
#include "dmtcp.h"           // DMTCP plugin API
#include "jassert.h"          // DMTCP assertions
#include "procselfmaps.h"     // Process memory maps
#include "procmapsarea.h"     // Memory area structures
#include "trampolines.h"      // Function wrapping
#include "plugin/ipc/event/eventwrappers.h" // IPC wrappers
```

### Constants and Definitions (Lines 22-26)

```cpp
#define PATH_MAX 4096        // Maximum path length for readlink
#define MAX_PIPE_FDS 1024    // Maximum pipes to track
```

**Purpose**: System limits for buffer sizes and pipe tracking.

### Data Structures (Lines 27-53)

#### Pipe Enumeration (Lines 28-31)
```cpp
typedef enum {
    PIPE_READ = O_RDONLY,     // Read end of pipe
    PIPE_WRITE = O_WRONLY     // Write end of pipe
} pipe_rw_t;
```

**Purpose**: Type-safe representation of pipe access modes.

#### Pipe Information Structure (Lines 34-38)
```cpp
typedef struct {
    int fd;                  // File descriptor number
    unsigned long pipe_inode;  // Pipe inode from /proc/self/fd
    pipe_rw_t rw_type;       // Read or write access
} pipe_info_t;
```

**Purpose**: Stores information about each pipe file descriptor for recreation during restart.

#### Pipe Mapping Structure (Lines 41-46)
```cpp
typedef struct {
    unsigned long old_inode;   // Original pipe inode
    int new_read_fd;          // New read-end file descriptor
    int new_write_fd;         // New write-end file descriptor
    int created;              // Creation flag
} pipe_map_t;
```

**Purpose**: Maps original pipe inodes to newly created pipes during restart.

### Global Variables (Lines 48-53)

```cpp
std::vector<ProcMapsArea> nvidia_device_files;  // NVIDIA device mappings
CUprocessState cuda_state = CU_PROCESS_STATE_RUNNING;  // CUDA state
CUcheckpointRestoreArgs restore_args = {0};     // CUDA restore arguments
int cuda_initialized = 0;                      // CUDA initialization flag
pipe_info_t pipe_list[MAX_PIPE_FDS] = {0};    // Pipe information array
int num_fds_found;                             // Number of pipes found
```

**Purpose**: Global state management across checkpoint/restart cycles.

## Key Functions

### Pipe Inspection Function (Lines 56-92)

**Function**: `inspect_pipes(pipe_info_t *pipe_fd_array, int max_pipe_fds)`

**Purpose**: Scan `/proc/self/fd` for pipe file descriptors and collect their information.

**Algorithm**:
1. Open `/proc/self/fd` directory
2. Read each entry (skip '.' and non-numeric entries)
3. For each file descriptor:
   - Skip stdin/stdout/stderr (0, 1, 2)
   - Read symlink target using `readlink()`
   - Check if target starts with "pipe:["
   - Extract inode number from "pipe:[inode]"
   - Get file access flags using `fcntl()`
   - Store pipe information in array

**Key Code Snippet**:
```cpp
if (strncmp(link_target, "pipe:[", 6) == 0) {
    JASSERT(count < max_pipe_fds);
    unsigned long inode = strtoul(link_target + 6, NULL, 10);
    int flags = fcntl(fd, F_GETFL);
    JASSERT(flags != -1);
    pipe_rw_t rw_type = (pipe_rw_t)(flags & O_ACCMODE);
    JASSERT(rw_type == PIPE_READ || rw_type == PIPE_WRITE);
    // Store pipe information...
}
```

**Return**: Number of pipe entries found.

### Pipe Recreation Function (Lines 94-170)

**Function**: `recreate_pipes(pipe_info_t *pipe_fd_array, int num_fds)`

**Purpose**: Recreate pipes with exact same file descriptor numbers during restart.

**Algorithm**:
1. **Load Real pipe() Function**: Get `pipe()` from libc using `dlsym()`
2. **Reserve FD Locations**: Occupy original FD numbers to prevent reuse
3. **Create New Pipes**: Create one new pipe for each unique pipe inode
4. **Release Reservations**: Free up original FD numbers
5. **Duplicate to Original FDs**: Use `dup2()` to place pipes at original FDs
6. **Cleanup**: Close temporary file descriptors

**Key Code Snippet**:
```cpp
// Reserve sites for the original pipes
int fd_unique = open("/dev/zero", O_RDONLY);
for (int i = 0; i < num_fds; i++) {
    int fd = pipe_fd_array[i].fd;
    JASSERT(fd == fd_unique || fcntl(fd, F_GETFD) == -1);
    dup2(fd_unique, fd); // Occupy fd; Close it later.
}

// Create new pipes for unique inodes
for (int i = 0; i < num_fds; i++) {
    unsigned long old_inode = pipe_fd_array[i].pipe_inode;
    // Check if we already created a pipe for this inode
    if (!found && pipe_map[map_count].created == 0) {
        int new_fds[2];
        JASSERT((*real_pipe)(new_fds) != -1);
        // Store mapping information...
    }
}
```

**Return**: 0 on success.

### Debug Function (Lines 173-190)

**Function**: `print_cuda_process_state()`

**Purpose**: Print current CUDA process state for debugging.

**Implementation**: Simple switch statement printing human-readable state names.

### GPU Checkpoint Function (Lines 195-245)

**Function**: `checkpoint_gpu()`

**Purpose**: Save GPU state using CUDA's native checkpoint APIs.

**Algorithm**:
1. **Initialize CUDA**: Call `cuInit(0)` if not already initialized
2. **Lock GPU Process**: Call `cuCheckpointProcessLock()` to block new GPU work
3. **Scan Device Files**: Find `/dev/nvidia*` mappings in `/proc/self/maps`
4. **Checkpoint GPU**: Call `cuCheckpointProcessCheckpoint()` to save GPU state
5. **Verify State**: Ensure GPU is in `CHECKPOINTED` state
6. **Reserve Memory**: Use `mmap(PROT_NONE)` to reserve device file regions

**Key Code Snippet**:
```cpp
CUcheckpointLockArgs lock_args = {0};
CUresult ret = cuCheckpointProcessLock(pid, &lock_args);
JASSERT(ret == CUDA_SUCCESS)(ret);

// Scan for NVIDIA device files
dmtcp::ProcSelfMaps proc_maps;
Area area;
while (proc_maps.getNextArea(&area)) {
    if (strstr(area.name, "/dev/nvidia")) {
        nvidia_device_files.push_back(area);
    }
}

ret = cuCheckpointProcessCheckpoint(pid, &checkpoint_args);
JASSERT(ret == CUDA_SUCCESS)(ret);

// Reserve device file regions
for (Area device_area : nvidia_device_files) {
    void *ret = mmap(device_area.addr, device_area.size, PROT_NONE,
                     MAP_FIXED | MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
    JASSERT(ret == device_area.addr)(ret)(device_area.addr)(device_area.name);
}
```

**Critical Timing**: Must be called in `DMTCP_EVENT_PRESUSPEND` before user threads are suspended.

### GPU Restore Function (Lines 247-262)

**Function**: `restore_gpu()`

**Purpose**: Restore GPU state using CUDA's native restore APIs.

**Algorithm**:
1. **Release Reservations**: Unmap memory regions reserved for device files
2. **Restore GPU State**: Call `cuCheckpointProcessRestore()` to restore GPU memory
3. **Unlock GPU**: Call `cuCheckpointProcessUnlock()` to resume GPU operations
4. **Verify State**: Ensure GPU is in `RUNNING` state
5. **Cleanup**: Clear device file tracking vector

**Key Code Snippet**:
```cpp
// Release reserved memories for NVIDIA device files
for (Area device_area : nvidia_device_files) {
    munmap(device_area.addr, device_area.size);
}

CUcheckpointUnlockArgs unlock_args = {0};
CUresult ret = cuCheckpointProcessRestore(pid, &restore_args);
JASSERT(ret == CUDA_SUCCESS)(ret);

ret = cuCheckpointProcessUnlock(pid, &unlock_args);
JASSERT(ret == CUDA_SUCCESS)(ret);

ret = cuCheckpointProcessGetState(pid, &cuda_state);
JASSERT(ret == CUDA_SUCCESS)(ret);
JASSERT(cuda_state == CU_PROCESS_STATE_RUNNING)(cuda_state);
```

**Critical Timing**: Must be called in `DMTCP_EVENT_RUNNING` after user threads are resumed.

### Trampoline Functions (Lines 264-279)

**Purpose**: Wrapper functions for system calls that need special handling during checkpoint/restart.

#### eventfd_trampoline (Lines 271-279)
```cpp
static int eventfd_trampoline(unsigned int initval, int flags) {
    UNINSTALL_TRAMPOLINE(eventfd_trampoline_info);
    int retval = eventfd_wrapper(initval, flags);
    INSTALL_TRAMPOLINE(eventfd_trampoline_info);
    return retval;
}
```

**Purpose**: Temporarily disable trampoline during `eventfd()` call to prevent interference with checkpoint/restart.

### Main Event Handler (Lines 281-344)

**Function**: `cuda_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)`

**Purpose**: Main entry point for DMTCP plugin events.

**Event Handling**:

#### DMTCP_EVENT_INIT (Lines 283-286)
```cpp
case DMTCP_EVENT_INIT:
    dmtcp_setup_trampoline("eventfd", (void *)&eventfd_trampoline,
                           &eventfd_trampoline_info);
    break;
```

**Purpose**: Initialize trampolines for system call wrapping.

#### DMTCP_EVENT_RUNNING (Lines 288-307)
```cpp
case DMTCP_EVENT_RUNNING:
    DMTCP_PLUGIN_DISABLE_CKPT();    
    if (cuda_state == CU_PROCESS_STATE_RUNNING) {
        // First launch - no restoration needed
    } else if (cuda_state == CU_PROCESS_STATE_CHECKPOINTED) {
        INSTALL_TRAMPOLINE(eventfd_trampoline_info);
        restore_gpu();
        nvidia_device_files.clear();
    }
    DMTCP_PLUGIN_ENABLE_CKPT();    
    break;
```

**Purpose**: Handle both initial launch and restart scenarios. GPU restoration happens here.

#### DMTCP_EVENT_PRESUSPEND (Lines 309-313)
```cpp
case DMTCP_EVENT_PRESUSPEND:
    checkpoint_gpu();
    break;
```

**Purpose**: Trigger GPU checkpoint before user threads are suspended.

#### DMTCP_EVENT_PRECHECKPOINT (Lines 315-319)
```cpp
case DMTCP_EVENT_PRECHECKPOINT:
    UNINSTALL_TRAMPOLINE(eventfd_trampoline_info);
    num_fds_found = inspect_pipes(pipe_list, MAX_PIPE_FDS);
    JASSERT(num_fds_found >= 0);
    break;
```

**Purpose**: Save pipe state and disable trampolines before memory checkpoint.

#### DMTCP_EVENT_RESUME (Lines 321-334)
```cpp
case DMTCP_EVENT_RESUME:
    // GPU restoration happens in RUNNING event
    break;
```

**Purpose**: Placeholder - actual GPU restoration deferred to RUNNING event.

#### DMTCP_EVENT_RESTART (Lines 336-339)
```cpp
case DMTCP_EVENT_RESTART:
    JASSERT(recreate_pipes(pipe_list, num_fds_found) != -1);
    break;
```

**Purpose**: Recreate pipes after memory restore but before thread resume.

### Plugin Registration (Lines 346-356)

```cpp
DmtcpPluginDescriptor_t cuda_plugin = {
  DMTCP_PLUGIN_API_VERSION,
  DMTCP_PACKAGE_VERSION,
  "CUDA-Plugin",
  "DMTCP",
  "xu.yao1@northeastern.edu",
  "CUDA Checkpoint API plugin",
  cuda_event_hook
};

DMTCP_DECL_PLUGIN(cuda_plugin);
```

**Purpose**: Register plugin with DMTCP system.

## Build Configuration: `Makefile`

### Key Variables (Lines 1-18)
```makefile
CC=gcc                    # C compiler
CX=g++                    # C++ compiler
NAME=${shell basename $$PWD}  # Directory name = "crac"
LIBNAME=libdmtcp_${NAME}  # Library name = "libdmtcp_crac"
LIBOBJS = ${NAME}.o       # Object files
DMTCP_ROOT=../../          # DMTCP installation path
CUDA_INCLUDE=-I/usr/local/cuda/include  # CUDA headers
```

### Build Flags (Lines 20-22)
```makefile
override CFLAGS += -g3 -O0 -fPIC -I${DMTCP_INCLUDE} ${CUDA_INCLUDE}
override CXXFLAGS += -g3 -O0 -fPIC ${DMTCP_INCLUDE} ${CUDA_INCLUDE}
LINK = ${CC}
```

**Purpose**: Debug build with position-independent code for shared library.

### Main Target (Lines 29-40)
```makefile
default: ${LIBNAME}.so tests

${LIBNAME}.so: ${LIBOBJS}
    ${LINK} -shared -fPIC -o $@ $^ -lcuda -ldl
```

**Purpose**: Build shared library linking with CUDA and dynamic loader.

## Test Programs

### `test/counter.cu` (Lines 1-20)

**Purpose**: Simple CUDA program with device counter that increments every second.

**Key Features**:
- Device variable `counter` incremented by kernel
- Host copies counter value and prints
- Infinite loop with 1-second sleep
- Minimal CUDA functionality for basic testing

**Kernel**:
```cuda
__device__ int counter = 0;
__global__ void increment() {
    counter++;
}
```

### `test/counter_mpi.cu` (Lines 1-23)

**Purpose**: MPI + CUDA version of counter test.

**Key Differences from counter.cu**:
- MPI initialization and finalization
- Same CUDA counter logic
- Tests MPI + CUDA checkpoint coordination

### `test/mpi_cuda.cu` (Lines 1-155)

**Purpose**: Complex MPI + CUDA vector addition test.

**Features**:
- Distributed vector addition across MPI processes
- Each process handles a chunk of the data
- CUDA kernel for vector addition
- MPI scatter/gather for data distribution
- Verification of results

**Algorithm**:
1. Initialize MPI
2. Distribute vector chunks to all processes
3. Each process:
   - Copy chunk to GPU
   - Perform vector addition on GPU
   - Copy results back to CPU
4. Gather results at rank 0
5. Verify correctness

## Control Flow Summary

### Initialization Flow
```
DMTCP loads plugin → DMTCP_EVENT_INIT → Setup trampolines
```

### Checkpoint Flow
```
DMTCP_EVENT_PRESUSPEND → checkpoint_gpu() → Lock/Checkpoint GPU
DMTCP_EVENT_PRECHECKPOINT → inspect_pipes() → Save pipe state
DMTCP memory checkpoint → Save process memory
```

### Restart Flow
```
DMTCP memory restore → Restore process memory
DMTCP_EVENT_RESTART → recreate_pipes() → Recreate pipes
DMTCP_EVENT_RUNNING → restore_gpu() → Restore GPU state
```

## Key Design Patterns

### Event-Driven Architecture
- All functionality triggered by DMTCP events
- Clear separation of concerns between different phases
- Deterministic ordering of operations

### Resource Tracking
- Pipes tracked by inode and FD number
- Device files tracked by memory mapping
- State preserved across checkpoint/restart

### Error Handling
- Extensive use of `JASSERT` for error checking
- Immediate abort on critical failures
- Clear error messages with context

### Memory Management
- RAII patterns with STL containers
- Explicit cleanup of resources
- Careful handling of memory reservations

## See Also

- [Architecture Overview](../architecture/overview.md) - High-level system design
- [Checkpoint Flow](../architecture/checkpoint-flow.md) - Detailed checkpoint process
- [Restart Flow](../architecture/restart-flow.md) - Detailed restart process
- [Data Structures](../architecture/data-structures.md) - Key data structures
- [Development Setup](development-setup.md) - Setting up development environment