# Restart Flow

This document provides a detailed step-by-step walkthrough of what happens when a checkpointed CUDA application is restarted with CRAC.

## Overview

The restart process reverses the checkpoint flow, restoring process memory first, then reconstructing GPU state and IPC resources. The timing is critical to ensure proper restoration order.

## Sequence Diagram

```
User           DMTCP Core    CRAC Plugin    CUDA Driver    Application
  │                │             │              │              │
  │ dmtcp_restart  │             │              │              │
  ├───────────────►│             │              │              │
  │                │ RESTART     │              │              │
  │                ├────────────►│              │              │
  │                │             │ recreate_pipes│              │
  │                │             │              │              │
  │                │ ◄────────────│              │              │
  │                │ restore     │              │              │
  │                │ memory      │              │              │
  │                ├────────────►│              │              │
  │                │             │              │              │
  │                │ RUNNING     │              │              │
  │                ├────────────►│              │              │
  │                │             │ restore_gpu   │              │
  │                │             ├─────────────►│              │
  │                │             │ cuCheckpointProcessRestore │
  │                │             ├─────────────►│              │
  │                │             │ ◄─────────────│              │
  │                │             │ cuCheckpointProcessUnlock │
  │                │             ├─────────────►│              │
  │                │             │ ◄─────────────│              │
  │                │ ◄────────────│              │              │
  │ running         │             │              │              │
  │ ◄───────────────│             │              │              │
```

## Detailed Step-by-Step Flow

### Step 1: Restart Initiation

**Trigger**: User runs `dmtcp_restart` with checkpoint files

**Command**: `dmtcp_restart ckpt_*.dmtcp`

**Action**: DMTCP reads checkpoint files and prepares to restore process memory

### Step 2: DMTCP_EVENT_RESTART

**Location**: `crac.cpp:336-339`

**Code**:
```cpp
case DMTCP_EVENT_RESTART:
  // See comments in DMTCP_EVENT_RESUME above.
  JASSERT(recreate_pipes(pipe_list, num_fds_found) != -1);
  break;
```

**Purpose**: This event fires after DMTCP has restored process memory but before user threads start running.

### Step 3: Pipe Recreation (`recreate_pipes` function)

**Location**: `crac.cpp:94-170`

#### 3.1 Load Real pipe() Function
**Code**: `crac.cpp:95-99`
```cpp
static void* handle = dlopen("libc.so.6", RTLD_LAZY);
JASSERT(handle != NULL)("Failed to open libc");
int (*real_pipe)(int*);
*(void**)(&real_pipe) = dlsym(handle, "pipe");
JASSERT(dlerror() == NULL)("Failed to find symbol for pipe");
```

**Purpose**: Get the real `pipe()` function from libc to create new pipes.

#### 3.2 Reserve Original FD Locations
**Code**: `crac.cpp:105-111`
```cpp
int fd_unique = open("/dev/zero", O_RDONLY);
for (int i = 0; i < num_fds; i++) {
  int fd = pipe_fd_array[i].fd;
  JASSERT(fd == fd_unique || fcntl(fd, F_GETFD) == -1);
  dup2(fd_unique, fd); // Occupy fd; Close it later.
}
```

**Purpose**: Occupy the original file descriptor numbers to prevent them from being reused during pipe creation.

#### 3.3 Create New Pipes
**Code**: `crac.cpp:114-133`
```cpp
for (int i = 0; i < num_fds; i++) {
  unsigned long old_inode = pipe_fd_array[i].pipe_inode;
  int found = 0;
  for (int j = 0; j < map_count; j++) {
    if (pipe_map[j].old_inode == old_inode) {
        found = 1;
        break;
    }
  }
  if (!found && pipe_map[map_count].created == 0) {
    int new_fds[2];
    JASSERT((*real_pipe)(new_fds) != -1);
    pipe_map[map_count].old_inode = old_inode;
    pipe_map[map_count].new_read_fd = new_fds[0];
    pipe_map[map_count].new_write_fd = new_fds[1];
    pipe_map[map_count].created = 1;
    map_count++;
  }
}
```

