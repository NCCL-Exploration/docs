# DMTCP Plugin API Reference

This document provides a quick reference for DMTCP plugin APIs used in CRAC.

## Core Plugin Structure

### Plugin Registration

#### DmtcpPluginDescriptor_t Structure
```cpp
typedef struct {
  int dmtcp_plugin_api_version;
  int dmtcp_package_version;
  const char *plugin_name;
  const char *author_name;
  const char *author_email;
  const char *description;
  DmtcpEventHookFn_t event_hook_fn;
} DmtcpPluginDescriptor_t;
```

**Usage in CRAC** (`crac.cpp:346-353`):
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
```

#### Plugin Declaration Macro
```cpp
DMTCP_DECL_PLUGIN(plugin_descriptor);
```

**Usage in CRAC** (`crac.cpp:356`):
```cpp
DMTCP_DECL_PLUGIN(cuda_plugin);
```

## Event Types

### Lifecycle Events

#### DMTCP_EVENT_INIT
**When**: Plugin is first loaded by DMTCP

**Usage in CRAC** (`crac.cpp:283-286`):
```cpp
case DMTCP_EVENT_INIT:
  dmtcp_setup_trampoline("eventfd", (void *)&eventfd_trampoline,
                         &eventfd_trampoline_info);
  break;
```

**Purpose**: Initialize plugin resources and set up trampolines.

#### DMTCP_EVENT_EXIT
**When**: Process is about to exit

**Purpose**: Clean up plugin resources.

#### DMTCP_EVENT_RUNNING
**When**: User threads start running (after initial launch, resume, or restart)

**Usage in CRAC** (`crac.cpp:288-307`):
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

**Purpose**: Handle both initial launch and restart scenarios.

### Checkpoint Events

#### DMTCP_EVENT_PRESUSPEND
**When**: Before user threads are suspended for checkpoint

**Usage in CRAC** (`crac.cpp:309-313`):
```cpp
case DMTCP_EVENT_PRESUSPEND:
  checkpoint_gpu();
  break;
```

**Purpose**: Perform operations that require running threads (like GPU checkpoint).

#### DMTCP_EVENT_PRECHECKPOINT
**When**: After threads are suspended, before memory checkpoint

**Usage in CRAC** (`crac.cpp:315-319`):
```cpp
case DMTCP_EVENT_PRECHECKPOINT:
  UNINSTALL_TRAMPOLINE(eventfd_trampoline_info);
  num_fds_found = inspect_pipes(pipe_list, MAX_PIPE_FDS);
  JASSERT(num_fds_found >= 0);
  break;
```

**Purpose**: Save state that doesn't require running threads.

#### DMTCP_EVENT_RESUME
**When**: After checkpoint is complete, threads resume

**Usage in CRAC** (`crac.cpp:321-334`):
```cpp
case DMTCP_EVENT_RESUME:
  // GPU restoration happens in RUNNING event
  break;
```

**Purpose**: Handle post-checkpoint cleanup.

### Restart Events

#### DMTCP_EVENT_RESTART
**When**: Process memory has been restored, before threads resume

**Usage in CRAC** (`crac.cpp:336-339`):
```cpp
case DMTCP_EVENT_RESTART:
  JASSERT(recreate_pipes(pipe_list, num_fds_found) != -1);
  break;
```

**Purpose**: Restore resources that need to be in place before threads run.

### Process Management Events

#### DMTCP_EVENT_PREEXEC
**When**: Before exec() system call

**Purpose**: Prepare process for exec().

#### DMTCP_EVENT_POSTEXEC
**When**: After exec() system call

**Purpose**: Reinitialize plugin after exec().

#### DMTCP_EVENT_ATFORK_PREPARE
**When**: Before fork() system call

**Purpose**: Prepare for fork().

#### DMTCP_EVENT_ATFORK_PARENT
**When**: After fork() in parent process

**Purpose**: Handle post-fork parent state.

#### DMTCP_EVENT_ATFORK_CHILD
**When**: After fork() in child process

**Purpose**: Handle post-fork child state.

## Event Data Structure

#### DmtcpEventData_t Structure
```cpp
typedef struct {
  // Event-specific data
  // Varies by event type
} DmtcpEventData_t;
```

**Usage**: Passed to event hook function, contains event-specific data.

## Event Hook Function

#### Function Signature
```cpp
typedef void (*DmtcpEventHookFn_t)(DmtcpEvent_t event, DmtcpEventData_t *data);
```

#### Implementation in CRAC
```cpp
static void cuda_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data) {
  switch (event) {
    // Handle various events
  }
}
```

## Plugin Control Macros

#### Checkpoint Control
```cpp
DMTCP_PLUGIN_DISABLE_CKPT();
DMTCP_PLUGIN_ENABLE_CKPT();
```

**Usage in CRAC** (`crac.cpp:295, 306`):
```cpp
DMTCP_PLUGIN_DISABLE_CKPT();    
// Critical section
DMTCP_PLUGIN_ENABLE_CKPT();    
```

**Purpose**: Prevent recursive checkpoints during critical operations.

## Trampoline Functions

#### Trampoline Setup
```cpp
void dmtcp_setup_trampoline(const char *symbol, 
                          void *trampoline_fn, 
                          trampoline_info_t *info);
