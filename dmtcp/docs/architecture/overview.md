# DMTCP Architecture Overview

DMTCP (Distributed MultiThreaded CheckPointing) is a transparent checkpointing system that enables saving and restoring the state of distributed applications without modifying the application code or requiring kernel changes.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DMTCP System                             │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  TCP/7779  ┌─────────────┐  TCP/7779          │
│  │   Worker 1  │ ◄─────────► │ Coordinator │ ◄─────────► Worker N│
│  │             │             │             │             │
│  │ libdmtcp.so │             │             │ libdmtcp.so │
│  └─────────────┘             └─────────────┘ └─────────────┘
│         │                           │                           │
│         ▼                           ▼                           ▼
│  ┌─────────────┐             ┌─────────────┐             ┌─────────────┐
│  │ Application │             │   DMTCP     │             │ Application │
│  │   Process   │             │ Coordinator │             │   Process   │
│  └─────────────┘             └─────────────┘             └─────────────┘
└─────────────────────────────────────────────────────────────────┘
```

## Two-Layer Design

DMTCP consists of two distinct layers:

### 1. DMTCP Layer (Distributed Coordination)
- **Coordinator**: Central synchronization point
- **Launch/Restart**: Process lifecycle management
- **Wrappers**: System call interception via LD_PRELOAD
- **Plugin System**: Extensibility framework

### 2. MTCP Layer (Single-Process Checkpointing)
- **Core checkpointing**: Memory and thread state management
- **Restart logic**: Process image restoration
- **Low-level operations**: Direct system interaction

## Core Components

### Coordinator (`src/dmtcp_coordinator.cpp`)

The coordinator is the central authority that manages checkpoint synchronization across all worker processes.

**Key Responsibilities:**
- Maintain computation state (RUNNING, SUSPENDED, CHECKPOINTED, etc.)
- Coordinate barrier synchronization
- Handle client connections and disconnections
- Manage checkpoint intervals and timing
- Generate restart scripts

**State Machine:**
```
RUNNING → SUSPENDED → FD_LEADER_ELECTION → DRAINED → CHECKPOINTED → REFILLED → RUNNING
```

### Worker Process Architecture

Each worker process follows this structure:

```
┌─────────────────────────────────────────────┐
│            Application Process              │
├─────────────────────────────────────────────┤
│         Application Code & Data            │
├─────────────────────────────────────────────┤
│           libdmtcp.so (LD_PRELOAD)          │
│  ┌─────────────┐  ┌─────────────────────┐   │
│  │   Wrappers  │  │    Event Hooks      │   │
│  │             │  │                     │   │
│  │ socket()    │  │ DMTCP_EVENT_INIT    │   │
│  │ fork()      │  │ DMTCP_EVENT_...     │   │
│  │ exec()      │  │                     │   │
│  │ close()     │  │                     │   │
│  └─────────────┘  └─────────────────────┘   │
├─────────────────────────────────────────────┤
│              MTCP Layer                      │
│  ┌─────────────┐  ┌─────────────────────┐   │
│  │   Memory    │  │     Thread          │   │
│  │ Management  │  │   Management        │   │
│  └─────────────┘  └─────────────────────┘   │
└─────────────────────────────────────────────┘
```

## Seven-Stage Checkpoint Algorithm

DMTCP uses a sophisticated seven-stage algorithm to ensure consistent checkpoints:

### Stage 1: Normal Execution
- Applications run normally
- Checkpoint thread waits at barrier
- Coordinator monitors state

### Stage 2: Suspend User Threads
- Coordinator broadcasts checkpoint request
- SIGUSR2 signal sent to all threads
- Thread contexts saved
- File descriptor ownership recorded

### Stage 3: Elect Shared FD Leaders
- `fcntl(fd, F_SETLCK)` determines leaders
- Only leaders handle shared descriptors
- Prevents race conditions during drain

### Stage 4: Drain Kernel Buffers
- Socket leaders flush buffers using "magic cookie" tokens
- Ensures all in-flight data is captured
- Network state synchronized

### Stage 5: Write Checkpoint
- MTCP reads `/proc/self/maps` for memory layout
- Memory sections saved via `getcontext()`
- Checkpoint image written to `.dmtcp` file

### Stage 6: Refill Kernel Buffers
- Drained data resent to recreate state
- Network connections restored
- File positions reset

### Stage 7: Resume User Threads
- Thread contexts restored
- Applications continue execution
- Return to Stage 1

## System Call Interception

DMTCP achieves transparency through LD_PRELOAD-based library interposition:

```
Application Call → libdmtcp.so Wrapper → Original libc Function → Return
        ↓                    ↓                        ↓
   Pre-processing     DMTCP Logic           Post-processing
