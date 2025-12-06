# DMTCP Wrapper Function System

This document explains DMTCP's wrapper function system, which enables transparent checkpointing by intercepting system calls and library functions through the `LD_PRELOAD` mechanism.

## Overview

DMTCP achieves transparency by intercepting system calls and library functions without requiring application source changes. This is accomplished through:

1. **LD_PRELOAD Mechanism** - Injects `libdmtcp.so` into application address space
2. **Function Interposition** - Replaces original functions with DMTCP wrappers
3. **Virtualization Tables** - Maintains mappings between real and virtual resources
4. **Wrapper Chains** - Handles multiple plugins wrapping the same function

## LD_PRELOAD Mechanism

### How It Works

```bash
# DMTCP launches applications with LD_PRELOAD
LD_PRELOAD=/path/to/libdmtcp.so ./myapplication

# The wrapper is automatically injected into the process
dmtcp_launch ./myapplication  # Sets up LD_PRELOAD automatically
```

### Library Loading Order

```
Application Code
       ↓
libdmtcp.so (LD_PRELOAD) ← First to be consulted
       ↓
libc.so (standard library)
       ↓
Kernel System Calls
```

### Wrapper Resolution

```cpp
// Wrapper function gets first chance to handle calls
extern "C" int open(const char *pathname, int flags, ...) {
  // DMTCP processing here
  return real_open(pathname, flags, mode);
}

// Original function obtained via dlsym
static int (*real_open) (const char *, int, mode_t) = NULL;
if (real_open == NULL) {
  real_open = (int (*)(const char *, int, mode_t)) dmtcp_dlsym(RTLD_NEXT, "open");
}
```

## Wrapper Function Categories

### Process and Thread Management

#### fork() Wrapper
```cpp
// In execwrappers.cpp
extern "C" pid_t fork() {
  static pid_t (*real_fork) () = NULL;
  if (real_fork == NULL) {
    real_fork = (pid_t (*)()) dmtcp_dlsym(RTLD_NEXT, "fork");
  }
  
  // Pre-fork processing
  prepareForFork();
  
  pid_t pid = real_fork();
  
  if (pid == 0) {
    // Child process
    initializeAfterFork(true);
  } else if (pid > 0) {
    // Parent process
    initializeAfterFork(false);
    VirtualPidTable::instance().addForkedChild(pid);
  }
  
  return pid;
}
```

#### getpid() Wrapper
```cpp
// In pidwrappers.cpp
extern "C" pid_t getpid() {
  static pid_t (*real_getpid) () = NULL;
  if (real_getpid == NULL) {
    real_getpid = (pid_t (*)()) dmtcp_dlsym(RTLD_NEXT, "getpid");
  }
  
  // Return virtual PID, not real PID
  return VirtualPidTable::instance().getVirtualPid(real_getpid());
}
```

#### clone() Wrapper
```cpp
// In pidwrappers.cpp
extern "C" int clone(int (*fn)(void *), void *child_stack, 
                    int flags, void *arg, ... /* pid_t *ptid, 
                    struct user_desc *tls, pid_t *ctid */) {
  static int (*real_clone) (int (*)(void *), void *, int, void *, 
                           pid_t *, struct user_desc *, pid_t *) = NULL;
  if (real_clone == NULL) {
    real_clone = (int (*)(int (*)(void *), void *, int, void *, 
                        pid_t *, struct user_desc *, pid_t *)) 
                 dmtcp_dlsym(RTLD_NEXT, "clone");
  }
  
  // Handle variable arguments
  va_list args;
  va_start(args, arg);
  pid_t *ptid = va_arg(args, pid_t *);
  struct user_desc *tls = va_arg(args, struct user_desc *);
  pid_t *ctid = va_arg(args, pid_t *);
  va_end(args);
  
  // Create thread with DMTCP wrapper
  return dmtcp_clone(fn, child_stack, flags, arg, ptid, tls, ctid);
}
```

