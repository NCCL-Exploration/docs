# Contributing to CRAC

Thank you for your interest in contributing to CRAC (Checkpoint-Restart Architecture for CUDA)! This guide covers how to contribute to the project.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Pull Request Process](#pull-request-process)
- [Code Style Guidelines](#code-style-guidelines)
- [Testing Requirements](#testing-requirements)
- [Documentation](#documentation)
- [Issue Reporting](#issue-reporting)
- [Community](#community)

## Code of Conduct

### Our Pledge

We are committed to making participation in this project a harassment-free experience for everyone, regardless of:

- Age, body size, disability, ethnicity, gender identity and expression
- Level of experience, education, socio-economic status, nationality, personal appearance
- Race, religion, or sexual identity and orientation

### Our Standards

**Positive behavior includes**:
- Using welcoming and inclusive language
- Being respectful of differing viewpoints and experiences
- Gracefully accepting constructive criticism
- Focusing on what is best for the community
- Showing empathy towards other community members

**Unacceptable behavior includes**:
- The use of sexualized language or imagery
- Trolling, insulting/derogatory comments, or personal/political attacks
- Public or private harassment
- Publishing others' private information (doxing)
- Any other conduct which could reasonably be considered inappropriate

### Enforcement

Project maintainers have the right and responsibility to remove, edit, or reject comments, commits, code, wiki edits, issues, and other contributions that are not aligned to this Code of Conduct.

## Getting Started

### Prerequisites

Before contributing, ensure you have:

- **CUDA 12.4+** and NVIDIA driver 550+
- **DMTCP 3.0+** built from source
- **Linux development environment** with build tools
- **Git** configured with your name and email

### Initial Setup

1. **Fork the Repository**:
   ```bash
   # Fork https://github.com/xu-yao0127/crac on GitHub
   # Then clone your fork
   git clone https://github.com/yourusername/crac.git
   cd crac
   ```

2. **Add Upstream Remote**:
   ```bash
   git remote add upstream https://github.com/xu-yao0127/crac.git
   git fetch upstream
   ```

3. **Set Up Development Environment**:
   ```bash
   # Follow the Development Setup guide
   # See docs/contributor-guide/development-setup.md
   make DMTCP_ROOT=$DMTCP_ROOT
   ```

## Development Workflow

### 1. Create a Branch

```bash
# Sync with main branch
git checkout main
git pull upstream main

# Create feature branch
git checkout -b feature/your-feature-name

# Or bugfix branch
git checkout -b bugfix/issue-number-description
```

### 2. Make Changes

- Follow the [Code Style Guidelines](#code-style-guidelines)
- Add tests for new functionality
- Update documentation as needed
- Make small, focused commits

### 3. Test Your Changes

```bash
# Build the project
make clean && make DMTCP_ROOT=$DMTCP_ROOT

# Run tests
make check DMTCP_ROOT=$DMTCP_ROOT

# Test manually with your scenario
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/your-test
```

### 4. Commit Your Changes

```bash
# Stage changes
git add .

# Commit with clear message
git commit -m "Add feature: brief description

- Detailed description of changes
- How it works
- Testing performed
- References to issues if any

Fixes #123"
```

## Pull Request Process

### Before Submitting

1. **Update Your Branch**:
   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

2. **Run Full Test Suite**:
   ```bash
   make clean
   make DMTCP_ROOT=$DMTCP_ROOT
   make check DMTCP_ROOT=$DMTCP_ROOT
   ```

3. **Check Code Style**:
   ```bash
   # Use clang-format if available
   clang-format -i crac.cpp
   
   # Check for common issues
   cppcheck --enable=all crac.cpp
   ```

### Submitting Pull Request

1. **Push to Your Fork**:
   ```bash
   git push origin feature/your-feature-name
   ```

2. **Create Pull Request**:
   - Go to your fork on GitHub
   - Click "New Pull Request"
   - Select your feature branch
   - Target the `main` branch of the upstream repository

3. **Fill Pull Request Template**:
   ```markdown
   ## Description
   Brief description of changes

   ## Type of Change
   - [ ] Bug fix
   - [ ] New feature
   - [ ] Breaking change
   - [ ] Documentation update

   ## Testing
   - [ ] Unit tests pass
   - [ ] Integration tests pass
   - [ ] Manual testing performed

   ## Checklist
   - [ ] Code follows style guidelines
   - [ ] Self-review completed
   - [ ] Documentation updated
   - [ ] Tests added/updated
   ```

### Pull Request Review Process

1. **Automated Checks**:
   - Code style verification
   - Build status
   - Test suite results

2. **Code Review**:
   - Maintainer review within 3-5 business days
   - Feedback provided via GitHub comments
   - Address review comments promptly

3. **Approval and Merge**:
   - Requires at least one maintainer approval
   - Merge after all comments resolved
   - Squash merge for clean history

## Code Style Guidelines

### C/C++ Style

#### Naming Conventions
```cpp
// Functions: snake_case
void checkpoint_gpu();
int inspect_pipes();

// Variables: snake_case
int num_fds_found;
CUprocessState cuda_state;

// Constants: UPPER_CASE
#define MAX_PIPE_FDS 1024
#define PATH_MAX 4096

// Types: snake_case with _t suffix
typedef struct {
    int fd;
    unsigned long pipe_inode;
} pipe_info_t;

// Enum values: UPPER_CASE
typedef enum {
    PIPE_READ = O_RDONLY,
    PIPE_WRITE = O_WRONLY
} pipe_rw_t;
```

#### Formatting
```cpp
// Indentation: 2 spaces (no tabs)
if (condition) {
  do_something();
} else {
  do_something_else();
}

// Function definitions
static void checkpoint_gpu() {
  // Implementation
}

// Line length: Maximum 80-100 characters
long_line_that_should_be_broken_into_multiple_lines_for_readability =
    function_call(with, many, parameters);
```

#### Comments
```cpp
/**
 * Brief description of function purpose
 * 
 * @param param1 Description of first parameter
 * @param param2 Description of second parameter
 * @return Description of return value
 */
static int my_function(int param1, char *param2) {
  // Inline comments for complex logic
  if (complex_condition) {
    // Explain why this is necessary
    workaround_for_issue();
  }
}
```

### CUDA Style

#### Kernel Naming
```cuda
// Kernels: descriptive names with _kernel suffix
__global__ void vector_add_kernel(float *a, float *b, float *c, int n) {
  int idx = blockIdx.x * blockDim.x + threadIdx.x;
  if (idx < n) {
    c[idx] = a[idx] + b[idx];
  }
}
```

#### Error Handling
```cpp
// Always check CUDA errors
CUresult ret = cuSomeFunction();
if (ret != CUDA_SUCCESS) {
  const char *error_str;
  cuGetErrorString(ret, &error_str);
  fprintf(stderr, "CUDA Error: %s\n", error_str);
  return -1;
}
```

### Makefile Style

```makefile
# Use variables for common paths
DMTCP_ROOT ?= ../../
CUDA_INCLUDE = -I/usr/local/cuda/include

# Use pattern rules
%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c -o $@ $<

# Keep targets organized
.PHONY: clean check test
```

## Testing Requirements

### All Contributions Must Include Tests

#### New Features
- Add test case to `test/` directory
- Test both success and failure scenarios
- Verify with different input sizes

#### Bug Fixes
- Add regression test that fails before fix
- Verify test passes after fix
- Test edge cases related to the bug

#### Performance Changes
- Include benchmark tests
- Compare performance before/after
- Document expected performance impact

### Test Categories

#### Unit Tests
```cpp
// Test individual functions
void test_inspect_pipes() {
  // Setup test pipes
  // Call inspect_pipes()
  // Verify results
}
```

#### Integration Tests
```bash
# Test complete checkpoint/restart cycle
#!/bin/bash
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/program &
sleep 2
$DMTCP_ROOT/bin/dmtcp_command --checkpoint
# ... restart and verify
```

#### Performance Tests
```bash
# Measure checkpoint/restart times
time $DMTCP_ROOT/bin/dmtcp_command --checkpoint
```

### Running Tests

```bash
# Quick test
make check DMTCP_ROOT=$DMTCP_ROOT

# Full test suite
make test-all DMTCP_ROOT=$DMTCP_ROOT

# Performance benchmarks
make benchmark DMTCP_ROOT=$DMTCP_ROOT
```

## Documentation

### When to Update Documentation

- **New Features**: Update relevant documentation sections
- **API Changes**: Update API reference documents
- **Configuration Changes**: Update installation/setup guides
- **Behavior Changes**: Update troubleshooting guide

### Documentation Types

#### Code Comments
- Add function-level documentation
- Comment complex algorithms
- Explain non-obvious design decisions

#### User Documentation
- Update `docs/getting-started/` for user-facing changes
- Update `docs/troubleshooting.md` for known issues
- Update `docs/reference/` for API changes

#### Contributor Documentation
- Update `docs/contributor-guide/` for development changes
- Update this CONTRIBUTING.md for process changes

### Documentation Style

- Use clear, concise language
- Include code examples
- Add cross-references between documents
- Keep examples up-to-date

## Issue Reporting

### Bug Reports

Use the [GitHub issue tracker](https://github.com/xu-yao0127/crac/issues) with:

```markdown
## Bug Description
Clear description of the issue

## Steps to Reproduce
1. Step one
2. Step two
3. Step three

## Expected Behavior
What should happen

## Actual Behavior
What actually happens

## Environment
- OS: Ubuntu 20.04
- CUDA: 12.4
- DMTCP: 3.0.0
- GPU: RTX 3080

## Additional Information
Logs, screenshots, or other relevant details
```

### Feature Requests

```markdown
## Feature Description
Clear description of desired feature

## Use Case
Why this feature is needed

## Proposed Solution
How you think it should work

## Alternatives Considered
Other approaches you've thought about

## Additional Context
Any other relevant information
```

## Community

### Communication Channels

- **GitHub Issues**: Bug reports and feature requests
- **GitHub Discussions**: General questions and discussions
- **Email**: `xu.yao1@northeastern.edu` for private matters

### Getting Help

1. **Check Documentation**:
   - [Getting Started](docs/getting-started/)
   - [Troubleshooting](docs/troubleshooting.md)
   - [FAQ](docs/reference/glossary.md)

2. **Search Existing Issues**:
   - Check if your issue has been reported
   - Look for related discussions

3. **Ask Questions**:
   - Use GitHub Discussions for questions
   - Provide context and what you've tried

### Recognition

Contributors are recognized in:
- `AUTHORS` file for significant contributions
- Release notes for new features and bug fixes
- Project documentation for special contributions

## Release Process

### Version Numbers

- Follow [Semantic Versioning](https://semver.org/)
- Format: MAJOR.MINOR.PATCH
- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes (backward compatible)

### Release Checklist

- [ ] All tests pass
- [ ] Documentation updated
- [ ] CHANGELOG.md updated
- [ ] Version numbers updated
- [ ] Tag created
- [ ] Release published

## Licensing

By contributing to CRAC, you agree that your contributions will be licensed under the same license as the project (currently the same as DMTCP - LGPL).

## Questions?

If you have questions about contributing:

1. Check existing [GitHub Issues](https://github.com/xu-yao0127/crac/issues)
2. Read the [Documentation](docs/)
3. Start a [GitHub Discussion](https://github.com/xu-yao0127/crac/discussions)
4. Contact the maintainers at `xu.yao1@northeastern.edu`

---

Thank you for contributing to CRAC! Your contributions help make CUDA checkpointing more reliable and accessible to the community.