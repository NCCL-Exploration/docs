# Contributing to DMTCP

Thank you for your interest in contributing to DMTCP (Distributed MultiThreaded CheckPointing)! This guide will help you get started with contributing to this transparent checkpointing system.

## Table of Contents

1. [Getting Started](#getting-started)
2. [Development Setup](#development-setup)
3. [Architecture Overview](#architecture-overview)
4. [Code Organization](#code-organization)
5. [Development Workflow](#development-workflow)
6. [Testing](#testing)
7. [Code Style](#code-style)
8. [Plugin Development](#plugin-development)
9. [Debugging](#debugging)
10. [Submitting Changes](#submitting-changes)

## Getting Started

### Prerequisites

- Linux system (x86_64, i386, arm64, or risc-v)
- GCC/G++ compiler with C++14 support
- Make and Autoconf/Automake tools
- Git

### First Time Setup

```bash
# Clone the repository
git clone https://github.com/dmtcp/dmtcp.git
cd dmtcp

# Configure the build
./configure

# Build DMTCP
make

# Run tests to verify your setup
make check
```

### Understanding the Basics

DMTCP provides transparent checkpointing for distributed applications. Before diving into code development:

1. **Read the core documentation**: Start with [QUICK-START.md](../QUICK-START.md) and [README.md](../README.md)
2. **Try the basic workflow**: Run a simple application under DMTCP control
3. **Explore the test suite**: Look at `test/` directory for examples

## Development Setup

### Build Configuration

DMTCP supports several build options:

```bash
# Enable debug mode (recommended for development)
./configure --enable-debug

# Support 32-bit applications on 64-bit system
./configure --enable-m32

# Build with unique checkpoint filenames
./configure --enable-unique-checkpoint-filenames

# See all options
./configure --help
```

### Development Tools

- **Debugging**: Use `--enable-debug` for detailed logs in `$DMTCP_TMPDIR/dmtcp-$USER@$HOST/jassertlog.*`
- **GDB integration**: Use `util/gdb-add-libdmtcp-symbol-file.py` for debugging
- **Code style**: Use `util/dmtcp-style.py` to check coding standards

## Architecture Overview

DMTCP consists of two main layers:

### DMTCP Layer
- **Coordinator**: Central synchronization point (`src/dmtcp_coordinator.cpp`)
- **Launch/Restart**: Process management (`src/dmtcp_launch.cpp`, `src/dmtcp_restart.cpp`)
- **Wrappers**: System call interception via LD_PRELOAD
- **Plugin System**: Extensibility framework

### MTCP Layer (Multi-Threaded CheckPointing)
- **Core checkpointing**: Single-process checkpoint/restart (`src/mtcp/`)
- **Memory management**: Process image restoration
- **Thread handling**: Context save/restore

### Key Concepts

1. **Seven-Stage Checkpoint Algorithm**: RUNNING → SUSPENDED → FD_LEADER_ELECTION → DRAINED → CHECKPOINTED → REFILLED → RUNNING
2. **Coordinator-Client Protocol**: TCP-based communication (default port 7779)
3. **Virtualization**: PIDs, file descriptors, sockets, MPI identifiers
4. **Plugin Architecture**: Event hooks, wrapper functions, publish/subscribe service

## Code Organization

```
src/
├── dmtcp_coordinator.cpp    # Central coordinator
├── dmtcp_launch.cpp         # Process launcher
├── dmtcp_restart.cpp        # Process restarter
├── threadlist.cpp           # Thread management
├── plugin/                  # Internal plugins
└── mtcp/                    # MTCP layer
    ├── mtcp_restart.c       # Core restart logic
    ├── mtcp_header.h        # MTCP data structures
    └── mtcp_sys.h           # System interfaces

include/
├── dmtcp.h                  # Plugin API
└── dmtcp/                   # Internal headers

jalib/                       # Utility library
test/plugin/                 # Plugin examples
plugin/                      # Optional plugins
```

## Development Workflow

### 1. Choose Your Contribution Type

- **Bug fixes**: Start with existing issues labeled "bug"
- **New features**: Discuss in issues first
- **Documentation**: Always welcome
- **Plugin development**: See [Plugin Development](#plugin-development)

### 2. Create a Branch

```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/issue-number-description
```

### 3. Make Changes

- Follow existing code patterns
- Add tests for new functionality
- Update documentation as needed

### 4. Test Your Changes

```bash
# Build with your changes
make

# Run the full test suite
make check

# Test specific components
cd test && ./autotest.py -t test_name
```

## Testing

### Test Structure

- `test/`: Main test suite using Python framework
- `test/plugin/`: Plugin examples and tests
- `make check`: Run all tests

### Running Tests

```bash
# Run all tests
make check

# Run specific test
./test/autotest.py -t dmtcp1

# Run with verbose output
./test/autotest.py -v -t test_name

# Run performance tests
./test/autotest.py -p
```

### Writing Tests

1. Place test programs in `test/`
2. Follow naming convention: `test_*.c` or `test_*.cpp`
3. Add to `test/Makefile.am`
4. Use DMTCP API functions for checkpoint control

## Code Style

DMTCP follows these conventions:

### C/C++ Style
- 2-space indentation
- K&R brace style
- 80-character line limit
- No trailing whitespace

### Naming Conventions
- **Functions**: `camelCase()` for public, `camelCase()` for private
- **Variables**: `camelCase` for local, `camelCase_` for member
- **Constants**: `ALL_CAPS_WITH_UNDERSCORES`
- **Files**: `lowercase-with-dashes.cpp`

### Comments
- Use `//` for single-line comments
- Use `/* */` for multi-line comments
- Document public APIs in header files

### Code Style Check

```bash
# Check code style
python util/dmtcp-style.py src/file.cpp

# Auto-fix some issues
python util/dmtcp-style.py --fix src/file.cpp
```

## Plugin Development

### Plugin Types

1. **Event Hooks**: React to DMTCP events
2. **Wrapper Functions**: Intercept system calls
3. **Publish/Subscribe**: Share data between processes

### Basic Plugin Structure

```c
#include "dmtcp.h"

// Event hook function
static void
myPlugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data)
{
  switch (event) {
    case DMTCP_EVENT_INIT:
      // Initialization
      break;
    case DMTCP_EVENT_PRECHECKPOINT:
      // Before checkpoint
      break;
    case DMTCP_EVENT_RESTART:
      // After restart
      break;
    default:
      break;
  }
  
  // Call next plugin in chain
  DMTCP_NEXT_EVENT_HOOK(event, data);
}

// Wrapper function example
static int
mySocket(int domain, int type, int protocol)
{
  // Pre-processing
  int sockfd = NEXT_FNC(socket)(domain, type, protocol);
  // Post-processing
  return sockfd;
}

// Plugin descriptor
static DmtcpPluginDescriptor_t myPlugin = {
  DMTCP_PLUGIN_API_VERSION,
  "MyPlugin",
  "1.0",
  "Author Name",
  "author@example.com",
  "My DMTCP plugin",
  myPlugin_event_hook
};

// Register plugin
DMTCP_REGISTER_PLUGIN(myPlugin);
```

### Plugin Examples

See `test/plugin/` for complete examples:
- `example/`: Basic event handling
- `sleep1/`, `sleep2/`: Wrapper functions
- `applic-delayed-ckpt/`: Application-controlled checkpointing

## Debugging

### Debug Mode

```bash
# Configure with debug support
./configure --enable-debug
make

# Run with debug output
DMTCP_DEBUG=1 dmtcp_launch ./your_app
```

### Debug Files

Debug logs are written to:
```
$DMTCP_TMPDIR/dmtcp-$USER@$HOST/jassertlog.*
```

### GDB Integration

```bash
# Attach to a restarted process
gdb ./your_app `pgrep -n MTCP`

# Load DMTCP symbols
python util/gdb-add-libdmtcp-symbol-file.py
```

### Common Debugging Scenarios

1. **Checkpoint failures**: Check coordinator logs and jassert files
2. **Restart issues**: Verify checkpoint file integrity
3. **Plugin problems**: Use event hooks to trace execution
4. **Memory issues**: Use Valgrind with DMTCP

## Submitting Changes

### Before Submitting

1. **Code Review**: Self-review your changes
2. **Testing**: Ensure all tests pass
3. **Documentation**: Update relevant documentation
4. **Style Check**: Run code style checks

### Pull Request Process

1. **Create Pull Request**: Use descriptive title and detailed description
2. **Link Issues**: Reference related issue numbers
3. **Review Process**: Address reviewer feedback promptly
4. **Testing**: Ensure CI passes

### Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] All tests pass
- [ ] Added new tests for new functionality
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
```

## Getting Help

### Resources

- **Documentation**: [doc/](../doc/) directory
- **Plugin Tutorial**: `doc/plugin-tutorial.pdf`
- **Mailing List**: dmtcp-forum@lists.sourceforge.net
- **GitHub Issues**: [github.com/dmtcp/dmtcp/issues](https://github.com/dmtcp/dmtcp/issues)

### Community

- **Forum**: dmtcp-forum@lists.sourceforge.net
- **GitHub Discussions**: Use GitHub Discussions for questions
- **IRC**: #dmtcp on OFTC (if available)

## Architecture Decision Records (ADRs)

Key design decisions documented in `doc/adr/`:

- **ADR-001**: LD_PRELOAD-based wrapper mechanism
- **ADR-002**: Coordinator-based architecture
- **ADR-003**: Plugin system design
- **ADR-004**: Seven-stage checkpoint algorithm

## Glossary

- **Coordinator**: Central process managing checkpoint synchronization
- **Worker**: Application process under DMTCP control
- **Plugin**: Extension module for DMTCP functionality
- **Wrapper**: Interposed function call
- **Checkpoint Image**: Serialized process state (.dmtcp file)
- **Library Interposition**: LD_PRELOAD-based function interception
- **Virtual PID**: Abstracted process identifier
- **MTCP**: Multi-Threaded CheckPointing layer

Thank you for contributing to DMTCP! Your contributions help make transparent checkpointing accessible to everyone.