### File Descriptor Operations

#### open() Wrapper
```cpp
// In filewrappers.cpp
extern "C" int open(const char *pathname, int flags, ...) {
  static int (*real_open) (const char *, int, mode_t) = NULL;
  if (real_open == NULL) {
    real_open = (int (*)(const char *, int, mode_t)) dmtcp_dlsym(RTLD_NEXT, "open");
  }
  
  mode_t mode = 0;
  if (flags & O_CREAT) {
    va_list args;
    va_start(args, flags);
    mode = va_arg(args, mode_t);
    va_end(args);
  }
  
  int fd = real_open(pathname, flags, mode);
  
  if (fd >= 0) {
    // Track new file descriptor
    ConnectionList::instance().add(fd, pathname);
    
    // Notify plugins
    DmtcpEventData_t data;
    data.event_data.file_info.filename = pathname;
    data.event_data.file_info.fd = fd;
    data.event_data.file_info.flags = flags;
    data.event_data.file_info.mode = mode;
    dmtcp_broadcast_event(DMTCP_EVENT_REGISTER_FD, &data);
  }
  
  return fd;
}
```

#### close() Wrapper
```cpp
// In filewrappers.cpp
extern "C" int close(int fd) {
  static int (*real_close) (int) = NULL;
  if (real_close == NULL) {
    real_close = (int (*)(int)) dmtcp_dlsym(RTLD_NEXT, "close");
  }
  
  // Notify plugins before closing
  DmtcpEventData_t data;
  data.event_data.fd_info.fd = fd;
  dmtcp_broadcast_event(DMTCP_EVENT_UNREGISTER_FD, &data);
  
  // Remove from connection tracking
  ConnectionList::instance().remove(fd);
  
  return real_close(fd);
}
```

### Socket Operations

#### socket() Wrapper
```cpp
// In socketwrappers.cpp
extern "C" int socket(int domain, int type, int protocol) {
  static int (*real_socket) (int, int, int) = NULL;
  if (real_socket == NULL) {
    real_socket = (int (*)(int, int, int)) dmtcp_dlsym(RTLD_NEXT, "socket");
  }
  
  int sockfd = real_socket(domain, type, protocol);
  
  if (sockfd >= 0) {
    // Create TCP connection object
    TcpConnection *conn = new TcpConnection(domain, type, protocol);
    ConnectionList::instance().add(sockfd, conn);
    
    // Set non-blocking for draining
    int flags = fcntl(sockfd, F_GETFL);
    fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);
  }
  
  return sockfd;
}
```

#### connect() Wrapper
```cpp
// In socketwrappers.cpp
extern "C" int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen) {
  static int (*real_connect) (int, const struct sockaddr *, socklen_t) = NULL;
  if (real_connect == NULL) {
    real_connect = (int (*)(int, const struct sockaddr *, socklen_t)) 
                  dmtcp_dlsym(RTLD_NEXT, "connect");
  }
  
  int result = real_connect(sockfd, addr, addrlen);
  
  if (result == 0) {
    // Connection successful - update connection state
    TcpConnection *conn = (TcpConnection *) ConnectionList::instance().retrieve(sockfd);
    if (conn) {
      conn->markAsConnected(addr, addrlen);
    }
  }
  
  return result;
}
```

#### pipe() Wrapper
```cpp
// In filewrappers.cpp
extern "C" int pipe(int pipefd[2]) {
  static int (*real_pipe) (int[2]) = NULL;
  if (real_pipe == NULL) {
    real_pipe = (int (*)(int[2])) dmtcp_dlsym(RTLD_NEXT, "pipe");
  }
  
  int result = real_pipe(pipefd);
  
  if (result == 0) {
    // Promote pipe to bidirectional socketpair for draining
    int sv[2];
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, sv) == 0) {
      // Close original pipe
      real_close(pipefd[0]);
      real_close(pipefd[1]);
      
      // Return socketpair file descriptors
      pipefd[0] = sv[0];
      pipefd[1] = sv[1];
      
      // Track as socket connections
      ConnectionList::instance().add(pipefd[0], new TcpConnection(AF_UNIX, SOCK_STREAM, 0));
      ConnectionList::instance().add(pipefd[1], new TcpConnection(AF_UNIX, SOCK_STREAM, 0));
    }
  }
  
  return result;
}
```

