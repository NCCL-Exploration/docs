# Checkpoint Flow

This document provides a detailed step-by-step walkthrough of what happens when a checkpoint is triggered in CRAC.

## Overview

The checkpoint process involves coordination between DMTCP, the CRAC plugin, and the CUDA driver. The flow ensures that GPU state is properly saved before the process memory is checkpointed.

## Sequence Diagram

```
Coordinator    DMTCP Core    CRAC Plugin    CUDA Driver    Application
     │              │             │              │              │
     │ checkpoint   │             │              │              │
     ├─────────────►│             │              │              │
     │              │ PRESUSPEND  │              │              │
     │              ├────────────►│              │              │
     │              │             │ checkpoint_gpu│            │
     │              │             ├─────────────►│              │
     │              │             │ cuCheckpointProcessLock     │
     │              │             ├─────────────►│              │
     │              │             │ ◄─────────────│              │
     │              │             │ cuCheckpointProcessCheckpoint│
     │              │             ├─────────────►│              │
     │              │             │ ◄─────────────│              │
     │              │             │              │ GPU work     │
     │              │             │              │ completes    │
     │              │ ◄────────────│              │              │
     │              │ PRECHECKPOINT│             │              │
     │              ├────────────►│              │              │
     │              │             │ inspect_pipes │              │
     │              │             │              │              │
     │              │ ◄────────────│              │              │
     │              │ checkpoint   │              │              │
     │              │ memory       │              │              │
     │              ├────────────►│              │              │
     │              │             │              │              │
     │ checkpointed │             │              │              │
     │ ◄─────────────│             │              │              │
```

## Detailed Step-by-Step Flow

### Step 1: Checkpoint Trigger

**Trigger**: User or automatic checkpoint request via DMTCP coordinator

**Source**: External command (`dmtcp_command --checkpoint`)

**Action**: DMTCP coordinator sends checkpoint signal to all managed processes

### Step 2: DMTCP_EVENT_PRESUSPEND

**Location**: `crac.cpp:309-313`

**Code**:
```cpp
case DMTCP_EVENT_PRESUSPEND:
  // We need to suspend and checkpoint the GPU before suspending user
  // threads on the CPU.
  checkpoint_gpu();
  break;
```

**Purpose**: This is the critical event where CRAC must checkpoint the GPU before DMTCP suspends user threads.

**Why PRESUSPEND?**: CUDA's checkpoint APIs require user threads to be running for communication with the GPU.

### Step 3: GPU Checkpoint (`checkpoint_gpu` function)

**Location**: `crac.cpp:195-245`

#### 3.1 Initialize CUDA Driver API
**Code**: `crac.cpp:197-200`
```cpp
if (!cuda_initialized) {
  cuInit(0);
  cuda_initialized = 1;
}
```

**Purpose**: Ensure CUDA driver is initialized before calling checkpoint APIs.

#### 3.2 Lock GPU Process
**Code**: `crac.cpp:206-207`
```cpp
CUcheckpointLockArgs lock_args = {0};
CUresult ret = cuCheckpointProcessLock(pid, &lock_args);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

**Purpose**: 
- Blocks new GPU work from being submitted
- Allows currently executing GPU work to complete
- Transitions GPU process state from RUNNING to LOCKED

**CUDA State Change**: `RUNNING → LOCKED`

#### 3.3 Scan NVIDIA Device Files
**Code**: `crac.cpp:216-222`
```cpp
dmtcp::ProcSelfMaps proc_maps;
Area area;
while (proc_maps.getNextArea(&area)) {
  if (strstr(area.name, "/dev/nvidia")) {
    nvidia_device_files.push_back(area);
  }
}
```

**Purpose**: Identify memory mappings for NVIDIA device files that will be released during checkpoint.

#### 3.4 Perform GPU Checkpoint
**Code**: `crac.cpp:223-232`
```cpp
printf("start checkpoining GPU\n");
fflush(stdout);

