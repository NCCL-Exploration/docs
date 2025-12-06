# DMTCP Contributor Documentation

Welcome to the DMTCP contributor documentation! This guide will help you understand the architecture, implementation details, and development practices for contributing to DMTCP (Distributed MultiThreaded CheckPointing).

## Table of Contents

1. [Quick Start Guide](CONTRIBUTOR-QUICKSTART.md)
2. [Architecture Overview](CONTRIBUTOR-ARCHITECTURE.md)
3. [Checkpoint-Restart Flow](CONTRIBUTOR-CHECKPOINT-FLOW.md)
4. [Plugin Development](CONTRIBUTOR-PLUGINS.md)
5. [Wrapper Function System](CONTRIBUTOR-WRAPPERS.md)
6. [Debugging and Troubleshooting](CONTRIBUTOR-DEBUGGING.md)

## About DMTCP

DMTCP is a transparent user-space checkpointing system that enables applications to be:
- **Frozen** at any point in execution
- **Saved** to disk as checkpoint images
- **Restarted** potentially on different machines

All of this happens **without kernel modifications** or **application source changes**.

## Key Design Principles

- **User-space operation**: No kernel patches required
- **Transparency**: Applications run unmodified
- **Distributed coordination**: Multiple processes checkpointed together
- **Plugin extensibility**: Modular architecture for custom behaviors
- **Libc independence**: Safe operation during checkpoint phases

## Repository Structure

```
dmtcp/
├── src/                    # Core DMTCP implementation
│   ├── mtcp/               # Low-level single-process checkpointing
│   ├── plugin/             # Internal required plugins
│   ├── dmtcp_coordinator.cpp   # Central coordinator
│   ├── dmtcp_launch.cpp        # Process launcher
│   ├── coordinatorapi.cpp      # Client-coordinator protocol
│   └── trampolines.*           # Function interposition
├── jalib/                  # Custom utility library (DMTCP-safe)
├── plugin/                 # Optional extension plugins
├── include/                # Public API headers
├── contrib/                # Contributed components
├── test/                   # Test suite and examples
└── doc/                    # Additional technical documentation
```

## Getting Started

1. **Read the Quick Start Guide** - Basic setup and first contribution
2. **Study the Architecture Overview** - Understand the two-layer design
3. **Learn the Checkpoint Flow** - Master the six-barrier protocol
4. **Explore Plugin Development** - Extend DMTCP functionality
5. **Master Debugging** - Effective troubleshooting techniques

## Core Concepts

### Two-Layer Architecture
- **DMTCP Layer**: Distributed coordination, resource management
- **MTCP Layer**: Single-process memory checkpointing, thread handling

### Six-Barrier Checkpoint Protocol
1. RUNNING → SUSPENDED
2. FD_LEADER_ELECTION  
3. DRAINED
4. CHECKPOINTED
5. REFILLED
6. RUNNING

### Virtualization Tables
- **PID/TID virtualization**: Maintain original process/thread IDs
- **File descriptor virtualization**: Preserve descriptor relationships
- **Connection IDs**: Globally unique socket identification

## Development Environment

### Build System
- Uses GNU Autotools (`configure.ac`, `Makefile.am`)
- Key binaries: `dmtcp_coordinator`, `dmtcp_launch`, `dmtcp_restart`, `libdmtcp.so`

### Essential Tools
- **GDB with DMTCP utils**: `util/gdb-dmtcp-utils.py`
- **Logging**: `./configure --enable-logging`
- **Test framework**: `test/autotest.py`

## Contributing Guidelines

### Code Style
- Follow existing conventions in each module
- Use JALIB utilities instead of standard library during checkpoint
- Maintain thread safety in all wrapper functions
- Document complex algorithms with inline comments

### Testing
- Add tests to `test/` directory
- Use existing test patterns as templates
- Test both checkpoint and restart scenarios
- Verify plugin compatibility

### Documentation
- Update relevant sections in contributor docs
- Add examples to `test/plugin/` for new features
- Maintain technical accuracy in `doc/` files

## Community Resources

- **Main repository**: https://github.com/dmtcp/dmtcp
- **Issues and discussions**: GitHub Issues
- **Technical papers**: See architecture documentation
- **Plugin examples**: `test/plugin/example/`

## Need Help?

- Check the debugging guide for common issues
- Review existing plugin implementations
- Study the technical documentation in `doc/`
- Examine test cases for usage patterns

---

This documentation is maintained alongside the codebase. If you find inaccuracies or missing information, please contribute improvements!