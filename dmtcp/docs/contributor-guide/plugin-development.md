# Plugin Development Guide

This guide provides comprehensive information for developing DMTCP plugins. Plugins enable domain-specific extensions and custom behavior while maintaining DMTCP's transparency.

## Plugin Architecture Overview

DMTCP plugins extend functionality through three main mechanisms:

1. **Event Hooks**: React to DMTCP lifecycle events
2. **Wrapper Functions**: Add custom system call interposition  
3. **Publish/Subscribe Service**: Share data between processes

## Plugin Types and Use Cases

### Event-Only Plugins
React to DMTCP events without modifying system calls.
- **Use cases**: Logging, monitoring, resource cleanup
- **Examples**: `test/plugin/example/`

### Wrapper Plugins
Intercept and modify system call behavior.
- **Use cases**: Custom virtualization, special resource handling
- **Examples**: `test/plugin/sleep1/`, `test/plugin/sleep2/`

### Hybrid Plugins
Combine event hooks and wrappers for complex functionality.
- **Use cases**: Application-specific checkpointing, custom resource management
- **Examples**: `test/plugin/applic-delayed-ckpt/`

## Creating a Basic Plugin

### Step 1: Plugin Structure

Create a directory for your plugin:
```
my_plugin/
├── my_plugin.c
├── Makefile
└── README
```

### Step 2: Basic Plugin Template

```c
#include "dmtcp.h"
#include <stdio.h>

// Event hook function
static void
myPlugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)
{
  switch (event) {
    case DMTCP_EVENT_INIT:
      printf("Plugin initialized\n");
      break;
      
    case DMTCP_EVENT_PRECHECKPOINT:
      printf("About to checkpoint\n");
      // Prepare for checkpoint
      break;
      
    case DMTCP_EVENT_RESTART:
      printf("Restarting from checkpoint\n");
      // Restore state after restart
      break;
      
    case DMTCP_EVENT_RESUME:
      printf("Resuming execution\n");
      // Cleanup after checkpoint/resume
      break;
      
    case DMTCP_EVENT_EXIT:
      printf("Plugin exiting\n");
      // Final cleanup
      break;
      
    default:
      break;
  }
  
  // Call next plugin in chain
  DMTCP_NEXT_EVENT_HOOK(event, data);
}

// Plugin descriptor
static DmtcpPluginDescriptor_t myPlugin = {
  DMTCP_PLUGIN_API_VERSION,
  "MyPlugin",
  "1.0",
  "Your Name",
  "your.email@example.com",
  "My custom DMTCP plugin",
  myPlugin_event_hook
};

// Register plugin with DMTCP
DMTCP_REGISTER_PLUGIN(myPlugin);
```

### Step 3: Build System

Create a `Makefile`:
```makefile
CC = gcc
CFLAGS = -fPIC -shared -Wall
LDFLAGS = -ldl

# DMTCP include path (adjust as needed)
DMTCP_INCLUDE = ../../include

# Target plugin library
TARGET = libmyplugin.so

SRCS = my_plugin.c
OBJS = $(SRCS:.c=.o)

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

%.o: %.c
	$(CC) $(CFLAGS) -I$(DMTCP_INCLUDE) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

install: $(TARGET)
	cp $(TARGET) ../../lib/
```

### Step 4: Using the Plugin

```bash
# Build the plugin
make

# Use with DMTCP
export DMTCP_PLUGIN_PATH=/path/to/my_plugin/libmyplugin.so
dmtcp_launch ./your_application
```

## Advanced Plugin Features

### Wrapper Functions

Add system call wrappers to your plugin:

```c
#include "dmtcp.h"
#include <dlfcn.h>
#include <unistd.h>

// Wrapper for sleep() function
static unsigned int
myPlugin_sleep(unsigned int seconds)
{
  static unsigned int (*real_sleep)(unsigned int) = NULL;
  if (real_sleep == NULL) {
    real_sleep = (unsigned int (*)(unsigned int)) dlsym(RTLD_NEXT, "sleep");
  }
  
  printf("Plugin: sleep(%u) called\n", seconds);
  
  // Call original function
  return real_sleep(seconds);
}

// Update event hook to include wrapper registration
static void
myPlugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)
{
  switch (event) {
    case DMTCP_EVENT_INIT:
      printf("Plugin initialized\n");
      // Register wrapper (if needed)
      break;
    // ... other cases
  }
  
  DMTCP_NEXT_EVENT_HOOK(event, data);
}

// Update plugin descriptor
static DmtcpPluginDescriptor_t myPlugin = {
  DMTCP_PLUGIN_API_VERSION,
  "MyPlugin",
  "1.0",
  "Your Name",
  "your.email@example.com",
  "My custom DMTCP plugin with wrappers",
  myPlugin_event_hook
};

DMTCP_REGISTER_PLUGIN(myPlugin);
```

