# Architecture Overview

This document provides a high-level overview of CRAC's architecture and how it integrates with DMTCP and CUDA.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Application                              │
│                    (CUDA Program)                               │
└─────────────────────┬───────────────────────────────────────────┘
                      │ CUDA Runtime/Driver API Calls
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                         CRAC Plugin                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Event Handler  │  │  Pipe Manager   │  │  State Tracker  │ │
│  │                 │  │                 │  │                 │ │
│  │ • DMTCP Events  │  │ • Pipe Inspection│  │ • CUDA State    │ │
│  │ • CUDA APIs     │  │ • Pipe Recreation│  │ • Device Files  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
                      │ DMTCP Plugin API
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                          DMTCP Core                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Coordinator   │  │   Checkpoint    │  │    Restart      │ │
│  │                 │  │   Engine        │  │    Engine       │ │
│  │ • Process Mgmt  │  │ • Memory Save   │  │ • Memory Restore│ │
│  │ • Coordination  │  │ • State Capture │  │ • State Restore │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────┬───────────────────────────────────────────┘
                      │ System Calls
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Operating System                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   CUDA Driver   │  │    File System  │  │     Memory      │ │
│  │                 │  │                 │  │                 │ │
│  │ • GPU Mgmt      │  │ • Device Files  │  │ • Process Memory│ │
│  │ • Checkpoint API│  │ • Pipes         │  │ • Mapping       │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. CRAC Plugin (`crac.cpp`)
The main plugin that bridges DMTCP and CUDA checkpoint capabilities.

**Source Reference**: `crac.cpp:1-357`

**Key Responsibilities**:
- Handle DMTCP lifecycle events
- Manage CUDA checkpoint/restate operations
- Preserve pipe file descriptors across checkpoint/restart
- Track NVIDIA device file mappings

### 2. DMTCP Integration
CRAC integrates with DMTCP through the plugin system using event hooks.

**Source Reference**: `crac.cpp:281-344` (`cuda_event_hook` function)

**Key Events**:
- `DMTCP_EVENT_PRESUSPEND`: Trigger GPU checkpoint
- `DMTCP_EVENT_PRECHECKPOINT`: Save pipe state
- `DMTCP_EVENT_RESTART`: Restore pipes
- `DMTCP_EVENT_RUNNING`: Restore GPU state

### 3. CUDA Checkpoint API Integration
CRAC uses NVIDIA's native checkpoint/restart APIs introduced in CUDA 12.4.

**Source Reference**: `crac.cpp:195-262` (`checkpoint_gpu` and `restore_gpu` functions)

**Key APIs**:
- `cuCheckpointProcessLock()`: Lock GPU for checkpointing
- `cuCheckpointProcessCheckpoint()`: Perform GPU checkpoint
- `cuCheckpointProcessRestore()`: Restore GPU state
- `cuCheckpointProcessUnlock()`: Unlock GPU after restore

## CUDA Checkpoint State Machine

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

**State Transitions**:
1. **RUNNING → LOCKED**: `cuCheckpointProcessLock()` blocks new GPU work
2. **LOCKED → CHECKPOINTED**: `cuCheckpointProcessCheckpoint()` saves GPU state
3. **CHECKPOINTED → LOCKED**: `cuCheckpointProcessRestore()` restores GPU state
4. **LOCKED → RUNNING**: `cuCheckpointProcessUnlock()` resumes GPU operations

## Data Flow

### Checkpoint Flow
```
Application → DMTCP Coordinator → CRAC Plugin → CUDA Driver API
    ↓              ↓                    ↓              ↓
Continue     Checkpoint Signal    Event Hook     GPU Checkpoint
Running       (c command)        (PRESUSPEND)   (Lock → Checkpoint)
```

### Restart Flow
```
DMTCP Restart → CRAC Plugin → CUDA Driver API → Application
      ↓            ↓              ↓              ↓
Memory      Event Hook     GPU Restore    Resume Execution
Restore     (RUNNING)    (Restore → Unlock)
```

