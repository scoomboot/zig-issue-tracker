# Zig Issue Tracker - HTTP API Server Design

## Overview

This document outlines the design for a centralized HTTP API server approach for the Zig Issue Tracker. This approach provides a single, shared issue tracking service that multiple projects can connect to, enabling cross-project visibility and team collaboration.

## Architecture

### Core Components

```
┌─────────────────┐     HTTP/REST      ┌─────────────────┐
│   Zig Project A │ ◄─────────────────► │                 │
├─────────────────┤                     │                 │
│   Zig Project B │ ◄─────────────────► │  Issue Tracker  │
├─────────────────┤                     │   API Server    │
│     CI/CD       │ ◄─────────────────► │                 │
├─────────────────┤                     │                 │
│   Web Browser   │ ◄─────────────────► │                 │
└─────────────────┘                     └────────┬────────┘
                                                 │
                                        ┌────────▼────────┐
                                        │  SQLite Database │
                                        │   (issues.db)    │
                                        └─────────────────┘
```

### Key Design Principles

1. **RESTful API** - Standard HTTP methods (GET, POST, PUT, DELETE)
2. **JSON Format** - All requests and responses use JSON
3. **Stateless** - Each request contains all necessary information
4. **Version Prefixed** - API endpoints start with `/api/v1/`
5. **Consistent Error Handling** - Standardized error response format

## Database Schema

### Core Tables

```sql
-- Projects table
CREATE TABLE projects (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    repository_url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Issues table
CREATE TABLE issues (
    id INTEGER PRIMARY KEY,
    project_id INTEGER NOT NULL,
    issue_number INTEGER NOT NULL,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT DEFAULT 'open',  -- open, in_progress, completed, blocked
    priority TEXT DEFAULT 'medium',  -- low, medium, high, critical
    created_by TEXT,
    assigned_to TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    FOREIGN KEY (project_id) REFERENCES projects(id),
    UNIQUE(project_id, issue_number)
);

-- Dependencies table
CREATE TABLE dependencies (
    id INTEGER PRIMARY KEY,
    issue_id INTEGER NOT NULL,
    depends_on_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (issue_id) REFERENCES issues(id),
    FOREIGN KEY (depends_on_id) REFERENCES issues(id),
    UNIQUE(issue_id, depends_on_id)
);

-- Tags table
CREATE TABLE tags (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

-- Issue tags junction table
CREATE TABLE issue_tags (
    issue_id INTEGER NOT NULL,
    tag_id INTEGER NOT NULL,
    FOREIGN KEY (issue_id) REFERENCES issues(id),
    FOREIGN KEY (tag_id) REFERENCES tags(id),
    PRIMARY KEY (issue_id, tag_id)
);

-- Comments table
CREATE TABLE comments (
    id INTEGER PRIMARY KEY,
    issue_id INTEGER NOT NULL,
    author TEXT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (issue_id) REFERENCES issues(id)
);
```

## API Endpoints

### Projects

```
GET    /api/v1/projects              # List all projects
GET    /api/v1/projects/:id          # Get project details
POST   /api/v1/projects              # Create new project
PUT    /api/v1/projects/:id          # Update project
DELETE /api/v1/projects/:id          # Delete project
```

### Issues

```
GET    /api/v1/issues                # List all issues (with filters)
GET    /api/v1/projects/:pid/issues  # List project issues
GET    /api/v1/issues/:id            # Get issue details
POST   /api/v1/projects/:pid/issues  # Create issue in project
PUT    /api/v1/issues/:id            # Update issue
DELETE /api/v1/issues/:id            # Delete issue

# Special endpoints
GET    /api/v1/issues/ready          # Get all ready issues (no blockers)
GET    /api/v1/issues/blocked        # Get all blocked issues
```

### Dependencies

```
POST   /api/v1/issues/:id/dependencies      # Add dependency
DELETE /api/v1/issues/:id/dependencies/:did  # Remove dependency
GET    /api/v1/issues/:id/dependencies      # List dependencies
GET    /api/v1/issues/:id/dependents        # List dependent issues
```

### Comments

```
GET    /api/v1/issues/:id/comments   # List comments
POST   /api/v1/issues/:id/comments   # Add comment
PUT    /api/v1/comments/:id          # Update comment
DELETE /api/v1/comments/:id          # Delete comment
```

### Tags

```
GET    /api/v1/tags                  # List all tags
POST   /api/v1/issues/:id/tags       # Add tag to issue
DELETE /api/v1/issues/:id/tags/:tid  # Remove tag from issue
```

## Request/Response Examples

### Create Issue
```http
POST /api/v1/projects/1/issues
Content-Type: application/json

{
  "title": "Implement user authentication",
  "description": "Add JWT-based auth to API endpoints",
  "priority": "high",
  "assigned_to": "alice@example.com",
  "tags": ["security", "api"]
}
```

Response:
```json
{
  "id": 42,
  "project_id": 1,
  "issue_number": 15,
  "title": "Implement user authentication",
  "description": "Add JWT-based auth to API endpoints",
  "status": "open",
  "priority": "high",
  "assigned_to": "alice@example.com",
  "created_by": "bob@example.com",
  "created_at": "2025-07-27T10:30:00Z",
  "updated_at": "2025-07-27T10:30:00Z",
  "tags": ["security", "api"],
  "dependencies": [],
  "_links": {
    "self": "/api/v1/issues/42",
    "project": "/api/v1/projects/1",
    "comments": "/api/v1/issues/42/comments"
  }
}
```

### Query Ready Issues
```http
GET /api/v1/issues/ready?project=my-app&priority=high
```