### Memory Operations

#### mmap() Wrapper
```cpp
// In memwrappers.cpp
extern "C" void *mmap(void *addr, size_t length, int prot, int flags, 
                      int fd, off_t offset) {
  static void *(*real_mmap) (void *, size_t, int, int, int, off_t) = NULL;
  if (real_mmap == NULL) {
    real_mmap = (void *(*)(void *, size_t, int, int, int, off_t)) 
                dmtcp_dlsym(RTLD_NEXT, "mmap");
  }
  
  void *result = real_mmap(addr, length, prot, flags, fd, offset);
  
  if (result != MAP_FAILED) {
    // Track memory mapping for checkpoint
    MemoryMapping::instance().add(result, length, prot, flags, fd, offset);
  }
  
  return result;
}
```

#### munmap() Wrapper
```cpp
// In memwrappers.cpp
extern "C" int munmap(void *addr, size_t length) {
  static int (*real_munmap) (void *, size_t) = NULL;
  if (real_munmap == NULL) {
    real_munmap = (int (*)(void *, size_t)) dmtcp_dlsym(RTLD_NEXT, "munmap");
  }
  
  // Remove from memory tracking
  MemoryMapping::instance().remove(addr, length);
  
  return real_munmap(addr, length);
}
```

## Virtualization Tables

### VirtualPidTable

```cpp
// In virtualpidtable.cpp
class VirtualPidTable {
private:
  // Maps real PIDs to virtual PIDs
  std::map<pid_t, pid_t> realToVirtual;
  
  // Maps virtual PIDs to real PIDs  
  std::map<pid_t, pid_t> virtualToReal;
  
  // Maps real TIDs to virtual TIDs
  std::map<pid_t, pid_t> realToVirtualTid;
  
  // Maps virtual TIDs to real TIDs
  std::map<pid_t, pid_t> virtualToRealTid;
  
public:
  // Add new process to table
  void addNewProcess(pid_t realPid, pid_t virtualPid) {
    realToVirtual[realPid] = virtualPid;
    virtualToReal[virtualPid] = realPid;
  }
  
  // Get virtual PID from real PID
  pid_t getVirtualPid(pid_t realPid) {
    if (realToVirtual.find(realPid) != realToVirtual.end()) {
      return realToVirtual[realPid];
    }
    // If not found, this is a new process
    return realPid;
  }
  
  // Get real PID from virtual PID
  pid_t getRealPid(pid_t virtualPid) {
    if (virtualToReal.find(virtualPid) != virtualToReal.end()) {
      return virtualToReal[virtualPid];
    }
    return virtualPid;
  }
  
  // Check for PID conflicts
  bool isConflictingPid(pid_t pid) {
    return virtualToReal.find(pid) != virtualToReal.end();
  }
  
  // Handle fork events
  void handleFork(pid_t childPid, pid_t parentVirtualPid) {
    pid_t childVirtualPid = generateNewVirtualPid();
    addNewProcess(childPid, childVirtualPid);
  }
};
```

### ConnectionList

