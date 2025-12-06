# Data Structures

This document describes the key data structures used in CRAC for managing state between checkpoint and restart operations.

## Global Variables

### CUDA State Management

#### `cuda_state`
**Location**: `crac.cpp:49`
```cpp
CUprocessState cuda_state = CU_PROCESS_STATE_RUNNING;
```

**Purpose**: Tracks the current state of the CUDA process for checkpoint/restart logic.

**Possible Values**:
- `CU_PROCESS_STATE_RUNNING`: Normal operation, GPU is active
- `CU_PROCESS_STATE_LOCKED`: GPU is locked, no new work accepted
- `CU_PROCESS_STATE_CHECKPOINTED`: GPU state has been saved to host memory
- `CU_PROCESS_STATE_FAILED`: Error state

**Usage**: 
- Checked in `DMTCP_EVENT_RUNNING` to determine if GPU restoration is needed
- Set in `checkpoint_gpu()` after successful checkpoint
- Verified in `restore_gpu()` after successful restore

#### `restore_args`
**Location**: `crac.cpp:50`
```cpp
CUcheckpointRestoreArgs restore_args = {0};
```

**Purpose**: Arguments structure for CUDA restore operation.

**Usage**: Passed to `cuCheckpointProcessRestore()` during restart.

#### `cuda_initialized`
**Location**: `crac.cpp:51`
```cpp
int cuda_initialized = 0;
```

**Purpose**: Flag indicating whether CUDA driver API has been initialized.

**Usage**: Prevents multiple calls to `cuInit(0)`.

### Device File Management

#### `nvidia_device_files`
**Location**: `crac.cpp:48`
```cpp
std::vector<ProcMapsArea> nvidia_device_files;
```

**Purpose**: Stores memory mapping information for NVIDIA device files (`/dev/nvidia*`).

**Structure**: Each `ProcMapsArea` contains:
- `addr`: Starting address of the mapping
- `size`: Size of the mapped region
- `name`: Path to the device file (e.g., "/dev/nvidia0")

**Usage**:
- Populated in `checkpoint_gpu()` by scanning `/proc/self/maps`
- Used to reserve memory regions after checkpoint
- Used to release reservations before GPU restore
- Cleared after successful restore

### Pipe Management

#### `pipe_list`
**Location**: `crac.cpp:52`
```cpp
pipe_info_t pipe_list[MAX_PIPE_FDS] = {0};
```

**Purpose**: Array storing information about pipe file descriptors at checkpoint time.

**Capacity**: `MAX_PIPE_FDS` (1024) pipes maximum.

#### `num_fds_found`
**Location**: `crac.cpp:53`
```cpp
int num_fds_found;
```

**Purpose**: Number of valid pipe entries stored in `pipe_list`.

## Data Structure Definitions

### Pipe Information Structure

#### `pipe_info_t`
**Location**: `crac.cpp:34-38`
```cpp
typedef struct {
    int fd;
    unsigned long pipe_inode;
    pipe_rw_t rw_type;
} pipe_info_t;
```

**Fields**:
- `fd`: File descriptor number
- `pipe_inode`: Inode number of the pipe (extracted from `/proc/self/fd` symlink)
- `rw_type`: Read/write access type (PIPE_READ or PIPE_WRITE)

**Usage**: 
- Populated by `inspect_pipes()` during checkpoint
- Used by `recreate_pipes()` during restart to recreate exact FD layout

### Pipe Read/Write Type Enumeration

#### `pipe_rw_t`
**Location**: `crac.cpp:28-31`
```cpp
typedef enum {
    PIPE_READ = O_RDONLY,
    PIPE_WRITE = O_WRONLY
} pipe_rw_t;
```

**Purpose**: Represents the access mode of a pipe file descriptor.

**Values**:
- `PIPE_READ`: Read-only end of pipe
- `PIPE_WRITE`: Write-only end of pipe

### Pipe Mapping Structure

#### `pipe_map_t`
**Location**: `crac.cpp:41-46`
```cpp
typedef struct {
    unsigned long old_inode;
    int new_read_fd;
    int new_write_fd;
    int created; // Flag to ensure we only create a new pipe pair once per inode
} pipe_map_t;
```