```

**Key Wrapped Functions:**
- **Socket operations**: `socket()`, `connect()`, `bind()`, `listen()`, `accept()`
- **Process operations**: `fork()`, `exec*()`, `wait()`
- **File operations**: `close()`, `dup2()`, `open()`
- **Identity functions**: `getpid()`, `getppid()`, `gettid()`

## Virtualization Layers

DMTCP virtualizes several system resources to ensure transparency:

### PID Virtualization
- Maintains translation tables between virtual and real PIDs
- Preserves process relationships across restarts
- Handles PID reuse scenarios

### File Descriptor Virtualization
- Tracks file descriptor ownership and sharing
- Manages descriptor restoration on restart
- Handles special cases (pipes, sockets, devices)

### Socket Virtualization
- Assigns globally unique socket IDs: `(hostid, pid, timestamp, connection_number)`
- Maintains connection state across checkpoints
- Supports TCP, UDP, and Unix domain sockets

## Plugin Architecture

The plugin system enables domain-specific extensions:

### Plugin Types
1. **Event Hooks**: React to DMTCP lifecycle events
2. **Wrapper Functions**: Add custom system call interposition
3. **Publish/Subscribe**: Share data between processes

### Plugin Chain
```
Application → Plugin 1 → Plugin 2 → ... → Plugin N → DMTCP Core
```

### Event Flow
```
DMTCP_EVENT_INIT
    ↓
DMTCP_EVENT_PRESUSPEND
    ↓
DMTCP_EVENT_PRECHECKPOINT
    ↓
[Checkpoint occurs]
    ↓
DMTCP_EVENT_RESTART (on restart)
    ↓
DMTCP_EVENT_RESUME
    ↓
DMTCP_EVENT_RUNNING
```

## Memory Management

### Checkpoint Image Structure
```
┌─────────────────────────────────────────────┐
│              Checkpoint Header              │
├─────────────────────────────────────────────┤
│            Memory Sections                  │
│  ┌─────────────┐  ┌─────────────────────┐   │
│  │    Code     │  │        Data         │   │
│  │   Segment   │  │      Segment        │   │
│  └─────────────┘  └─────────────────────┘   │
├─────────────────────────────────────────────┤
│            Thread Contexts                  │
├─────────────────────────────────────────────┤
│           File Descriptors                   │
├─────────────────────────────────────────────┤
│            Socket State                      │
├─────────────────────────────────────────────┤
│            Plugin Data                       │
└─────────────────────────────────────────────┘
```

### Holebase Mechanism
MTCP reserves memory regions during launch to bootstrap restart:
- **Holebase**: Reserved memory for `mtcp_restart` code
- **Bootstrap**: Copy restart code into holebase
- **Memory restoration**: Copy checkpointed memory back to original locations

## Communication Protocols

### Coordinator-Client Protocol
- **Transport**: TCP sockets (default port 7779)
- **Messages**: Structured commands and status updates
- **Authentication**: Basic host-based validation

### Message Types
- **Checkpoint requests**: Initiate checkpoint sequence
- **Status updates**: Report worker state transitions
- **Barrier synchronization**: Coordinate stage progression
- **Error handling**: Report failures and recovery

## Performance Characteristics

### Runtime Overhead
- **< 1%** typical overhead during normal execution
- **LD_PRELOAD** cost minimal for wrapped functions
- **Coordinator communication** lightweight

### Checkpoint Time
- **< 1 second** for moderate-sized applications (without compression)
- **Compression** adds seconds but reduces disk usage
- **Network I/O** dominates for distributed applications

### Restart Time
- **Memory restoration** dominates
- **Socket recreation** adds overhead
- **Plugin initialization** varies by complexity

## Security Considerations

### Privilege Requirements
- **No root privileges** required
- **User-space operation** only
- **Standard POSIX interfaces** used

### Isolation
- **Process isolation** maintained
- **No kernel modifications** needed
- **Standard file permissions** apply

## Scalability Limits

### Process Count
- **Hundreds of processes** supported
- **Coordinator** becomes bottleneck at very large scale
- **Network topology** affects performance

### Memory Size
- **Large memory applications** supported
- **Checkpoint image size** grows with memory usage
- **Compression** helps with disk I/O

### Network Scale
- **Distributed applications** across multiple hosts
- **SSH-based process launching** supported
- **Network latency** affects checkpoint time

## Integration Points

### Build System
- **Autoconf/Automake** for configuration
- **Makefile.am** for build rules
- **Plugin discovery** at build time

### External Tools
- **GDB integration** for debugging
- **Valgrind compatibility** for memory checking
- **MPI support** via MANA plugin

This architecture enables DMTCP to provide transparent checkpointing while maintaining performance and extensibility across a wide range of applications and deployment scenarios.