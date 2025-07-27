# Zig Issue Tracker

A SQLite-based issue tracking system built with Zig, designed to replace markdown-based issue management with a structured database approach.

## Overview

This project aims to convert our current markdown-based issue tracking system into a SQLite database, providing better querying capabilities, dependency tracking, and programmatic access while maintaining the simplicity we value.

### Current System

We currently manage issues using markdown files:
- [`ISSUES.md`](ISSUES.md) - Master list of all issues organized by status and priority
- [`READY.md`](READY.md) - Filtered view of issues ready to work on (no blockers)

### Future System

The SQLite-based system will provide:
- Structured data storage for issues, dependencies, and metadata
- CLI interface for issue management
- HTTP API for integration with other tools
- Better querying and filtering capabilities
- Automated dependency tracking

## Project Status

**🚧 In Development** - Currently in planning phase. The markdown files remain the active issue tracking system.

## Technology Stack

- **Language**: Zig
- **Database**: SQLite
- **Build System**: Zig build system

## Getting Started

### Prerequisites

- Zig 0.15.0-dev.847+ (see `build.zig.zon` for exact version)
- SQLite development libraries (when implemented)

### Building

```bash
zig build
```

### Running

```bash
zig build run
```

### Testing

```bash
zig build test
```

## Planned Features

- [ ] SQLite schema for issues, projects, and dependencies
- [ ] CLI commands for issue CRUD operations
- [ ] Dependency tracking and resolution
- [ ] Import/export from markdown format
- [ ] HTTP API for external integrations
- [ ] Query interface for complex searches

## License

TBD