### State Management

Maintain plugin state across checkpoints:

```c
#include "dmtcp.h"
#include <stdlib.h>

// Plugin state structure
typedef struct {
  int counter;
  char *message;
  FILE *log_file;
} PluginState;

static PluginState *plugin_state = NULL;

// Save state before checkpoint
static void
save_plugin_state(void)
{
  if (plugin_state) {
    // Save to DMTCP's checkpoint area
    dmtcp_send_key_val_pair_to_coordinator("myplugin_counter", 
                                          &plugin_state->counter, sizeof(int));
    dmtcp_send_key_val_pair_to_coordinator("myplugin_message", 
                                          plugin_state->message, 
                                          strlen(plugin_state->message) + 1);
  }
}

// Restore state after restart
static void
restore_plugin_state(void)
{
  if (!plugin_state) {
    plugin_state = malloc(sizeof(PluginState));
  }
  
  // Restore from coordinator
  size_t size;
  void *data;
  
  data = dmtcp_send_query_to_coordinator("myplugin_counter", &size);
  if (data && size == sizeof(int)) {
    plugin_state->counter = *(int*)data;
  }
  
  data = dmtcp_send_query_to_coordinator("myplugin_message", &size);
  if (data) {
    free(plugin_state->message);
    plugin_state->message = malloc(size);
    memcpy(plugin_state->message, data, size);
  }
}

// Updated event hook
static void
myPlugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)
{
  switch (event) {
    case DMTCP_EVENT_INIT:
      plugin_state = malloc(sizeof(PluginState));
      plugin_state->counter = 0;
      plugin_state->message = strdup("Hello from plugin");
      plugin_state->log_file = fopen("plugin.log", "w");
      break;
      
    case DMTCP_EVENT_PRECHECKPOINT:
      save_plugin_state();
      if (plugin_state->log_file) {
        fflush(plugin_state->log_file);
      }
      break;
      
    case DMTCP_EVENT_RESTART:
      restore_plugin_state();
      if (plugin_state->log_file) {
        plugin_state->log_file = fopen("plugin.log", "a");
      }
      break;
      
    case DMTCP_EVENT_EXIT:
      if (plugin_state) {
        if (plugin_state->log_file) {
          fclose(plugin_state->log_file);
        }
        free(plugin_state->message);
        free(plugin_state);
      }
      break;
      
    default:
      break;
  }
  
  DMTCP_NEXT_EVENT_HOOK(event, data);
}
```

### Thread-Safe Plugins

Ensure thread safety for multi-threaded applications:

```c
#include "dmtcp.h"
#include <pthread.h>

static pthread_mutex_t plugin_mutex = PTHREAD_MUTEX_INITIALIZER;
static int shared_counter = 0;

static void
thread_safe_function(void)
{
  pthread_mutex_lock(&plugin_mutex);
  shared_counter++;
  printf("Counter: %d\n", shared_counter);
  pthread_mutex_unlock(&plugin_mutex);
}

static void
myPlugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)
{
  switch (event) {
    case DMTCP_EVENT_INIT:
      pthread_mutex_init(&plugin_mutex, NULL);
      break;
      
    case DMTCP_EVENT_EXIT:
      pthread_mutex_destroy(&plugin_mutex);
      break;
      
    default:
      break;
  }
  
  DMTCP_NEXT_EVENT_HOOK(event, data);
}
```

## Event Reference

### Lifecycle Events

| Event | Description | When Called |
|-------|-------------|-------------|
| `DMTCP_EVENT_INIT` | Plugin initialization | Process startup |
| `DMTCP_EVENT_EXIT` | Plugin cleanup | Process exit |
| `DMTCP_EVENT_PRECHECKPOINT` | Before checkpoint | Checkpoint begins |
| `DMTCP_EVENT_PRECHECKPOINT` | Before checkpoint | Checkpoint begins |
| `DMTCP_EVENT_RESTART` | After restart | Process restart |
| `DMTCP_EVENT_RESUME` | After resume | Checkpoint complete |
| `DMTCP_EVENT_RUNNING` | Normal execution | After resume/restart |

### Process Events

