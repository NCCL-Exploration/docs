# Checkpoint Flow Documentation

This document provides a detailed walkthrough of the DMTCP checkpoint process, from initiation through completion. Understanding this flow is essential for contributing to DMTCP's core checkpointing logic.

## Overview

The DMTCP checkpoint process involves coordination between the coordinator, multiple worker processes, and their threads. The process follows a seven-stage algorithm to ensure consistent checkpointing across distributed applications.

## Participants

```
┌─────────────────┐    TCP/7779    ┌─────────────────┐
│   Coordinator   │ ◄─────────────► │  Worker Process │
│                 │                 │                 │
│ - State machine │                 │ - Main thread   │
│ - Barrier sync  │                 │ - Checkpoint    │
│ - Message relay │                 │   thread        │
└─────────────────┘                 │ - User threads  │
                                    └─────────────────┘
```

## Detailed Checkpoint Flow

### Phase 1: Checkpoint Initiation

#### 1.1 User/Timer Trigger
```cpp
// User initiates via dmtcp_command or coordinator UI
// Location: src/dmtcp_coordinator.cpp:handleUserCommand()
void DmtcpCoordinator::handleUserCommand(char cmd, int *clientSock) {
  switch (cmd) {
    case 'c':  // checkpoint command
      startCheckpoint();
      break;
  }
}
```

#### 1.2 Coordinator Broadcast
```cpp
// Coordinator broadcasts checkpoint request to all workers
// Location: src/dmtcp_coordinator.cpp:startCheckpoint()
void DmtcpCoordinator::startCheckpoint() {
  // Update minimum state to initiate checkpoint
  broadcastMessage(DMT_KILL_PEER, "Checkpoint starting");
  
  // Send checkpoint command to all connected workers
  for (each worker) {
    sendMessage(worker, DMT_DO_CHECKPOINT);
  }
}
```

### Phase 2: Worker Response and Thread Suspension

#### 2.1 Checkpoint Thread Receives Signal
```cpp
// Location: src/threadlist.cpp
// The checkpoint thread is always waiting for coordinator commands
static void *checkpointThreadFunc(void *arg) {
  while (true) {
    // Wait for coordinator message
    msg = coordinatorAPI::recvMsgFromCoordinator();
    
    if (msg.type == DMT_DO_CHECKPOINT) {
      initiateCheckpoint();
    }
  }
}
```

#### 2.2 Suspend User Threads
```cpp
// Location: src/threadlist.cpp:suspendThreads()
static void suspendThreads() {
  // Send SIGUSR2 to all user threads
  for (each user thread) {
    pthread_kill(thread_id, SIGUSR2);
  }
  
  // Wait for all threads to reach signal handler
  waitForThreadsToSuspend();
}
```

#### 2.3 Signal Handler Execution
```cpp
// Location: src/signalwrappers.cpp
static void checkpointSignalHandler(int signum, siginfo_t *info, void *context) {
  // Save thread context
  saveThreadContext(context);
  
  // Block at barrier
  threadSyncBarrier();
}
```

### Phase 3: File Descriptor Leader Election

#### 3.1 Identify Shared File Descriptors
```cpp
// Location: src/filewrappers.cpp
static void electFDLeaders() {
  for (each open file descriptor) {
    // Use fcntl to determine ownership
    if (fcntl(fd, F_SETLKW, &write_lock) == 0) {
      // This process is the leader for this fd
      markAsFDLeader(fd);
    }
  }
}
```

#### 3.2 Leader Registration
```cpp
// Leaders coordinate to handle shared resources
// Location: src/connection.cpp
void ConnectionManager::electLeaders() {
  for (each shared connection) {
    Connection *leader = determineLeader(connection);
    leader->setAsLeader();
  }
}
```

### Phase 4: Drain Kernel Buffers

#### 4.1 Socket Buffer Draining
```cpp
// Location: src/socketwrappers.cpp
static void drainSocketBuffers() {
  for (each socket connection) {
    if (isSocketLeader(sockfd)) {
      // Send magic cookie to drain buffers
      sendDrainToken(sockfd);
      
      // Read all pending data
      while (hasPendingData(sockfd)) {
        readAndBufferData(sockfd);
      }
    }
  }
}
```

#### 4.2 Pipe and FIFO Draining
```cpp
// Location: src/filewrappers.cpp
static void drainPipes() {
  for (each pipe/fifo) {
    if (isLeader(pipe_fd)) {
      drainPipeData(pipe_fd);
    }
  }
}
```

### Phase 5: Memory Checkpointing

#### 5.1 MTCP Checkpoint Initiation
```c
// Location: src/mtcp/mtcp.c
int mtcp_checkpoint(void) {
  // Read memory layout from /proc/self/maps
  readMemoryAreas();
  
  // Save thread contexts
  saveThreadContexts();
  
  // Write checkpoint image
  writeCheckpointImage();
  
  return 0;
}
```

#### 5.2 Memory Area Processing
```c
// Location: src/mtcp/mtcp.c
static void readMemoryAreas() {
  FILE *maps = fopen("/proc/self/maps", "r");
  
  while (fgets(line, sizeof(line), maps)) {
    Area area;
    parseMapsLine(line, &area);
    
    // Save memory area
    saveMemoryArea(&area);
  }
}
```

#### 5.3 Checkpoint Image Writing
```c
// Location: src/mtcp/mtcp_write.c
static void writeCheckpointImage() {
  // Write header
  writeCheckpointHeader();
  
  // Write memory sections
  for (each memory area) {
    writeMemorySection(area);
  }
  
  // Write thread contexts
  writeThreadContexts();
  
  // Write file descriptor information
  writeFDInfo();
}
```

