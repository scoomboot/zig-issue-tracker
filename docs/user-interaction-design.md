# User Interaction Design for Zig Issue Tracker

## Overview

This document outlines how users will interact with the Zig Issue Tracker system, focusing on providing multiple interaction methods while maintaining simplicity and flexibility. The design prioritizes a smooth migration path from the current markdown-based system to the new database-backed solution.

### Key Principles

1. **Flexibility** - Users can choose their preferred interaction method
2. **Migration Path** - Smooth transition from markdown to database
3. **Offline Support** - Local caching with online sync capabilities
4. **Version Control** - Maintain git history through markdown exports
5. **Automation** - Easy integration with existing development workflows

## User Interaction Methods

### 1. CLI Tool (Primary Interface)

The `zig-issues` command-line tool will be the primary interface for interacting with the issue tracker.

#### Basic Commands

```bash
# Initialize project connection
zig-issues init --project my-app --server https://issues.local

# Issue management
zig-issues add "Fix memory leak in parser" --priority high --assign joe@example.com
zig-issues list --status open --priority high
zig-issues list --ready  # Shows issues with no blockers
zig-issues show 42
zig-issues update 42 --status in_progress
zig-issues close 42 --comment "Fixed in commit abc123"

# Dependency management
zig-issues depend 42 --on 41  # Issue 42 depends on 41
zig-issues depend 42 --remove 41

# Bulk operations
zig-issues import --from issues.json
zig-issues export --format json > backup.json
```

#### Advanced Features

```bash
# Query with filters
zig-issues query "status:open AND priority:high AND assigned:me"

# Templates for common issues
zig-issues add --template bug
zig-issues template create bug --file bug-template.json

# Offline mode
zig-issues --offline list  # Uses local cache
zig-issues sync  # Sync local changes with server
```

### 2. Markdown Bridge (Compatibility Layer)

Provides bidirectional synchronization between markdown files and the database.

#### Import/Export Commands

```bash
# Initial import from existing markdown
zig-issues import --from-markdown ISSUES.md

# Export current state to markdown
zig-issues export --to-markdown > ISSUES.md

# Watch mode for continuous sync
zig-issues watch ISSUES.md --bidirectional
```

#### Markdown Format Support

The system will parse markdown files with this structure:

```markdown
## 🚨 Critical Issues

- [ ] #001: Memory leak in HTTP handler
  - **Priority**: Critical
  - **Status**: Open
  - **Assigned**: alice@example.com
  - **Created**: 2025-01-20
  - **Dependencies**: #002, #003
  - **Details**: Detailed description here...

## 🔧 In Progress

- [ ] #002: Refactor database connection pool
  - **Priority**: High
  - **Status**: In Progress
  - **Assigned**: bob@example.com
```

### 3. Direct API Integration

For automation and programmatic access.

#### Zig Client Library

```zig
const issue_tracker = @import("issue_tracker_client");

pub fn main() !void {
    const client = try issue_tracker.Client.init("https://issues.local", "api-key");
    defer client.deinit();
    
    // Create issue
    const issue = try client.createIssue("my-project", .{
        .title = "Memory leak in parser",
        .priority = .high,
        .assigned_to = "alice@example.com",
    });
    
    // Query issues
    const ready_issues = try client.queryIssues(.{
        .project = "my-project",
        .status = .open,
        .has_blockers = false,
    });
}
```

#### Git Hooks

```bash
#!/bin/bash
# .git/hooks/post-commit
# Auto-close issues mentioned in commit messages

commit_msg=$(git log -1 --pretty=%B)
if [[ $commit_msg =~ "Fixes #([0-9]+)" ]]; then
    issue_id="${BASH_REMATCH[1]}"
    zig-issues close "$issue_id" --comment "Fixed in $(git rev-parse HEAD)"
fi
```

### 4. Future: Editor/IDE Integration

- VSCode extension for inline issue management
- View issue details on hover
- Create issues from TODO comments
- Show issue status in sidebar

## Data Flow Architecture

```
User Interfaces                    Sync Layer                    Storage
┌─────────────────┐                                          
│   CLI Tool      │ ─────┐                                   
│  (zig-issues)   │      │                                   
└─────────────────┘      │                                   
                         │         ┌─────────────────┐       
┌─────────────────┐      │         │                 │       
│ Markdown Files  │ ←────┼────────►│   HTTP API      │       
│  (ISSUES.md)    │      │         │   Server        │       
└─────────────────┘      │         │                 │       
                         │         └────────┬────────┘       
┌─────────────────┐      │                  │                
│   Git Hooks     │ ─────┤                  │                
│  (automated)    │      │                  ▼                
└─────────────────┘      │         ┌─────────────────┐       
                         │         │ SQLite Database │       
┌─────────────────┐      │         │  (issues.db)    │       
│  Zig Library    │ ─────┘         └─────────────────┘       
│    (API)        │                                          
└─────────────────┘                                          

                    ↕ Local Cache
             ┌─────────────────┐
             │  Local Storage  │
             │ (.zig-issues/)  │
             └─────────────────┘
```

