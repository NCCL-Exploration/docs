# Code Walkthrough: Adding a System Call Wrapper

This walkthrough demonstrates how to add a new system call wrapper to DMTCP. System call wrappers are the primary mechanism for intercepting and virtualizing system calls to enable transparent checkpointing.

## Overview

DMTCP uses LD_PRELOAD to intercept system calls. When an application calls a wrapped function, DMTCP's wrapper executes first, performs necessary bookkeeping, and then calls the original function.

## Example: Adding `stat()` Wrapper

We'll add a wrapper for the `stat()` system call to demonstrate the process.

### Step 1: Identify the System Call

First, locate where similar system calls are wrapped:

```bash
# Find existing file operation wrappers
find src/ -name "*.cpp" -exec grep -l "stat\|fstat\|lstat" {} \;
```

This shows that file operations are typically wrapped in `src/filewrappers.cpp`.

### Step 2: Add Wrapper Declaration

In `src/filewrappers.cpp`, add the wrapper function:

```cpp
// Add this with other function declarations
extern "C" {
  int stat(const char *pathname, struct stat *statbuf);
  // ... other existing declarations
}
```

### Step 3: Implement the Wrapper

```cpp
extern "C" int
stat(const char *pathname, struct stat *statbuf)
{
  // Get original function pointer
  static int (*real_stat)(const char *, struct stat *) = NULL;
  if (real_stat == NULL) {
    real_stat = (int (*)(const char *, struct stat *)) dlsym(RTLD_NEXT, "stat");
  }

  // DMTCP pre-processing
  WRAPPER_EXECUTION_DISABLE_CKPT();
  
  // Call original function
  int retval = real_stat(pathname, statbuf);
  
  // DMTCP post-processing
  if (retval == 0) {
    // Track the file if needed
    // For example, add to file descriptor tracking
    jalib::JTrace("stat called on", pathname);
  }
  
  WRAPPER_EXECUTION_ENABLE_CKPT();
  return retval;
}
```

### Step 4: Handle Special Cases

Some system calls require special handling:

#### Path Virtualization
```cpp
extern "C" int
stat(const char *pathname, struct stat *statbuf)
{
  // ... previous code ...
  
  // Virtualize path if needed
  const char *virtualPath = pathname;
  if (dmtcp::Util::isVirtualPath(pathname)) {
    virtualPath = dmtcp::Util::realPath(pathname);
  }
  
  int retval = real_stat(virtualPath, statbuf);
  
  // ... rest of wrapper ...
}
```

#### Checkpoint Safety
```cpp
// Ensure the wrapper is checkpoint-safe
WRAPPER_EXECUTION_DISABLE_CKPT();

// Perform operations that shouldn't be interrupted
int retval = real_stat(pathname, statbuf);

// Re-enable checkpointing
WRAPPER_EXECUTION_ENABLE_CKPT();
```

### Step 5: Update Build System

Add the wrapper to the appropriate Makefile:

```makefile
# In src/Makefile.am
libdmtcp_la_SOURCES += \
  filewrappers.cpp \
  # ... other sources
```

### Step 6: Add Tests

Create a test in `test/` directory:

```c
// test_stat.c
#include <stdio.h>
#include <sys/stat.h>
#include <unistd.h>

int main() {
  struct stat st;
  
  // Test basic stat functionality
  if (stat("/etc/passwd", &st) == 0) {
    printf("stat() works: size = %ld\n", st.st_size);
  } else {
    perror("stat() failed");
    return 1;
  }
  
  // Test with DMTCP checkpoint
  if (dmtcp_is_enabled()) {
    printf("DMTCP is enabled\n");
    dmtcp_checkpoint();
  }
  
  return 0;
}
```

### Step 7: Test the Wrapper

```bash
# Build DMTCP
make

# Test the wrapper
./bin/dmtcp_launch ./test_stat

# Verify it works with checkpointing
./bin/dmtcp_coordinator &
./bin/dmtcp_launch ./test_stat
# In coordinator: type 'c' to checkpoint
```

## Advanced Wrapper Patterns

### Wrapper with State Tracking

```cpp
static std::set<std::string> trackedFiles;

extern "C" int
open(const char *pathname, int flags, ...)
{
  static int (*real_open)(const char *, int, mode_t) = NULL;
  if (real_open == NULL) {
    real_open = (int (*)(const char *, int, mode_t)) dlsym(RTLD_NEXT, "open");
  }

  WRAPPER_EXECUTION_DISABLE_CKPT();
  
  int fd;
  if (flags & O_CREAT) {
    va_list args;
    va_start(args, flags);
    mode_t mode = va_arg(args, mode_t);
    va_end(args);
    fd = real_open(pathname, flags, mode);
  } else {
    fd = real_open(pathname, flags);
  }
  
  if (fd >= 0) {
    // Track the opened file
    trackedFiles.insert(pathname);
    
    // Register with DMTCP's file tracking
    dmtcp::FileConnection::trackFile(fd, pathname, flags);
  }
  
  WRAPPER_EXECUTION_ENABLE_CKPT();
  return fd;
}
```