```cpp
// In connectionmanager.cpp
class ConnectionList {
private:
  // Map file descriptors to connections
  std::map<int, Connection*> fdToConnection;
  
  // Map connection IDs to connections
  std::map<ConnectionIdentifier, Connection*> idToConnection;
  
public:
  // Add new connection
  void add(int fd, Connection *conn) {
    fdToConnection[fd] = conn;
    idToConnection[conn->id()] = conn;
  }
  
  // Retrieve connection by file descriptor
  Connection* retrieve(int fd) {
    if (fdToConnection.find(fd) != fdToConnection.end()) {
      return fdToConnection[fd];
    }
    return NULL;
  }
  
  // Retrieve connection by ID
  Connection* retrieve(const ConnectionIdentifier& id) {
    if (idToConnection.find(id) != idToConnection.end()) {
      return idToConnection[id];
    }
    return NULL;
  }
  
  // Remove connection
  void remove(int fd) {
    Connection *conn = retrieve(fd);
    if (conn) {
      idToConnection.erase(conn->id());
      fdToConnection.erase(fd);
      delete conn;
    }
  }
  
  // Iterate over all connections
  std::map<int, Connection*>::iterator begin() { return fdToConnection.begin(); }
  std::map<int, Connection*>::iterator end() { return fdToConnection.end(); }
};
```

## Wrapper Chains and Plugin Integration

### Multiple Wrapper Support

```cpp
// In dmtcp_dlsym.cpp
void *dmtcp_dlsym(void *handle, const char *symbol) {
  // First check if any plugin wraps this symbol
  void *plugin_wrapper = dmtcp_find_plugin_wrapper(symbol);
  if (plugin_wrapper) {
    return plugin_wrapper;
  }
  
  // Then check for DMTCP wrapper
  void *dmtcp_wrapper = dmtcp_find_dmtcp_wrapper(symbol);
  if (dmtcp_wrapper) {
    return dmtcp_wrapper;
  }
  
  // Finally, get the real function
  return dlsym(handle, symbol);
}
```

### Plugin Wrapper Registration

```cpp
// Plugin registers wrapper function
void dmtcp_wrap_function(const char *symbol, void *wrapper_func) {
  plugin_wrappers[symbol] = wrapper_func;
}

// DMTCP wrapper calls next in chain
extern "C" int open(const char *pathname, int flags, ...) {
  // DMTCP processing
  
  // Call next wrapper in chain or real function
  static int (*next_open) (const char *, int, mode_t) = NULL;
  if (next_open == NULL) {
    next_open = (int (*)(const char *, int, mode_t)) 
                dmtcp_dlsym(RTLD_NEXT, "open");
  }
  
  // ... continue with next_open
}
```

## Special Cases and Edge Cases

### Statically Linked Binaries

```cpp
// In dmtcp_nocheckpoint.c
// For applications that can't use LD_PRELOAD
int dmtcp_nocheckpoint() {
  // Disable checkpointing for this process
  setenv("DMTCP_NO_CHECKPOINT", "1", 1);
  return 0;
}

// Wrapper that detects statically linked binaries
extern "C" void _init() {
  if (is_statically_linked()) {
    JNOTE("Statically linked binary detected, checkpointing disabled");
    setenv("DMTCP_NO_CHECKPOINT", "1", 1);
    return;
  }
  
  // Normal DMTCP initialization
  initialize_dmtcp();
}
```

### Thread-Local Storage

```cpp
// Thread-local wrapper state
static __thread int wrapper_recursion_depth = 0;

extern "C" int read(int fd, void *buf, size_t count) {
  // Prevent recursive wrapper calls
  if (wrapper_recursion_depth > 0) {
    static int (*real_read) (int, void *, size_t) = NULL;
    if (real_read == NULL) {
      real_read = (int (*)(int, void *, size_t)) dmtcp_dlsym(RTLD_NEXT, "read");
    }
    return real_read(fd, buf, count);
  }
  
  wrapper_recursion_depth++;
  
  // DMTCP processing
  int result = real_read(fd, buf, count);
  
  wrapper_recursion_depth--;
  return result;
}
```

### Signal-Safe Wrappers