| Event | Description | When Called |
|-------|-------------|-------------|
| `DMTCP_EVENT_PRE_EXEC` | Before exec() | Process about to exec |
| `DMTCP_EVENT_POST_EXEC` | After exec() | Process completed exec |
| `DMTCP_EVENT_ATFORK_PREPARE` | Before fork() | Fork preparation |
| `DMTCP_EVENT_ATFORK_PARENT` | In parent after fork() | Parent continues |
| `DMTCP_EVENT_ATFORK_CHILD` | In child after fork() | Child starts |
| `DMTCP_EVENT_VFORK_PARENT` | In parent after vfork() | Parent continues |
| `DMTCP_EVENT_VFORK_CHILD` | In child after vfork() | Child starts |

### Thread Events

| Event | Description | When Called |
|-------|-------------|-------------|
| `DMTCP_EVENT_PTHREAD_START` | Thread creation | New thread starts |
| `DMTCP_EVENT_PTHREAD_EXIT` | Thread exit | Thread terminates |
| `DMTCP_EVENT_PTHREAD_RETURN` | Thread completion | Thread function returns |

### File Descriptor Events

| Event | Description | When Called |
|-------|-------------|-------------|
| `DMTCP_EVENT_OPEN_FD` | File opened | New file descriptor |
| `DMTCP_EVENT_CLOSE_FD` | File closed | File descriptor closed |
| `DMTCP_EVENT_DUP_FD` | File duplicated | dup()/dup2() called |

## Plugin Examples

### Example 1: Simple Logging Plugin

```c
#include "dmtcp.h"
#include <stdio.h>
#include <time.h>

static FILE *log_file = NULL;

static void
log_event(const char *event_name)
{
  time_t now = time(NULL);
  fprintf(log_file, "[%s] %s\n", ctime(&now), event_name);
  fflush(log_file);
}

static void
logging_plugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)
{
  switch (event) {
    case DMTCP_EVENT_INIT:
      log_file = fopen("dmtcp_events.log", "w");
      log_event("DMTCP initialized");
      break;
      
    case DMTCP_EVENT_PRECHECKPOINT:
      log_event("Checkpoint starting");
      break;
      
    case DMTCP_EVENT_RESTART:
      log_event("Restarting from checkpoint");
      break;
      
    case DMTCP_EVENT_RESUME:
      log_event("Resuming execution");
      break;
      
    case DMTCP_EVENT_EXIT:
      log_event("DMTCP exiting");
      fclose(log_file);
      break;
      
    default:
      break;
  }
  
  DMTCP_NEXT_EVENT_HOOK(event, data);
}

static DmtcpPluginDescriptor_t loggingPlugin = {
  DMTCP_PLUGIN_API_VERSION,
  "LoggingPlugin",
  "1.0",
  "DMTCP Team",
  "dmtcp-forum@lists.sourceforge.net",
  "Logs DMTCP events to file",
  logging_plugin_event_hook
};

DMTCP_REGISTER_PLUGIN(loggingPlugin);
```

### Example 2: Resource Monitoring Plugin

```c
#include "dmtcp.h"
#include <stdio.h>
#include <sys/resource.h>
#include <unistd.h>

typedef struct {
  long max_rss;
  long max_open_files;
  double max_cpu_time;
} ResourceStats;

static ResourceStats stats = {0};

static void
update_resource_stats(void)
{
  struct rusage usage;
  if (getrusage(RUSAGE_SELF, &usage) == 0) {
    if (usage.ru_maxrss > stats.max_rss) {
      stats.max_rss = usage.ru_maxrss;
    }
    
    // Count open files
    long open_files = 0;
    for (int fd = 0; fd < 1024; fd++) {
      if (fcntl(fd, F_GETFD) != -1) {
        open_files++;
      }
    }
    if (open_files > stats.max_open_files) {
      stats.max_open_files = open_files;
    }
    
    double cpu_time = usage.ru_utime.tv_sec + usage.ru_utime.tv_usec / 1000000.0 +
                     usage.ru_stime.tv_sec + usage.ru_stime.tv_usec / 1000000.0;
    if (cpu_time > stats.max_cpu_time) {
      stats.max_cpu_time = cpu_time;
    }
  }
}

static void
monitoring_plugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)
{
  switch (event) {
    case DMTCP_EVENT_INIT:
      printf("Resource monitoring started\n");
      break;
      
    case DMTCP_EVENT_PRECHECKPOINT:
      update_resource_stats();
      printf("Max RSS: %ld KB\n", stats.max_rss);
      printf("Max open files: %ld\n", stats.max_open_files);
      printf("Max CPU time: %.2f seconds\n", stats.max_cpu_time);
      break;
      
    case DMTCP_EVENT_EXIT:
      update_resource_stats();
      printf("Final resource usage:\n");
      printf("  Max RSS: %ld KB\n", stats.max_rss);
      printf("  Max open files: %ld\n", stats.max_open_files);
      printf("  Max CPU time: %.2f seconds\n", stats.max_cpu_time);
      break;
      
    default:
      break;
  }
  
  DMTCP_NEXT_EVENT_HOOK(event, data);
}

static DmtcpPluginDescriptor_t monitoringPlugin = {
  DMTCP_PLUGIN_API_VERSION,
  "MonitoringPlugin",
  "1.0",
  "DMTCP Team",
  "dmtcp-forum@lists.sourceforge.net",
  "Monitors resource usage during execution",
  monitoring_plugin_event_hook
};

DMTCP_REGISTER_PLUGIN(monitoringPlugin);
```

