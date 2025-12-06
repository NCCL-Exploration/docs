# CRAC Documentation

Welcome to the CRAC (Checkpoint-Restart Architecture for CUDA) documentation. CRAC is a DMTCP plugin that enables transparent checkpointing of CUDA applications using NVIDIA's native checkpoint/restart APIs introduced in CUDA 12.4.

## Documentation Structure

### Getting Started
- [Installation](getting-started/installation.md) - Prerequisites and build instructions
- [Quick Start](getting-started/quick-start.md) - Minimal example and basic usage

### Architecture
- [Overview](architecture/overview.md) - High-level architecture and component relationships
- [Checkpoint Flow](architecture/checkpoint-flow.md) - Step-by-step checkpoint process walkthrough
- [Restart Flow](architecture/restart-flow.md) - Step-by-step restart process walkthrough
- [Data Structures](architecture/data-structures.md) - Key data structures and state management

### Contributor Guide
- [Development Setup](contributor-guide/development-setup.md) - Setting up a development environment
- [Code Walkthrough](contributor-guide/code-walkthrough.md) - File-by-file explanation of the codebase
- [Adding Features](contributor-guide/adding-features.md) - How to extend CRAC functionality
- [Testing](contributor-guide/testing.md) - Test suite overview and testing guidelines

### Reference
- [DMTCP Plugin API](reference/dmtcp-plugin-api.md) - Quick reference for DMTCP plugin APIs
- [CUDA Checkpoint API](reference/cuda-checkpoint-api.md) - Quick reference for CUDA checkpoint APIs
- [Glossary](reference/glossary.md) - Key terms and definitions

### Other
- [Troubleshooting](troubleshooting.md) - Common errors and solutions
- [Contributing](../CONTRIBUTING.md) - How to contribute to the project

## Prerequisites for Reading the Docs

To get the most out of this documentation, you should have:

- Basic understanding of CUDA programming concepts (kernels, device memory, streams)
- Familiarity with Linux systems programming (file descriptors, memory mapping)
- Some knowledge of checkpoint/restart concepts (helpful but not required)
- CUDA 12.4+ and NVIDIA driver 550+ installed on your system

## Quick Navigation

| If you want to... | Start here... |
|-------------------|---------------|
| Build and run CRAC for the first time | [Installation](getting-started/installation.md) → [Quick Start](getting-started/quick-start.md) |
| Understand how CRAC works internally | [Architecture Overview](architecture/overview.md) |
| Contribute code to CRAC | [Development Setup](contributor-guide/development-setup.md) → [Code Walkthrough](contributor-guide/code-walkthrough.md) |
| Debug checkpoint/restart issues | [Troubleshooting](troubleshooting.md) |
| Learn about the APIs CRAC uses | [DMTCP Plugin API](reference/dmtcp-plugin-api.md) → [CUDA Checkpoint API](reference/cuda-checkpoint-api.md) |

## Project Repository

The main repository is available at: https://github.com/xu-yao0127/crac

## Key Technologies

- **DMTCP**: User-space transparent checkpointing tool for Linux applications
- **CUDA Driver API Checkpoint Functions**: Native NVIDIA checkpoint/restart APIs (CUDA 12.4+)
- **DMTCP Plugin System**: Event hooks and wrapper functions for extending DMTCP

## See Also

- [DMTCP Official Documentation](https://dmtcp.sourceforge.io/)
- [NVIDIA CUDA Checkpoint/Restart Documentation](https://docs.nvidia.com/cuda/cuda-checkpoint/)
- [CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)