### Wrapper with Plugin Integration

```cpp
extern "C" int
socket(int domain, int type, int protocol)
{
  static int (*real_socket)(int, int, int) = NULL;
  if (real_socket == NULL) {
    real_socket = (int (*)(int, int, int)) dlsym(RTLD_NEXT, "socket");
  }

  WRAPPER_EXECUTION_DISABLE_CKPT();
  
  int sockfd = real_socket(domain, type, protocol);
  
  if (sockfd >= 0) {
    // Notify plugins about socket creation
    DmtcpEventData_t data;
    data.socketInfo.domain = domain;
    data.socketInfo.type = type;
    data.socketInfo.protocol = protocol;
    data.socketInfo.sockfd = sockfd;
    
    DMTCP_PLUGIN_EVENT_CALL(DMTCP_EVENT_SOCKET_CREATE, &data);
    
    // Register socket with DMTCP
    dmtcp::SocketConnection::createSocket(sockfd, domain, type, protocol);
  }
  
  WRAPPER_EXECUTION_ENABLE_CKPT();
  return sockfd;
}
```

## Common Wrapper Patterns

### 1. Basic Wrapper Template
```cpp
extern "C" return_type
function_name(param_list)
{
  // Get real function
  static return_type (*real_func)(param_list) = NULL;
  if (real_func == NULL) {
    real_func = (return_type (*)(param_list)) dlsym(RTLD_NEXT, "function_name");
  }

  WRAPPER_EXECUTION_DISABLE_CKPT();
  
  // Call real function
  return_type result = real_func(params);
  
  // Post-processing if needed
  
  WRAPPER_EXECUTION_ENABLE_CKPT();
  return result;
}
```

### 2. Wrapper with Error Handling
```cpp
extern "C" int
function_with_error_handling(param_list)
{
  static int (*real_func)(param_list) = NULL;
  if (real_func == NULL) {
    real_func = (int (*)(param_list)) dlsym(RTLD_NEXT, "function_name");
  }

  WRAPPER_EXECUTION_DISABLE_CKPT();
  
  int result = real_func(params);
  
  if (result == -1) {
    // Handle error case
    int error = errno;
    JTRACE("function failed") (JASSERT_ERRNO);
    errno = error;
  } else {
    // Handle success case
  }
  
  WRAPPER_EXECUTION_ENABLE_CKPT();
  return result;
}
```

### 3. Wrapper with Resource Tracking
```cpp
extern "C" int
resource_tracking_wrapper(param_list)
{
  static int (*real_func)(param_list) = NULL;
  if (real_func == NULL) {
    real_func = (int (*)(param_list)) dlsym(RTLD_NEXT, "function_name");
  }

  WRAPPER_EXECUTION_DISABLE_CKPT();
  
  int result = real_func(params);
  
  if (result >= 0) {
    // Track the resource
    dmtcp::ResourceManager::trackResource(result, params);
  }
  
  WRAPPER_EXECUTION_ENABLE_CKPT();
  return result;
}
```

## Debugging Wrappers

### Adding Debug Output
```cpp
extern "C" int
debug_wrapper(param_list)
{
  // ... wrapper setup ...
  
  JTRACE("Entering wrapper") (function_name) (param1) (param2);
  
  int result = real_func(params);
  
  JTRACE("Exiting wrapper") (function_name) (result);
  
  // ... wrapper cleanup ...
  return result;
}
```

### Testing Wrapper Behavior
```bash
# Enable debug mode
./configure --enable-debug
make

# Run with debug output
DMTCP_DEBUG=1 ./bin/dmtcp_launch ./test_program

# Check debug logs
tail -f $DMTCP_TMPDIR/dmtcp-$USER@$HOST/jassertlog.*
```

## Best Practices

1. **Always use `WRAPPER_EXECUTION_DISABLE_CKPT()`/`WRAPPER_EXECUTION_ENABLE_CKPT()`** around system calls
2. **Cache function pointers** using `static` variables for performance
3. **Handle errors properly** and preserve `errno`
4. **Add debug traces** for complex wrappers
5. **Test with checkpointing** to ensure wrapper works correctly
6. **Consider thread safety** for global state
7. **Document special cases** and assumptions

## Common Pitfalls

1. **Forgetting to disable checkpointing** around system calls
2. **Not preserving errno** after system call failures
3. **Memory leaks** in wrapper functions
4. **Infinite recursion** when calling wrapped functions
5. **Race conditions** in multi-threaded applications
6. **Path virtualization** issues with relative paths

This walkthrough provides the foundation for adding system call wrappers to DMTCP. The same patterns apply to most system calls, with variations based on the specific requirements of each function.