```

**Usage in CRAC** (`crac.cpp:284-285`):
```cpp
dmtcp_setup_trampoline("eventfd", (void *)&eventfd_trampoline,
                       &eventfd_trampoline_info);
```

#### Trampoline Installation/Removal
```cpp
INSTALL_TRAMPOLINE(trampoline_info);
UNINSTALL_TRAMPOLINE(trampoline_info);
```

**Usage in CRAC**:
```cpp
// Install
INSTALL_TRAMPOLINE(eventfd_trampoline_info);

// Remove
UNINSTALL_TRAMPOLINE(eventfd_trampoline_info);
```

## Utility Functions

#### Process Memory Maps
```cpp
class ProcSelfMaps {
public:
  bool getNextArea(Area *area);
  // Other methods...
};
```

**Usage in CRAC** (`crac.cpp:216-222`):
```cpp
dmtcp::ProcSelfMaps proc_maps;
Area area;
while (proc_maps.getNextArea(&area)) {
  if (strstr(area.name, "/dev/nvidia")) {
    nvidia_device_files.push_back(area);
  }
}
```

#### Assertions
```cpp
JASSERT(condition)(optional_params...);
```

**Usage throughout CRAC**:
```cpp
JASSERT(ret == CUDA_SUCCESS)(ret);
JASSERT(num_fds_found >= 0);
```

**Purpose**: Error checking with detailed error messages.

## Event Flow Summary

### Initial Launch
```
DMTCP_EVENT_INIT → DMTCP_EVENT_RUNNING
```

### Checkpoint Cycle
```
DMTCP_EVENT_PRESUSPEND → DMTCP_EVENT_PRECHECKPOINT → 
[DMCP Checkpoint] → DMTCP_EVENT_RESUME → DMTCP_EVENT_RUNNING
```

### Restart Cycle
```
[DMCP Restart] → DMTCP_EVENT_RESTART → DMTCP_EVENT_RUNNING
```

## Best Practices

### 1. Event Handler Guidelines
- Keep event handlers short and fast
- Avoid blocking operations in event handlers
- Use appropriate events for different operations

### 2. Resource Management
- Clean up resources in DMTCP_EVENT_EXIT
- Use DMTCP_PLUGIN_DISABLE_CKPT() for critical sections
- Restore resources in appropriate events

### 3. Error Handling
- Use JASSERT for error checking
- Provide meaningful error messages
- Handle partial failures gracefully

### 4. Threading Considerations
- Event handlers run in DMTCP context
- User threads are suspended during most events
- Only DMTCP_EVENT_RUNNING has active user threads

## Common Patterns

### Initialization Pattern
```cpp
case DMTCP_EVENT_INIT:
  // Set up trampolines
  dmtcp_setup_trampoline("symbol", wrapper, &info);
  // Initialize global state
  my_global_state = initialize();
  break;
```

### Checkpoint Pattern
```cpp
case DMTCP_EVENT_PRESUSPEND:
  // Operations requiring running threads
  checkpoint_running_state();
  break;

case DMTCP_EVENT_PRECHECKPOINT:
  // Operations that can work with suspended threads
  save_static_state();
  break;
```

### Restart Pattern
```cpp
case DMTCP_EVENT_RESTART:
  // Restore resources before threads run
  restore_resources();
  break;

case DMTCP_EVENT_RUNNING:
  if (is_restart) {
    // Complete restoration with running threads
    finish_restoration();
  }
  break;
```

## See Also

- [DMTCP Documentation](https://dmtcp.sourceforge.net/) - Official DMTCP documentation
- [CUDA Checkpoint API](cuda-checkpoint-api.md) - CUDA APIs used by CRAC
- [Architecture Overview](../architecture/overview.md) - How CRAC uses DMTCP events
- [Code Walkthrough](../contributor-guide/code-walkthrough.md) - Detailed implementation