### Phase 6: Buffer Refilling

#### 6.1 Restore Socket State
```cpp
// Location: src/socketwrappers.cpp
static void refillSocketBuffers() {
  for (each socket connection) {
    if (isSocketLeader(sockfd)) {
      // Resend drained data
      resendBufferedData(sockfd);
      
      // Restore socket options
      restoreSocketOptions(sockfd);
    }
  }
}
```

#### 6.2 Restore Pipe State
```cpp
// Location: src/filewrappers.cpp
static void refillPipes() {
  for (each pipe/fifo) {
    if (isLeader(pipe_fd)) {
      refillPipeData(pipe_fd);
    }
  }
}
```

### Phase 7: Thread Resumption

#### 7.1 Resume User Threads
```cpp
// Location: src/threadlist.cpp
static void resumeThreads() {
  // Release threads from signal handler barrier
  releaseThreadsFromBarrier();
  
  // Wait for threads to resume execution
  waitForThreadsToResume();
}
```

#### 7.2 Notify Coordinator
```cpp
// Location: src/coordinatorapi.cpp
static void notifyCoordinatorCheckpointComplete() {
  DmtcpMessage msg;
  msg.type = DMT_CHECKPOINT_COMPLETE;
  
  coordinatorAPI::sendMsgToCoordinator(&msg);
}
```

## State Transitions

### Coordinator State Machine
```
RUNNING
    ↓ (broadcast checkpoint)
SUSPENDED
    ↓ (all workers suspended)
FD_LEADER_ELECTION
    ↓ (leaders elected)
DRAINED
    ↓ (buffers drained)
CHECKPOINTING
    ↓ (checkpoint written)
CHECKPOINTED
    ↓ (buffers refilled)
REFILLED
    ↓ (threads resumed)
RUNNING
```

### Worker State Machine
```
RUNNING
    ↓ (receive checkpoint command)
SUSPENDING
    ↓ (threads suspended)
SUSPENDED
    ↓ (FD leader election)
FD_LEADER_ELECTION
    ↓ (draining buffers)
DRAINING
    ↓ (buffers drained)
CHECKPOINTING
    ↓ (checkpoint written)
CHECKPOINTED
    ↓ (refilling buffers)
REFILLING
    ↓ (buffers refilled)
RESUMING
    ↓ (threads resumed)
RUNNING
```

## Key Data Structures

### Checkpoint Thread Context
```cpp
// Location: src/threadlist.h
struct Thread {
  pthread_t tid;
  void *stack;
  ucontext_t context;
  int state;  // ST_RUNNING, ST_SUSPENDED, etc.
};
```

### Connection Information
```cpp
// Location: src/connection.h
struct Connection {
  int id;
  int type;  // SOCK_STREAM, SOCK_DGRAM, etc.
  int fd;
  bool isLeader;
  vector<char> drainedData;
};
```

### Checkpoint Image Header
```c
// Location: src/mtcp/mtcp_header.h
struct mtcpHeader {
  uint32_t magic;
  uint32_t version;
  uint32_t numAreas;
  uint32_t numThreads;
  uint64_t brk;
  uint64_t saved_sp;
};
```

## Error Handling

### Checkpoint Failure Scenarios
1. **Thread suspension failure**: Some threads don't respond to SIGUSR2
2. **FD leader election conflict**: Multiple processes claim leadership
3. **Buffer draining timeout**: Network operations hang
4. **Disk space exhaustion**: Cannot write checkpoint image
5. **Coordinator communication loss**: Network partition

### Recovery Mechanisms
```cpp
// Location: src/dmtcp_coordinator.cpp
void DmtcpCoordinator::handleCheckpointFailure() {
  // Broadcast abort message
  broadcastMessage(DMT_CHECKPOINT_FAILED);
  
  // Reset state machine
  resetComputationState();
  
  // Resume all workers
  broadcastMessage(DMT_RESUME);
}
```

## Performance Considerations

### Bottlenecks
1. **Network I/O**: Buffer draining/refilling for distributed applications
2. **Disk I/O**: Checkpoint image writing speed
3. **Memory copying**: Large memory sections
4. **Thread synchronization**: Barrier wait times

### Optimizations
1. **Compression**: Optional gzip compression of checkpoint images
2. **Incremental checkpointing**: Only save changed memory pages
3. **Parallel checkpointing**: Multiple processes checkpoint simultaneously
4. **Memory mapping**: Use mmap for efficient I/O

## Debugging Checkpoint Issues

### Common Problems
1. **Deadlocks**: Threads stuck in system calls during suspension
2. **Resource leaks**: File descriptors not properly tracked
3. **Race conditions**: Timing issues in multi-threaded applications
4. **Memory corruption**: Invalid pointer restoration

### Debug Tools
```bash
# Enable debug logging
DMTCP_DEBUG=1 dmtcp_launch ./application

# Check debug logs
tail -f $DMTCP_TMPDIR/dmtcp-$USER@$HOST/jassertlog.*

# Use GDB with DMTCP
gdb ./application `pgrep application`
```

### Debug Information Sources
1. **Coordinator logs**: State transitions and messages
2. **Worker logs**: Thread states and wrapper calls
3. **MTCP logs**: Memory area information and checkpoint details
4. **Plugin logs**: Plugin-specific events and data

This checkpoint flow forms the core of DMTCP's functionality. Understanding each phase and the interactions between components is crucial for contributing to the checkpointing logic and diagnosing issues.