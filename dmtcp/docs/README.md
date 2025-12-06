# DMTCP Documentation Index

This directory contains comprehensive documentation for DMTCP contributors and users.

## Documentation Structure

### Contributor Documentation
- **[CONTRIBUTING.md](../CONTRIBUTING.md)** - Main contributor guide
- **[contributor-guide/](./contributor-guide/)** - Detailed development guides
  - [adding-wrapper.md](./contributor-guide/adding-wrapper.md) - Adding system call wrappers
  - [plugin-development.md](./contributor-guide/plugin-development.md) - Plugin development guide

### Architecture Documentation
- **[architecture/](./architecture/)** - System architecture and design
  - [overview.md](./architecture/overview.md) - High-level architecture overview
  - [checkpoint-flow.md](./architecture/checkpoint-flow.md) - Detailed checkpoint process

### Architecture Decisions
- **[adr/](./adr/)** - Architecture Decision Records
  - [README.md](./adr/README.md) - ADR index and process

### Legacy Documentation
- **[../doc/](../doc/)** - Original documentation (preserved for reference)
  - [plugin-tutorial.pdf](../doc/plugin-tutorial.pdf) - Plugin tutorial (PDF)
  - Various text files covering specific topics

## Quick Links

### For New Contributors
1. Read [CONTRIBUTING.md](../CONTRIBUTING.md) for setup instructions
2. Study [architecture/overview.md](./architecture/overview.md) for system understanding
3. Follow [contributor-guide/adding-wrapper.md](./contributor-guide/adding-wrapper.md) for code patterns

### For Plugin Developers
1. Start with [contributor-guide/plugin-development.md](./contributor-guide/plugin-development.md)
2. Review [../doc/plugin-tutorial.pdf](../doc/plugin-tutorial.pdf) for examples
3. Study [../test/plugin/](../test/plugin/) for sample implementations

### For Architecture Understanding
1. Read [architecture/overview.md](./architecture/overview.md) for big picture
2. Study [architecture/checkpoint-flow.md](./architecture/checkpoint-flow.md) for process details
3. Review [adr/](./adr/) for design rationale

## Documentation Standards

### Writing Style
- Use clear, concise language
- Include code examples for complex concepts
- Provide diagrams for architectural concepts
- Maintain consistent formatting

### Code Documentation
- Document public APIs in header files
- Use inline comments for complex logic
- Provide examples for wrapper functions
- Include error handling patterns

### Keeping Documentation Updated
- Update docs when making API changes
- Review documentation during code reviews
- Add ADRs for architectural decisions
- Maintain examples with current codebase

## Getting Help

### Community Resources
- **Mailing List**: dmtcp-forum@lists.sourceforge.net
- **GitHub Issues**: [github.com/dmtcp/dmtcp/issues](https://github.com/dmtcp/dmtcp/issues)
- **GitHub Discussions**: [github.com/dmtcp/dmtcp/discussions](https://github.com/dmtcp/dmtcp/discussions)

### Documentation Issues
- Report documentation problems via GitHub Issues
- Use "documentation" label for doc-specific issues
- Suggest improvements via pull requests

This documentation system aims to provide comprehensive, accessible information for all aspects of DMTCP development and usage.