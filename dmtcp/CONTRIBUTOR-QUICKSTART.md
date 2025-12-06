# DMTCP Contributor Quick Start Guide

This guide helps new contributors get started with DMTCP development quickly and efficiently.

## Prerequisites

### System Requirements

- **Linux**: DMTCP primarily targets Linux systems
- **GCC**: Version 4.8+ for C++11 support
- **GNU Autotools**: autoconf, automake, libtool
- **Git**: For source code management
- **GDB**: For debugging (recommended)

### Development Tools

```bash
# Ubuntu/Debian
sudo apt-get install build-essential autoconf automake libtool git gdb

# CentOS/RHEL
sudo yum install gcc gcc-c++ autoconf automake libtool git gdb

# Fedora
sudo dnf install gcc gcc-c++ autoconf automake libtool git gdb
```

## Getting the Source Code

```bash
# Clone the repository
git clone https://github.com/dmtcp/dmtcp.git
cd dmtcp

# Explore the structure
ls -la
tree -L 2  # If tree is installed
```

## Building DMTCP

### Basic Build

```bash
# Generate build configuration
./autogen.sh
./configure

# Build DMTCP
make -j$(nproc)

# Verify installation
./bin/dmtcp_launch --version
```

### Development Build with Debugging

```bash
# Configure with debugging symbols and logging
./configure CFLAGS="-g3 -O0" CXXFLAGS="-g3 -O0" --enable-logging

# Clean build
make clean && make -j$(nproc)

# Test the build
make check
```

### Build Options

| Option | Purpose | Recommended for Development |
|--------|---------|----------------------------|
| `--enable-debug` | Enable debug assertions | Yes |
| `--enable-logging` | Enable JTRACE logging | Yes |
| `--enable-fault-injection` | Enable fault injection for testing | Advanced |
| `--with-mpi` | Enable MPI support (MANA) | If working with MPI |
| `--enable-pid-virtualization` | Enable PID virtualization | Default |

## Your First Contribution

### 1. Run the Test Suite

```bash
# Run all tests
make check

# Run specific test
make check-dmtcp1

# Run with verbose output
make check VERBOSE=1
```

### 2. Explore the Codebase

#### Key Files to Understand

```bash
# Core coordinator
less src/dmtcp_coordinator.cpp

# Process launch logic
less src/dmtcp_launch.cpp

# Worker implementation
less src/dmtcpworker.cpp

# Plugin system
less include/dmtcp/plugin.h

# Example plugin
ls test/plugin/example/
```

#### Understanding the Flow

```bash
# Trace a simple application
./bin/dmtcp_launch test/dmtcp1 &
APP_PID=$!

# Check what's happening
ps aux | grep dmtcp
ls /tmp/dmtcp-$USER@$(hostname)/

# Initiate checkpoint
./bin/dmtcp_command --checkpoint

# Restart
./bin/dmtcp_restart ckpt_dmtcp1_*.dmtcp
```

### 3. Make a Simple Change

Let's add a debug message to understand the flow:

```bash
# Edit the worker file
vim src/dmtcpworker.cpp
```

Add this line in the constructor:
```cpp
JTRACE("DMTCP Worker initialized") (getpid()) (getppid());
```

```bash
# Rebuild
make -j$(nproc)

# Test your change
./bin/dmtcp_launch test/dmtcp1

# Check the log
cat /tmp/dmtcp-$USER@$(hostname)/jassertlog.* | grep "initialized"
```

## Development Workflow

### 1. Create a Branch

```bash
# Create a feature branch
git checkout -b feature/my-first-contribution

# Make your changes
# ... edit files ...

# Check what changed
git status
git diff
```

### 2. Test Your Changes

```bash
# Build with your changes
make clean && make -j$(nproc)

# Run relevant tests
make check-dmtcp1
make check-dmtcp2

# Test manually if needed
./bin/dmtcp_launch test/dmtcp1
./bin/dmtcp_command --checkpoint
./bin/dmtcp_restart ckpt_dmtcp1_*.dmtcp
```