**Purpose**: Create new pipes for each unique pipe inode that existed at checkpoint time.

#### 3.4 Close Reserved FDs
**Code**: `crac.cpp:135-139`
```cpp
for (int i = 0; i < num_fds; i++) {
  close(pipe_fd_array[i].fd);
}
close(fd_unique);
```

**Purpose**: Free up the original file descriptor numbers for duplication.

#### 3.5 Duplicate to Original FDs
**Code**: `crac.cpp:142-160`
```cpp
for (int i = 0; i < num_fds; i++) {
  unsigned long old_inode = pipe_fd_array[i].pipe_inode;
  int old_fd = pipe_fd_array[i].fd;
  pipe_rw_t rw_type = pipe_fd_array[i].rw_type;
  int new_fd_to_dup = -1;
  // Find the newly created pipe FDs using the old pipe inode
  for (int j = 0; j < map_count; j++) {
    if (pipe_map[j].old_inode == old_inode) {
      if (rw_type == PIPE_READ) {
        new_fd_to_dup = pipe_map[j].new_read_fd;
      } else if (rw_type == PIPE_WRITE) {
        new_fd_to_dup = pipe_map[j].new_write_fd;
      }
      break;
    }
  }
  assert(new_fd_to_dup != -1);
  assert(dup2(new_fd_to_dup, old_fd) != -1);
}
```

**Purpose**: Duplicate the new pipe ends to the original file descriptor numbers, preserving the exact FD layout.

#### 3.6 Cleanup Temporary FDs
**Code**: `crac.cpp:164-167`
```cpp
for (int j = 0; j < map_count; j++) {
  close(pipe_map[j].new_read_fd);
  close(pipe_map[j].new_write_fd);
}
```

**Purpose**: Close the temporary file descriptors used for pipe creation.

### Step 4: DMTCP Memory Restore

**Handled by**: DMTCP core (not CRAC)

**Process**:
- Restores process memory from checkpoint files
- Restores CPU registers and process state
- Restores CRAC plugin variables including `pipe_list` and `num_fds_found`
- GPU state is still CHECKPOINTED at this point

### Step 5: DMTCP_EVENT_RUNNING

**Location**: `crac.cpp:288-307`

**Code**:
```cpp
case DMTCP_EVENT_RUNNING:
  DMTCP_PLUGIN_DISABLE_CKPT();    
  if (cuda_state == CU_PROCESS_STATE_RUNNING) {
    // In this case, the CUDA process is launched for the first time.
  } else if (cuda_state == CU_PROCESS_STATE_CHECKPOINTED) {
    INSTALL_TRAMPOLINE(eventfd_trampoline_info);
    // In this case, the CUDA process has been checkpointed, it can be
    // either restart or resume. We need to restore the GPU states and
    // memories.
    restore_gpu();
    nvidia_device_files.clear();
  }
  DMTCP_PLUGIN_ENABLE_CKPT();    
  break;
```

**Purpose**: This event fires when user threads start running after restart.

### Step 6: GPU Restoration (`restore_gpu` function)

**Location**: `crac.cpp:247-262`

#### 6.1 Release Device File Reservations
**Code**: `crac.cpp:250-252`
```cpp
for (Area device_area : nvidia_device_files) {
  munmap(device_area.addr, device_area.size);
}
```

**Purpose**: Release the memory regions that were reserved for NVIDIA device files during checkpoint.