## Memory Management

### Device File Reservation
CRAC reserves memory regions for NVIDIA device files that are released during checkpoint.

**Source Reference**: `crac.cpp:216-244`

**Process**:
1. Scan `/proc/self/maps` for `/dev/nvidia*` entries
2. Store mappings in `nvidia_device_files` vector
3. After checkpoint, reserve these regions with `mmap(PROT_NONE)`
4. During restart, release reservations before GPU restore

### Pipe Preservation
CRAC preserves pipe file descriptors across checkpoint/restart to maintain IPC.

**Source Reference**: `crac.cpp:56-170` (`inspect_pipes` and `recreate_pipes` functions)

**Process**:
1. **Checkpoint**: Scan `/proc/self/fd` for pipe entries
2. **Restart**: Recreate pipes and duplicate to original FD numbers

## Plugin Lifecycle

### Initialization
```
DMTCP_EVENT_INIT
├── Setup eventfd trampoline
└── Register plugin with DMTCP
```

**Source Reference**: `crac.cpp:283-286`

### Normal Operation
```
DMTCP_EVENT_RUNNING
├── Check CUDA state
├── If CHECKPOINTED → restore_gpu()
└── Reinstall trampolines
```

**Source Reference**: `crac.cpp:288-307`

### Checkpoint Sequence
```
DMTCP_EVENT_PRESUSPEND → checkpoint_gpu()
├── Lock GPU process
├── Checkpoint GPU state
└── Reserve device file regions

DMTCP_EVENT_PRECHECKPOINT
├── Uninstall trampolines
└── Save pipe information
```

**Source Reference**: `crac.cpp:309-319`

### Restart Sequence
```
DMTCP_EVENT_RESTART → recreate_pipes()
└── Recreate pipes with original FDs

DMTCP_EVENT_RUNNING → restore_gpu()
├── Release device file reservations
├── Restore GPU state
└── Unlock GPU process
```

**Source Reference**: `crac.cpp:336-339, 288-307`

## Error Handling and Assertions

CRAC uses DMTCP's assertion system (`JASSERT`) for error checking and debugging.

**Source Reference**: Throughout `crac.cpp`

**Key Assertions**:
- CUDA API call success verification
- Process state validation
- Memory mapping success checks
- Pipe recreation validation

## Integration Points

### With DMTCP
- **Plugin Registration**: `DMTCP_DECL_PLUGIN(cuda_plugin)` at `crac.cpp:356`
- **Event Dispatch**: `dmtcp_event_hook` function at `crac.cpp:281`
- **Checkpoint Coordination**: Works with DMTCP coordinator for timing

### With CUDA Driver
- **API Calls**: Direct calls to CUDA checkpoint APIs
- **State Management**: Tracks `CUprocessState` transitions
- **Resource Management**: Handles device file mappings

### With Operating System
- **Process Inspection**: Reads `/proc/self/maps` and `/proc/self/fd`
- **Memory Management**: Uses `mmap`/`munmap` for reservations
- **File Descriptor Management**: Preserves pipes across restart

## Design Decisions

### Why CUDA 12.4+?
- Native checkpoint/restart APIs only available from CUDA 12.4
- Eliminates need for complex interception or split-process approaches
- Provides official NVIDIA support for checkpointing

### Why DMTCP Plugin Architecture?
- Leverages DMTCP's mature checkpoint/restart infrastructure
- Provides transparent operation - no application changes needed
- Handles complex process state management automatically

### Why Pipe Preservation?
- CUDA applications often use pipes for IPC
- Standard DMTCP may not handle all pipe scenarios correctly
- Ensures communication channels survive checkpoint/restart

## See Also

- [Checkpoint Flow](checkpoint-flow.md) - Detailed step-by-step checkpoint process
- [Restart Flow](restart-flow.md) - Detailed step-by-step restart process
- [Data Structures](data-structures.md) - Key data structures and their purposes
- [Code Walkthrough](../contributor-guide/code-walkthrough.md) - File-by-file analysis