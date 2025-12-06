# CUDA Checkpoint API Reference

This document provides a quick reference for CUDA checkpoint/restart APIs used in CRAC, introduced in CUDA 12.4.

## Overview

NVIDIA introduced native checkpoint/restart APIs in CUDA 12.4 that allow:
- Transparent checkpointing of GPU state
- Restoration of GPU memory and contexts
- Integration with user-space checkpoint tools like DMTCP

## Requirements

- **CUDA Version**: 12.4 or later
- **Driver Version**: 550.54.14 or later
- **GPU Architecture**: All supported architectures with CUDA 12.4+

## Core API Functions

### Process State Management

#### cuCheckpointProcessGetState
```cpp
CUresult cuCheckpointProcessGetState(int pid, CUprocessState *state);
```

**Purpose**: Get current checkpoint state of CUDA process.

**Parameters**:
- `pid`: Process ID (use `getpid()` for current process)
- `state`: Pointer to store process state

**Return Values**:
- `CUDA_SUCCESS`: Operation successful
- `CUDA_ERROR_INVALID_VALUE`: Invalid parameters
- `CUDA_ERROR_NOT_SUPPORTED`: Checkpoint not supported

**Usage in CRAC** (`crac.cpp:230, 258`):
```cpp
CUresult ret = cuCheckpointProcessGetState(pid, &cuda_state);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

#### Process State Enumeration
```cpp
typedef enum {
    CU_PROCESS_STATE_RUNNING = 0,
    CU_PROCESS_STATE_LOCKED = 1,
    CU_PROCESS_STATE_CHECKPOINTED = 2,
    CU_PROCESS_STATE_FAILED = 3
} CUprocessState;
```

**State Descriptions**:
- `RUNNING`: Normal operation, GPU is active
- `LOCKED`: GPU is locked, no new work accepted
- `CHECKPOINTED`: GPU state has been saved
- `FAILED`: Error state

### Checkpoint Operations

#### cuCheckpointProcessLock
```cpp
CUresult cuCheckpointProcessLock(int pid, CUcheckpointLockArgs *args);
```

**Purpose**: Lock GPU process to prepare for checkpointing.

**Parameters**:
- `pid`: Process ID
- `args`: Lock arguments structure (currently unused, pass `{0}`)

**Behavior**:
- Blocks new GPU work from being submitted
- Allows currently executing GPU work to complete
- Transitions process state from RUNNING to LOCKED

**Usage in CRAC** (`crac.cpp:206-207`):
```cpp
CUcheckpointLockArgs lock_args = {0};
CUresult ret = cuCheckpointProcessLock(pid, &lock_args);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

#### cuCheckpointProcessCheckpoint
```cpp
CUresult cuCheckpointProcessCheckpoint(int pid, CUcheckpointCheckpointArgs *args);
```

**Purpose**: Perform GPU checkpoint operation.

**Parameters**:
- `pid`: Process ID
- `args`: Checkpoint arguments structure (currently unused, pass `{0}`)

**Behavior**:
- Copies GPU memory to host memory
- Saves GPU contexts, streams, and other state
- Releases GPU resources
- Transitions process state from LOCKED to CHECKPOINTED

**Usage in CRAC** (`crac.cpp:226-227`):
```cpp
CUcheckpointCheckpointArgs checkpoint_args = {0};
CUresult ret = cuCheckpointProcessCheckpoint(pid, &checkpoint_args);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

### Restore Operations

#### cuCheckpointProcessRestore
```cpp
CUresult cuCheckpointProcessRestore(int pid, CUcheckpointRestoreArgs *args);
```

**Purpose**: Restore GPU state from previously saved checkpoint.

**Parameters**:
- `pid`: Process ID
- `args`: Restore arguments structure (currently unused, pass `{0}`)

**Behavior**:
- Restores GPU memory from host memory
- Recreates CUDA contexts and streams
- Reacquires GPU resources
- Transitions process state from CHECKPOINTED to LOCKED

**Usage in CRAC** (`crac.cpp:254-255`):
```cpp
CUresult ret = cuCheckpointProcessRestore(pid, &restore_args);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

#### cuCheckpointProcessUnlock
```cpp
CUresult cuCheckpointProcessUnlock(int pid, CUcheckpointUnlockArgs *args);
```

**Purpose**: Unlock GPU process after restore to resume normal operation.

**Parameters**:
- `pid`: Process ID
- `args`: Unlock arguments structure (currently unused, pass `{0}`)

**Behavior**:
- Allows new GPU work to be submitted
- Resumes normal GPU operations
- Transitions process state from LOCKED to RUNNING

**Usage in CRAC** (`crac.cpp:256-257`):
```cpp
CUcheckpointUnlockArgs unlock_args = {0};
CUresult ret = cuCheckpointProcessUnlock(pid, &unlock_args);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

## Argument Structures

### Lock Arguments
```cpp
typedef struct {
    // Currently unused, reserved for future use
} CUcheckpointLockArgs;
```

### Checkpoint Arguments
```cpp
typedef struct {
    // Currently unused, reserved for future use
} CUcheckpointCheckpointArgs;
```

### Restore Arguments
```cpp
typedef struct {
    // Currently unused, reserved for future use
} CUcheckpointRestoreArgs;
```

### Unlock Arguments
```cpp
typedef struct {
    // Currently unused, reserved for future use
} CUcheckpointUnlockArgs;
```

**Usage**: Initialize with zeros: `{0}`

## State Machine

```
    ┌─────────────┐
    │  RUNNING    │ ←─────┐
    └──────┬──────┘       │
           │ lock         │ unlock
           ▼              │
    ┌─────────────┐       │
    │   LOCKED    │       │
    └──────┬──────┘       │
           │ checkpoint   │ restore
           ▼              │
    ┌─────────────┐       │
    │ CHECKPOINTED│ ──────┘
    └─────────────┘