## Configuration

### Project Configuration File

Each project will have a `.zig-issues.json` configuration file:

```json
{
  "version": "1.0",
  "server": {
    "url": "https://issues.example.com",
    "api_key": "${ZIG_ISSUES_API_KEY}"
  },
  "project": {
    "name": "my-app",
    "id": "proj_123"
  },
  "sync": {
    "markdown_file": "ISSUES.md",
    "auto_sync": true,
    "sync_interval": 300,
    "conflict_resolution": "server"
  },
  "offline": {
    "enabled": true,
    "cache_dir": ".zig-issues/cache"
  },
  "templates": {
    "bug": ".zig-issues/templates/bug.json",
    "feature": ".zig-issues/templates/feature.json"
  }
}
```

### Global Configuration

User-level configuration in `~/.config/zig-issues/config.json`:

```json
{
  "default_server": "https://issues.example.com",
  "editor": "vim",
  "display": {
    "date_format": "YYYY-MM-DD",
    "color_output": true,
    "page_size": 20
  },
  "aliases": {
    "ls": "list",
    "rm": "delete"
  }
}
```

## Implementation Phases

### Phase 1: Core CLI Tool

**Goal**: Basic issue management via command line

1. **Server Communication**
   - HTTP client implementation
   - Authentication handling
   - Error handling and retries

2. **Basic Commands**
   - init, add, list, show, update, close
   - Project configuration management
   - JSON output format

3. **Local Configuration**
   - Read/write .zig-issues.json
   - Environment variable support
   - API key management

**Deliverables**: Working CLI with basic CRUD operations

### Phase 2: Markdown Integration

**Goal**: Bidirectional sync with markdown files

1. **Markdown Parser**
   - Parse existing ISSUES.md format
   - Handle various markdown structures
   - Extract issue metadata

2. **Sync Engine**
   - Import from markdown
   - Export to markdown
   - Conflict detection and resolution

3. **Watch Mode**
   - File system monitoring
   - Incremental updates
   - Debouncing and batching

**Deliverables**: Seamless markdown compatibility

### Phase 3: Enhanced Workflows

**Goal**: Advanced features and automation

1. **Offline Support**
   - Local caching strategy
   - Sync queue management
   - Conflict resolution

2. **Automation**
   - Git hook templates
   - CI/CD integration examples
   - Batch operations

3. **Advanced Queries**
   - Query language parser
   - Complex filters
   - Saved queries

**Deliverables**: Production-ready tooling

## Use Cases

### Daily Developer Workflow

```bash
# Morning standup - check assigned issues
zig-issues list --assigned me --status "in_progress"

# Start work on an issue
zig-issues update 42 --status in_progress
zig-issues show 42  # View details

# Create a blocker issue
zig-issues add "Database connection fails in test" --priority high
zig-issues depend 42 --on 99  # Original issue depends on new one

# Complete work
zig-issues close 42 --comment "Fixed with new connection pool"
```

### Team Lead Workflow

```bash
# Review ready issues
zig-issues list --ready --unassigned

# Assign work
zig-issues update 43 --assign alice@example.com

# Check team progress
zig-issues report --team --week

# Export for meeting
zig-issues export --format markdown --status in_progress > sprint-status.md
```

### Migration from Markdown

```bash
# Initial setup
cd my-project
zig-issues init --project my-project --import ISSUES.md

# Verify import
zig-issues list --limit 10

# Enable watch mode for gradual migration
zig-issues watch ISSUES.md --bidirectional

# After full migration
zig-issues export --to-markdown > ISSUES_ARCHIVE.md
rm ISSUES.md
```

## Error Handling and Recovery

### Offline Scenarios

- Queue changes locally when server unreachable
- Clear indication of offline mode
- Automatic sync when connection restored
- Conflict resolution on sync

### Data Integrity

- Validate markdown format before import
- Backup before bulk operations
- Transaction support for multi-step operations
- Rollback capability

### User Experience

- Clear error messages with suggested fixes
- Progress indicators for long operations
- Verbose mode for debugging
- Dry-run option for dangerous operations

## Security Considerations

### API Key Management

- Never store keys in version control
- Support environment variables
- Per-project keys for isolation
- Key rotation reminders

### Data Privacy

- Local cache encryption option
- Secure key storage (OS keychain)
- Audit logging for changes
- Project-level access control

## Future Enhancements

1. **Web Dashboard** - Browser-based interface
2. **Mobile App** - View/update issues on the go
3. **IDE Plugins** - Deep editor integration
4. **Voice Interface** - "Hey Zig, create a bug report"
5. **AI Assistant** - Smart issue categorization and assignment

## Conclusion

This interaction design provides multiple ways for users to work with the issue tracker while maintaining the simplicity and flexibility that makes the current markdown system appealing. The phased implementation approach ensures a smooth transition path and allows for iterative improvements based on user feedback.