ret = cuCheckpointProcessCheckpoint(pid, &checkpoint_args);
printf("finished checkpoining GPU\n");
fflush(stdout);
JASSERT(ret == CUDA_SUCCESS)(ret);
ret = cuCheckpointProcessGetState(pid, &cuda_state);
JASSERT(ret == CUDA_SUCCESS)(ret);
JASSERT(cuda_state == CU_PROCESS_STATE_CHECKPOINTED);
```

**Purpose**:
- Copies GPU memory to host memory
- Saves GPU state (contexts, streams, etc.)
- Releases GPU resources
- Transitions GPU process state to CHECKPOINTED

**CUDA State Change**: `LOCKED → CHECKPOINTED`

#### 3.5 Reserve Device File Regions
**Code**: `crac.cpp:240-244`
```cpp
for (Area device_area : nvidia_device_files) {
  void *ret = mmap(device_area.addr, device_area.size, PROT_NONE,
                   MAP_FIXED | MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
  JASSERT(ret == device_area.addr)(ret)(device_area.addr)(device_area.name);
}
```

**Purpose**: Reserve memory regions where NVIDIA device files were mapped to prevent other code from using these addresses during the checkpoint process.

### Step 4: DMTCP_EVENT_PRECHECKPOINT

**Location**: `crac.cpp:315-319`

**Code**:
```cpp
case DMTCP_EVENT_PRECHECKPOINT:
  UNINSTALL_TRAMPOLINE(eventfd_trampoline_info);
  num_fds_found = inspect_pipes(pipe_list, MAX_PIPE_FDS);
  JASSERT(num_fds_found >= 0);
  break;
```

#### 4.1 Uninstall Trampolines
**Purpose**: Remove any function wrappers before DMTCP saves process memory.

#### 4.2 Inspect Pipes
**Location**: `crac.cpp:56-92` (`inspect_pipes` function)

**Purpose**: Scan `/proc/self/fd` for pipe file descriptors to preserve across checkpoint/restart.

**Process**:
1. Open `/proc/self/fd` directory
2. Read each file descriptor entry
3. Check if the symlink target starts with "pipe:["
4. Extract pipe inode number
5. Store pipe information (fd, inode, read/write type)

### Step 5: DMTCP Memory Checkpoint

**Handled by**: DMTCP core (not CRAC)

**Process**:
- Saves entire process memory to checkpoint files
- Includes CPU memory, registers, and process state
- GPU state has already been saved by CRAC in Step 3

### Step 6: Checkpoint Complete

**Result**: 
- Process memory saved to checkpoint files
- GPU state saved in host memory
- All resources ready for restart
- Process can be killed and restarted later

## Key Timing Considerations

### Why PRESUSPEND?
- CUDA checkpoint APIs require active user threads
- DMTCP suspends threads after PRESUSPEND
- GPU checkpoint must complete before thread suspension

### Why PRECHECKPOINT for Pipes?
- Pipes are part of process state saved by DMTCP
- Pipe inspection must happen after GPU checkpoint
- Trampolines must be removed before memory save

### GPU Work Completion
- `cuCheckpointProcessLock()` ensures no new work is submitted
- Existing GPU work completes during lock phase
- `cuCheckpointProcessCheckpoint()` waits for completion

## Error Handling

### CUDA API Failures
**Code**: Throughout `checkpoint_gpu` function
```cpp
JASSERT(ret == CUDA_SUCCESS)(ret);
```

**Action**: Any CUDA API failure causes an assertion failure and aborts the checkpoint.

### Memory Reservation Failures
**Code**: `crac.cpp:242-243`
```cpp
JASSERT(ret == device_area.addr)(ret)(device_area.addr)(device_area.name);
```

**Action**: Failed memory reservation causes checkpoint abort.

### Pipe Inspection Failures
**Code**: `crac.cpp:318`
```cpp
JASSERT(num_fds_found >= 0);
```

**Action**: Pipe inspection failures are caught and reported.

## Performance Considerations

### GPU Memory Size
- Larger GPU memory allocations result in longer checkpoint times
- GPU memory is copied to host memory during checkpoint

### Network Storage
- Checkpoint files on network storage can increase checkpoint time
- Consider local SSD storage for better performance

### Application State
- Complex CUDA applications with many contexts/streams may take longer
- GPU work completion time affects overall checkpoint duration

## Debug Information

### Debug Output
**Code**: `crac.cpp:223-228`
```cpp
printf("start checkpoining GPU\n");
fflush(stdout);
// ... checkpoint operation ...
printf("finished checkpoining GPU\n");
fflush(stdout);
```

### State Debugging
**Code**: `crac.cpp:173-190` (`print_cuda_process_state`)
```cpp
void print_cuda_process_state() {
  int pid = getpid();
  switch (cuda_state) {
    case CU_PROCESS_STATE_RUNNING:
      printf("CUDA process %d is running\n", pid);
      break;
    // ... other states ...
  }
}
```

## See Also

- [Restart Flow](restart-flow.md) - How the checkpointed state is restored
- [CUDA Checkpoint API](../reference/cuda-checkpoint-api.md) - Details on CUDA APIs used
- [DMTCP Plugin API](../reference/dmtcp-plugin-api.md) - Details on DMTCP events
- [Architecture Overview](overview.md) - High-level architecture context