#### 6.2 Restore GPU State
**Code**: `crac.cpp:254-255`
```cpp
CUresult ret = cuCheckpointProcessRestore(pid, &restore_args);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

**Purpose**:
- Restore GPU memory from host memory
- Recreate CUDA contexts and streams
- Restore GPU device state
- Transition GPU process state from CHECKPOINTED to LOCKED

**CUDA State Change**: `CHECKPOINTED → LOCKED`

#### 6.3 Unlock GPU Process
**Code**: `crac.cpp:256-257`
```cpp
ret = cuCheckpointProcessUnlock(pid, &unlock_args);
JASSERT(ret == CUDA_SUCCESS)(ret);
```

**Purpose**:
- Allow new GPU work to be submitted
- Resume normal GPU operations
- Transition GPU process state to RUNNING

**CUDA State Change**: `LOCKED → RUNNING`

#### 6.4 Verify State and Cleanup
**Code**: `crac.cpp:258-261`
```cpp
ret = cuCheckpointProcessGetState(pid, &cuda_state);
JASSERT(ret == CUDA_SUCCESS)(ret);
JASSERT(cuda_state == CU_PROCESS_STATE_RUNNING)(cuda_state);
nvidia_device_files.clear();
```

**Purpose**: 
- Verify GPU is in RUNNING state
- Clear the device file tracking vector

### Step 7: Application Resumes

**Result**: 
- Process memory fully restored
- GPU state fully restored
- Pipes recreated with original FDs
- Application continues execution from checkpoint point

## Key Timing Considerations

### Why RUNNING Event for GPU Restore?
- GPU restore requires active user threads
- RUNNING event fires after threads are resumed
- Ensures CUDA driver can communicate with GPU

### Why RESTART Event for Pipes?
- Pipes must be recreated before user threads run
- RESTART event fires after memory restore but before thread resume
- Ensures FD layout is correct when application starts

### State Dependencies
- GPU restore depends on device file reservations being released
- Pipe recreation depends on saved pipe information from checkpoint
- Application resume depends on both pipes and GPU being ready

## Error Handling

### Pipe Recreation Failures
**Code**: `crac.cpp:126, 158-159`
```cpp
JASSERT((*real_pipe)(new_fds) != -1);
assert(new_fd_to_dup != -1);
assert(dup2(new_fd_to_dup, old_fd) != -1);
```

**Action**: Any pipe recreation failure causes restart to abort.

### GPU Restore Failures
**Code**: `crac.cpp:255, 257, 260`
```cpp
JASSERT(ret == CUDA_SUCCESS)(ret);
JASSERT(cuda_state == CU_PROCESS_STATE_RUNNING)(cuda_state);
```

**Action**: GPU restore failures cause restart abort.

### Memory Reservation Issues
**Code**: `crac.cpp:251`
```cpp
munmap(device_area.addr, device_area.size);
```

**Action**: Memory reservation release failures are logged but don't abort restart.

## Performance Considerations

### GPU Memory Size
- Larger GPU allocations result in longer restore times
- GPU memory is copied from host to device during restore

### Pipe Complexity
- Applications with many pipes may take longer to recreate
- Pipe recreation involves multiple system calls

### Device File Count
- More NVIDIA device mappings mean more `munmap` operations
- Typically minimal impact on overall restore time

## Debug Information

### State Verification
**Code**: `crac.cpp:258-260`
```cpp
ret = cuCheckpointProcessGetState(pid, &cuda_state);
JASSERT(ret == CUDA_SUCCESS)(ret);
JASSERT(cuda_state == CU_PROCESS_STATE_RUNNING)(cuda_state);
```

### Debug Function
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

## Special Cases

### First Launch vs Restart
**Code**: `crac.cpp:296-297`
```cpp
if (cuda_state == CU_PROCESS_STATE_RUNNING) {
  // In this case, the CUDA process is launched for the first time.
}
```

**Logic**: If CUDA state is already RUNNING, this is a fresh launch, not a restart.

### Resume vs Restart
**Code**: `crac.cpp:298-305`
```cpp
else if (cuda_state == CU_PROCESS_STATE_CHECKPOINTED) {
  // In this case, the CUDA process has been checkpointed, it can be
  // either restart or resume. We need to restore the GPU states and
  // memories.
  restore_gpu();
}
```

**Logic**: Both resume (after checkpoint) and restart (from disk) require GPU restoration.

## See Also

- [Checkpoint Flow](checkpoint-flow.md) - How the checkpointed state was created
- [CUDA Checkpoint API](../reference/cuda-checkpoint-api.md) - Details on CUDA APIs used
- [DMTCP Plugin API](../reference/dmtcp-plugin-api.md) - Details on DMTCP events
- [Architecture Overview](overview.md) - High-level architecture context