### 3. Commit Your Changes

```bash
# Add files
git add src/dmtcpworker.cpp

# Commit with clear message
git commit -m "Add debug trace to DMTCP worker initialization

This helps new contributors understand when the worker
process starts and its relationship to the parent process."

# Push to your fork
git push origin feature/my-first-contribution
```

## Understanding the Architecture

### Two-Layer Design

```
┌─────────────────────────────────────┐
│           DMTCP Layer              │  ← Distributed coordination
│  ┌─────────────────────────────┐   │
│  │      Coordinator            │   │
│  │   (dmtcp_coordinator)       │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│            MTCP Layer               │  ← Local checkpointing
│  ┌─────────────────────────────┐   │
│  │    Memory Checkpointing     │   │
│  │     Thread Management       │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### Key Concepts to Understand

1. **Coordinator**: Central authority managing checkpoint/restart
2. **Worker Process**: Application process with `libdmtcp.so` injected
3. **Checkpoint Thread**: Special thread managing checkpoint/restart
4. **Wrapper Functions**: Intercept system calls via `LD_PRELOAD`
5. **Virtualization Tables**: Maintain mappings for PIDs, TIDs, FDs

## Common Development Tasks

### Adding a New Wrapper Function

```cpp
// In appropriate wrapper file (e.g., filewrappers.cpp)
extern "C" int my_new_function(const char *arg) {
  static int (*real_func) (const char *) = NULL;
  if (real_func == NULL) {
    real_func = (int (*)(const char *)) dmtcp_dlsym(RTLD_NEXT, "my_new_function");
  }
  
  JTRACE("my_new_function called") (arg);
  
  int result = real_func(arg);
  
  // Post-processing if needed
  
  return result;
}
```

### Adding a New Plugin

```bash
# Create plugin directory
mkdir test/plugin/myplugin
cd test/plugin/myplugin

# Create basic plugin
cat > myplugin.cpp << 'EOF'
#include <dmtcp/plugin.h>

static void myplugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data) {
  switch (event) {
    case DMTCP_EVENT_INIT:
      JTRACE("MyPlugin initialized");
      break;
    case DMTCP_EVENT_PRECHECKPOINT:
      JTRACE("MyPlugin pre-checkpoint");
      break;
    default:
      break;
  }
}

DmtcpPluginDescriptor_t myplugin_plugin = {
  DMTCP_PLUGIN_API_VERSION,
  "MyPlugin",
  "1.0.0",
  "Your Name",
  "your@email.com",
  "My first DMTCP plugin",
  myplugin_event_hook
};

DMTCP_DECL_PLUGIN(myplugin_plugin);
EOF

# Create Makefile
cat > Makefile << 'EOF'
lib_LTLIBRARIES = libmyplugin.la
libmyplugin_la_SOURCES = myplugin.cpp
libmyplugin_la_CPPFLAGS = $(DMTCP_INCLUDES)
libmyplugin_la_LDFLAGS = -avoid-version -module
EOF

# Build and test
cd ../../..
./configure
make -j$(nproc)
./bin/dmtcp_launch --with-plugin test/plugin/myplugin/.libs/libmyplugin.so test/dmtcp1
```

### Debugging a Checkpoint Issue

```bash
# Enable detailed logging
export DMTCP_LOG_TIMESTAMP=1
export DMTCP_LOG_COMPACT=0

# Run with debugging
./bin/dmtcp_launch test/dmtcp1 &
APP_PID=$!

# Monitor logs
tail -f /tmp/dmtcp-$USER@$(hostname)/jassertlog.$APP_PID

# Initiate checkpoint
./bin/dmtcp_command --checkpoint

# If it hangs, attach GDB
gdb -p $APP_PID
(gdb) source util/gdb-dmtcp-utils.py
(gdb) info threads
(gdb) thread apply all bt
```

## Testing Your Changes

### Running Tests

```bash
# Quick test
make check-dmtcp1

# Full test suite
make check