**Fields**:
- `old_inode`: Original pipe inode from checkpoint
- `new_read_fd`: File descriptor of new pipe read end
- `new_write_fd`: File descriptor of new pipe write end
- `created`: Flag indicating if this pipe pair has been created

**Usage**: 
- Used only in `recreate_pipes()` during restart
- Maps original pipe inodes to newly created pipe file descriptors
- Ensures each unique pipe inode creates only one new pipe pair

## Memory Layout and Persistence

### Checkpoint-Time Data
These structures are populated during checkpoint and saved as part of process memory:

1. **`pipe_list`**: Contains all pipe FD information
2. **`num_fds_found`**: Number of pipes to restore
3. **`nvidia_device_files`**: Device file mapping information
4. **`cuda_state`**: Current GPU state (CHECKPOINTED)
5. **`restore_args`**: CUDA restore arguments

### Restart-Time Data
These structures are restored from checkpoint memory and used during restart:

1. **`pipe_list`**: Used to recreate pipes with original FDs
2. **`num_fds_found`**: Number of pipes to recreate
3. **`nvidia_device_files`**: Used to release memory reservations
4. **`cuda_state`**: Determines if GPU restoration is needed
5. **`restore_args`**: Used for GPU restore operation

## Data Flow

### Checkpoint Flow
```
inspect_pipes() → pipe_list[] + num_fds_found
     ↓
DMTCP saves process memory (including pipe_list)
     ↓
checkpoint_gpu() → nvidia_device_files[]
     ↓
cuda_state = CU_PROCESS_STATE_CHECKPOINTED
```

### Restart Flow
```
DMTCP restores process memory (including pipe_list)
     ↓
recreate_pipes() ← pipe_list[] + num_fds_found
     ↓
restore_gpu() ← nvidia_device_files[]
     ↓
cuda_state = CU_PROCESS_STATE_RUNNING
```

## Memory Management

### Dynamic Allocation
- **`nvidia_device_files`**: Uses `std::vector` with dynamic allocation
- **Static arrays**: `pipe_list` and `pipe_map` use fixed-size arrays

### Memory Cleanup
- **`nvidia_device_files`**: Cleared with `clear()` after GPU restore
- **Static arrays**: Automatically cleaned up when process exits

### Thread Safety
- All data structures are accessed only during DMTCP event callbacks
- DMTCP ensures single-threaded execution during these callbacks
- No additional synchronization needed

## Size Considerations

### Pipe Information
- **Maximum pipes**: 1024 (`MAX_PIPE_FDS`)
- **Memory per pipe**: ~24 bytes (`pipe_info_t`)
- **Total pipe memory**: ~24KB maximum

### Device File Information
- **Variable size**: Depends on number of NVIDIA device mappings
- **Typical size**: 1-10 mappings, ~100-1000 bytes total
- **Memory per mapping**: ~100 bytes (`ProcMapsArea`)

### Overall Memory Footprint
- **Static data**: ~25KB (mostly pipe arrays)
- **Dynamic data**: <1KB (typical device file mappings)
- **Total impact**: Minimal compared to application memory

## Error Handling

### Validation
- **Array bounds**: Checked with `JASSERT(count < max_pipe_fds)`
- **FD validity**: Checked with `fcntl(fd, F_GETFD) == -1`
- **Pipe type**: Verified as `PIPE_READ` or `PIPE_WRITE`

### Failure Modes
- **Too many pipes**: Assertion failure if `MAX_PIPE_FDS` exceeded
- **Invalid FDs**: Handled gracefully in pipe inspection
- **Memory allocation**: Vector allocation failures cause abort

## Debug Information

### State Inspection
**Function**: `print_cuda_process_state()` at `crac.cpp:173-190`

**Purpose**: Prints current CUDA process state for debugging.

### Pipe Debugging
The `inspect_pipes()` function includes debug output through `JASSERT` macros that can help identify pipe-related issues during development.

## See Also

- [Checkpoint Flow](checkpoint-flow.md) - How these structures are used during checkpoint
- [Restart Flow](restart-flow.md) - How these structures are used during restart
- [Code Walkthrough](../contributor-guide/code-walkthrough.md) - Detailed code analysis
- [DMTCP Plugin API](../reference/dmtcp-plugin-api.md) - DMTCP event handling context