## Best Practices

### 1. Error Handling
```c
static void
safe_operation(void)
{
  FILE *file = fopen("important.txt", "r");
  if (!file) {
    JTRACE("Failed to open file") (JASSERT_ERRNO);
    return;
  }
  
  // Use file...
  
  if (fclose(file) != 0) {
    JTRACE("Failed to close file") (JASSERT_ERRNO);
  }
}
```

### 2. Memory Management
```c
static void
cleanup_resources(void)
{
  if (plugin_state) {
    if (plugin_state->allocated_memory) {
      free(plugin_state->allocated_memory);
    }
    if (plugin_state->file_handle) {
      fclose(plugin_state->file_handle);
    }
    free(plugin_state);
    plugin_state = NULL;
  }
}
```

### 3. Thread Safety
```c
static pthread_mutex_t state_mutex = PTHREAD_MUTEX_INITIALIZER;

static void
thread_safe_update(int value)
{
  pthread_mutex_lock(&state_mutex);
  // Update shared state
  pthread_mutex_unlock(&state_mutex);
}
```

### 4. Checkpoint Safety
```c
static void
checkpoint_safe_operation(void)
{
  WRAPPER_EXECUTION_DISABLE_CKPT();
  
  // Perform operation that shouldn't be interrupted
  
  WRAPPER_EXECUTION_ENABLE_CKPT();
}
```

## Testing Plugins

### Unit Testing
```c
// test_my_plugin.c
#include "dmtcp.h"
#include <assert.h>

void test_plugin_functionality(void)
{
  // Test plugin initialization
  assert(dmtcp_is_enabled());
  
  // Test plugin behavior
  // ... test code ...
  
  printf("Plugin tests passed\n");
}

int main(void)
{
  test_plugin_functionality();
  return 0;
}
```

### Integration Testing
```bash
# Build plugin
make

# Test with simple application
dmtcp_launch --with-plugin ./libmyplugin.so ./test_program

# Test checkpointing
dmtcp_coordinator &
dmtcp_launch --with-plugin ./libmyplugin.so ./test_program
# In coordinator: type 'c' to checkpoint
```

## Debugging Plugins

### Debug Output
```c
static void
debug_event(DmtcpEvent_t event)
{
  JTRACE("Plugin event") (event);
  
  // More detailed debugging
  switch (event) {
    case DMTCP_EVENT_PRECHECKPOINT:
      JTRACE("Preparing for checkpoint");
      break;
    // ... other cases
  }
}
```

### Debug Mode
```bash
# Enable DMTCP debug
DMTCP_DEBUG=1 dmtcp_launch --with-plugin ./libmyplugin.so ./application

# Check debug logs
tail -f $DMTCP_TMPDIR/dmtcp-$USER@$HOST/jassertlog.*
```

## Plugin Distribution

### Packaging
```bash
# Create plugin package
tar czf myplugin-1.0.tar.gz myplugin/

# Include installation script
#!/bin/bash
# install.sh
cp libmyplugin.so /usr/local/lib/dmtcp/
echo "Plugin installed to /usr/local/lib/dmtcp/"
```

### Documentation
Include comprehensive README with:
- Plugin purpose and functionality
- Build instructions
- Usage examples
- Configuration options
- Troubleshooting guide

This guide provides foundation for developing DMTCP plugins. The plugin system enables powerful extensions while maintaining DMTCP's core transparency and performance characteristics.