# Specific test categories
make check-plugin
make check-mpi  # If MPI enabled

# Performance tests
make check-performance
```

### Writing Tests

```bash
# Create a simple test
cat > test/mytest.c << 'EOF'
#include <stdio.h>
#include <unistd.h>

int main() {
  printf("Hello from my test!\n");
  printf("PID: %d\n", getpid());
  sleep(1);
  printf("Done!\n");
  return 0;
}
EOF

# Compile test
gcc -o test/mytest test/mytest.c

# Test with DMTCP
./bin/dmtcp_launch ./test/mytest
./bin/dmtcp_command --checkpoint
./bin/dmtcp_restart ckpt_mytest_*.dmtcp
```

## Code Style Guidelines

### C++ Style

```cpp
// Use DMTCP naming conventions
class DmtcpWorker {
public:
  void suspendUserThreads();
  
private:
  int _checkpointThread;
};

// Use JALIB utilities for logging
JTRACE("Message") (variable1) (variable2);
JNOTE("Important event") (context);
JWARNING("Warning") (warning_context);
JASSERT(condition) (error_context);
```

### Memory Management

```cpp
// Use JALIB allocator for DMTCP internal memory
void *ptr = JALLOC_HELPER_MALLOC(size);
JALLOC_HELPER_FREE(ptr);

// Never use malloc/free in checkpoint-critical code
// Use new/delete for C++ objects
MyClass *obj = new MyClass();
delete obj;
```

### Error Handling

```cpp
// Use JASSERT for fatal errors
JASSERT(fd >= 0) (fd) (errno) .Text("Failed to open file");

// Use JWARNING for recoverable errors
JWARNING(result == 0) (result) .Text("Operation failed, continuing");

// Always check return values
int result = some_function();
if (result == -1) {
  JTRACE("Function failed") (errno);
  return -1;
}
```

## Getting Help

### Documentation

```bash
# Read existing documentation
less doc/debugging-dmtcp.txt
less doc/pid-tid-virtualization.txt
less doc/connections.txt

# View plugin tutorial
evince doc/plugin-tutorial.pdf  # If evince available
```

### Community Resources

- **GitHub Issues**: https://github.com/dmtcp/dmtcp/issues
- **Mailing List**: dmtcp-forum@lists.sourceforge.net
- **Website**: https://dmtcp.org

### Asking Good Questions

When asking for help, include:

1. **What you're trying to accomplish**
2. **What you've tried so far**
3. **Error messages or logs**
4. **System information**: `uname -a`, `gcc --version`
5. **DMTCP version**: `./bin/dmtcp_launch --version`

Example:
```
Subject: Help with adding new socket wrapper

I'm trying to add a wrapper for the sendmsg() system call to track
message sizes. I've added the wrapper function in socketwrappers.cpp
similar to the send() wrapper, but when I run the test, I get a
segmentation fault.

Here's my wrapper function:
[... code ...]

The error occurs when I run:
./bin/dmtcp_launch test/client-server

System: Ubuntu 20.04, GCC 9.3.0
DMTCP: version 2.6.0

Log file shows:
[... relevant log entries ...]
```

## Next Steps

### Suggested First Contributions

1. **Fix a simple bug**: Look for issues labeled "good first issue"
2. **Add a test**: Improve test coverage for existing functionality
3. **Improve documentation**: Fix typos or clarify explanations
4. **Add a small feature**: Implement a requested enhancement

### Advanced Topics

Once you're comfortable with the basics:

1. **Plugin Development**: Create complex plugins with state management
2. **Performance Optimization**: Improve checkpoint/restart performance
3. **MPI Integration**: Work on MANA for MPI applications
4. **Architecture Changes**: Propose significant architectural improvements

### Staying Involved

- **Watch the repository**: Stay updated on changes
- **Review pull requests**: Learn from other contributors
- **Join discussions**: Participate in design discussions
- **Attend conferences**: Present your DMTCP work

---

Welcome to the DMTCP community! We're glad to have you contributing to transparent checkpointing technology.