Response:
```json
{
  "issues": [
    {
      "id": 42,
      "title": "Implement user authentication",
      "project": {
        "id": 1,
        "name": "my-app"
      },
      "priority": "high",
      "status": "open",
      "assigned_to": "alice@example.com"
    }
  ],
  "total": 1,
  "_links": {
    "self": "/api/v1/issues/ready?project=my-app&priority=high"
  }
}
```

### Error Response Format
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Issue with ID 999 not found",
    "details": {
      "resource": "issue",
      "id": 999
    }
  },
  "timestamp": "2025-07-27T10:30:00Z"
}
```

## Authentication & Authorization

### Options for v1

1. **API Keys** (Simplest for v1)
   ```http
   GET /api/v1/issues
   Authorization: Bearer your-api-key-here
   ```

2. **Basic Auth** (For simple deployments)
   ```http
   GET /api/v1/issues
   Authorization: Basic base64(username:password)
   ```

### Future Considerations
- JWT tokens for stateless auth
- OAuth2 for third-party integrations
- Project-level permissions
- Role-based access control (RBAC)

## Integration Patterns

### 1. Zig Client Library
```zig
const issue_tracker = @import("issue_tracker_client");

pub fn main() !void {
    const client = try issue_tracker.Client.init("https://issues.example.com", "api-key");
    defer client.deinit();
    
    // Create issue
    const issue = try client.createIssue("my-project", .{
        .title = "Memory leak in parser",
        .priority = .high,
    });
    
    // Query ready issues
    const ready = try client.getReadyIssues(.{
        .project = "my-project",
        .priority = .high,
    });
}
```

### 2. Command Line Integration
```bash
# Configure server
export ISSUE_TRACKER_URL="https://issues.example.com"
export ISSUE_TRACKER_KEY="your-api-key"

# Create issue
curl -X POST $ISSUE_TRACKER_URL/api/v1/projects/1/issues \
  -H "Authorization: Bearer $ISSUE_TRACKER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Fix bug", "priority": "high"}'
```

### 3. CI/CD Integration
```yaml
# GitHub Actions example
- name: Create Issue on Test Failure
  if: failure()
  run: |
    curl -X POST ${{ secrets.ISSUE_TRACKER_URL }}/api/v1/projects/1/issues \
      -H "Authorization: Bearer ${{ secrets.ISSUE_TRACKER_KEY }}" \
      -H "Content-Type: application/json" \
      -d '{
        "title": "CI Build Failed: ${{ github.run_id }}",
        "description": "Build failed on commit ${{ github.sha }}",
        "priority": "high",
        "tags": ["ci", "automated"]
      }'
```

## Benefits of Centralized Approach

### Advantages
1. **Cross-Project Visibility** - See issues across all your projects
2. **Team Collaboration** - Shared issue tracking for team members
3. **Unified Reporting** - Generate reports across projects
4. **Single Source of Truth** - One database to backup/manage
5. **Web UI Potential** - Easy to add web interface later
6. **Integration Friendly** - Works with any language/platform

### Trade-offs
1. **Network Dependency** - Requires HTTP connectivity
2. **Deployment Complexity** - Need to run and maintain server
3. **Single Point of Failure** - Server downtime affects all projects
4. **Latency** - Network calls vs local file access
5. **Security Considerations** - Need authentication/authorization

## Deployment Options

### 1. Local Development
```bash
zig build run -- serve --port 8080 --db ./issues.db
```

### 2. Docker Container
```dockerfile
FROM alpine:latest
RUN apk add --no-cache sqlite
COPY zig-issue-tracker /usr/local/bin/
EXPOSE 8080
CMD ["zig-issue-tracker", "serve", "--port", "8080"]
```

### 3. SystemD Service
```ini
[Unit]
Description=Zig Issue Tracker API Server
After=network.target

[Service]
Type=simple
User=issues
ExecStart=/usr/local/bin/zig-issue-tracker serve --port 8080
Restart=always

[Install]
WantedBy=multi-user.target
```

## Future Enhancements

1. **WebSocket Support** - Real-time issue updates
2. **Webhooks** - Notify external services of changes
3. **Full-Text Search** - Search across issue content
4. **Metrics/Analytics** - Issue velocity, burndown charts
5. **Import/Export** - Migrate from GitHub Issues, Jira, etc.
6. **Web UI** - Browser-based interface
7. **GraphQL API** - Alternative to REST for complex queries

## Example Workflows

### Daily Standup Query
```bash
# Get all in-progress issues assigned to team
curl "$API_URL/api/v1/issues?status=in_progress&assigned_to=team" \
  -H "Authorization: Bearer $API_KEY"
```

### Dependency Check
```bash
# Check if issue has blocking dependencies
curl "$API_URL/api/v1/issues/42/dependencies" \
  -H "Authorization: Bearer $API_KEY" | \
  jq '.dependencies[] | select(.status != "completed")'
```

### Project Migration
```bash
# Export issues from one project
curl "$API_URL/api/v1/projects/old-project/issues" > issues.json

# Import to new project
cat issues.json | jq '.issues[]' | while read -r issue; do
  curl -X POST "$API_URL/api/v1/projects/new-project/issues" \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d "$issue"
done
```

## Implementation Phases

### Phase 1: Core API
- Basic CRUD for issues and projects
- SQLite database setup
- Simple API key authentication

### Phase 2: Relationships
- Dependencies tracking
- Tags and comments
- Query filtering

### Phase 3: Advanced Features  
- Webhooks
- Batch operations
- Performance optimizations

### Phase 4: Ecosystem
- Client libraries
- CLI tool
- Documentation site

## Conclusion

The HTTP API server approach provides a flexible, scalable solution for centralized issue tracking. While it adds deployment complexity compared to embedded solutions, it enables powerful cross-project workflows and team collaboration features that justify the trade-offs for most team environments.