```

### State Transitions

#### RUNNING → LOCKED
- **Trigger**: `cuCheckpointProcessLock()`
- **Purpose**: Prepare for checkpointing
- **Effect**: Blocks new GPU work

#### LOCKED → CHECKPOINTED
- **Trigger**: `cuCheckpointProcessCheckpoint()`
- **Purpose**: Save GPU state
- **Effect**: GPU resources released

#### CHECKPOINTED → LOCKED
- **Trigger**: `cuCheckpointProcessRestore()`
- **Purpose**: Begin restoration
- **Effect**: GPU resources reacquired

#### LOCKED → RUNNING
- **Trigger**: `cuCheckpointProcessUnlock()`
- **Purpose**: Resume normal operation
- **Effect**: New GPU work allowed

## Error Handling

### Common Error Codes

#### CUDA_ERROR_INVALID_VALUE
- **Cause**: Invalid process ID or null pointer
- **Solution**: Verify parameters, use `getpid()` for current process

#### CUDA_ERROR_NOT_SUPPORTED
- **Cause**: CUDA version doesn't support checkpointing
- **Solution**: Upgrade to CUDA 12.4+

#### CUDA_ERROR_OPERATING_SYSTEM
- **Cause**: System-level error (permissions, resources)
- **Solution**: Check system permissions and available resources

#### CUDA_ERROR_UNKNOWN
- **Cause**: Unexpected error
- **Solution**: Check NVIDIA driver logs, restart system

### Error Handling Pattern
```cpp
CUresult ret = cuCheckpointProcessLock(pid, &lock_args);
if (ret != CUDA_SUCCESS) {
    const char *error_str;
    cuGetErrorString(ret, &error_str);
    fprintf(stderr, "CUDA checkpoint error: %s\n", error_str);
    return -1;
}
```

## Integration Considerations

### Thread Requirements
- **Checkpoint**: Requires user threads to be running
- **Restore**: Requires user threads to be running
- **State Query**: Can be called anytime

### Memory Requirements
- **Checkpoint**: Additional host memory for GPU state copy
- **Restore**: Sufficient host memory for saved state

### Timing Considerations
- **Lock Time**: Depends on outstanding GPU work
- **Checkpoint Time**: Proportional to GPU memory usage
- **Restore Time**: Proportional to GPU memory usage

## Limitations

### Unsupported Features
- **UVM (Unified Virtual Memory)**: Not checkpointable
- **IPC Memory**: Inter-process GPU memory not supported
- **Dynamic Parallelism**: Limited support
- **Multi-GPU**: Single GPU per process

### Resource Constraints
- **Host Memory**: Must accommodate GPU state copy
- **Disk Space**: Checkpoint files include GPU state
- **GPU Memory**: Must fit in available host memory

### Application Constraints
- **No CUDA API calls during checkpoint**: Must be in quiescent state
- **No kernel launches during restore**: GPU is locked during restore
- **Single checkpoint per process**: Cannot nest checkpoints

## Best Practices

### Checkpoint Preparation
1. **Synchronize CUDA operations**:
   ```cpp
   cudaDeviceSynchronize();
   cuCtxSynchronize(context);
   ```

2. **Complete outstanding work**:
   ```cpp
   cuStreamSynchronize(stream);
   ```

3. **Check GPU state**:
   ```cpp
   CUprocessState state;
   cuCheckpointProcessGetState(getpid(), &state);
   if (state != CU_PROCESS_STATE_RUNNING) {
       // Handle unexpected state
   }
   ```

### Error Recovery
1. **Retry failed operations**:
   ```cpp
   int retries = 3;
   CUresult ret;
   do {
       ret = cuCheckpointProcessLock(pid, &lock_args);
       if (ret == CUDA_SUCCESS) break;
       sleep(1);
   } while (retries-- > 0);
   ```

2. **Fallback strategies**:
   ```cpp
   if (ret != CUDA_SUCCESS) {
       // Log error and continue without checkpoint
       log_checkpoint_failure(ret);
       return 0; // Non-fatal
   }
   ```

### Performance Optimization
1. **Minimize GPU memory**: Reduce checkpoint size
2. **Batch operations**: Group multiple CUDA operations
3. **Overlap work**: Use asynchronous operations where possible

## Debugging

### State Monitoring
```cpp
void print_cuda_state() {
    CUprocessState state;
    CUresult ret = cuCheckpointProcessGetState(getpid(), &state);
    if (ret == CUDA_SUCCESS) {
        switch (state) {
            case CU_PROCESS_STATE_RUNNING:
                printf("CUDA State: RUNNING\n");
                break;
            case CU_PROCESS_STATE_LOCKED:
                printf("CUDA State: LOCKED\n");
                break;
            case CU_PROCESS_STATE_CHECKPOINTED:
                printf("CUDA State: CHECKPOINTED\n");
                break;
            case CU_PROCESS_STATE_FAILED:
                printf("CUDA State: FAILED\n");
                break;
        }
    }
}
```

### Error Logging
```cpp
void log_cuda_error(CUresult ret, const char *operation) {
    const char *error_str;
    cuGetErrorString(ret, &error_str);
    fprintf(stderr, "CUDA Error in %s: %s\n", operation, error_str);
}
```

## See Also

- [DMTCP Plugin API](dmtcp-plugin-api.md) - How CRAC uses these APIs
- [Checkpoint Flow](../architecture/checkpoint-flow.md) - API usage in context
- [Restart Flow](../architecture/restart-flow.md) - API usage in context
- [NVIDIA CUDA Documentation](https://docs.nvidia.com/cuda/cuda-checkpoint/) - Official NVIDIA docs