```cpp
// Signal-safe wrapper for async-signal-safe functions
extern "C" int write(int fd, const void *buf, size_t count) {
  static int (*real_write) (int, const void *, size_t) = NULL;
  if (real_write == NULL) {
    real_write = (int (*)(int, const void *, size_t)) dmtcp_dlsym(RTLD_NEXT, "write");
  }
  
  // Check if we're in a signal handler
  if (in_signal_handler()) {
    // Use direct syscall for signal safety
    return syscall(SYS_write, fd, buf, count);
  }
  
  // Normal wrapper processing
  return real_write(fd, buf, count);
}
```

## Performance Considerations

### Function Pointer Caching

```cpp
// Cache function pointers to avoid repeated dlsym calls
template<typename T>
static T get_real_function(const char *name) {
  static T real_func = NULL;
  if (real_func == NULL) {
    real_func = (T) dmtcp_dlsym(RTLD_NEXT, name);
  }
  return real_func;
}

// Usage
extern "C" int close(int fd) {
  static int (*real_close) (int) = NULL;
  real_close = get_real_function<int (*)(int)>("close");
  return real_close(fd);
}
```

### Fast Path Optimization

```cpp
// Fast path for common operations
extern "C" ssize_t read(int fd, void *buf, size_t count) {
  static int (*real_read) (int, void *, size_t) = NULL;
  if (real_read == NULL) {
    real_read = (int (*)(int, void *, size_t)) dmtcp_dlsym(RTLD_NEXT, "read");
  }
  
  // Fast path: if not a tracked FD, call directly
  if (!ConnectionList::instance().isTracked(fd)) {
    return real_read(fd, buf, count);
  }
  
  // Slow path: DMTCP processing
  return tracked_read(fd, buf, count);
}
```

## Debugging Wrappers

### Wrapper Tracing

```cpp
// Enable wrapper tracing for debugging
#ifdef DMTCP_WRAPPER_DEBUG
#define WRAPPER_TRACE(name, ...) \
  JTRACE("Wrapper called") (name) (__VA_ARGS__)
#else
#define WRAPPER_TRACE(name, ...)
#endif

extern "C" int open(const char *pathname, int flags, ...) {
  WRAPPER_TRACE("open", pathname, flags);
  
  // ... wrapper implementation
}
```

### GDB Integration

```bash
# Set breakpoint in wrapper function
gdb myapplication
(gdb) break open
(gdb) run

# Examine wrapper chain
(gdb) print dmtcp_find_plugin_wrapper("open")
(gdb) print dlsym(RTLD_DEFAULT, "open")
```

## Testing Wrappers

### Unit Tests

```cpp
// Test wrapper functionality
TEST(WrappersTest, PidVirtualization) {
  pid_t real_pid = syscall(SYS_getpid);
  pid_t virtual_pid = getpid();
  
  EXPECT_NE(real_pid, virtual_pid);
  EXPECT_EQ(virtual_pid, VirtualPidTable::instance().getVirtualPid(real_pid));
}

TEST(WrappersTest, FileDescriptorTracking) {
  int fd = open("/tmp/test", O_CREAT | O_WRONLY, 0644);
  EXPECT_GE(fd, 0);
  
  Connection *conn = ConnectionList::instance().retrieve(fd);
  EXPECT_NE(conn, nullptr);
  EXPECT_EQ(conn->type(), FILE_CONNECTION);
  
  close(fd);
  conn = ConnectionList::instance().retrieve(fd);
  EXPECT_EQ(conn, nullptr);
}
```

### Integration Tests

```bash
# Test wrapper with real application
dmtcp_launch --with-plugin ./libtestwrapper.so test/dmtcp1
dmtcp_command --checkpoint
dmtcp_restart ckpt_dmtcp1_*.dmtcp

# Verify wrapper behavior through logs
grep "Wrapper called" /tmp/dmtcp-$USER@$(hostname)/jassertlog.*
```

---

The wrapper function system is the core mechanism that enables DMTCP's transparent checkpointing. Understanding this system is essential for extending DMTCP's functionality and troubleshooting wrapper-related issues.