# Zig Issue Tracker - HTTP API Server Design

## Overview

This document outlines the design for a centralized HTTP API server approach for the Zig Issue Tracker. This approach provides a single, shared issue tracking service that multiple projects can connect to, enabling cross-project visibility and team collaboration.

### HTTP Server Framework

The server will be built using **httpz**, a lightweight and performant HTTP server library for Zig. httpz provides:
- Simple and intuitive API for building web services
- Built-in JSON response handling
- Efficient memory management with request arenas
- Path parameter and query string parsing
- Middleware and error handler support
- Native Zig implementation with zero dependencies

### Reference Implementation

> **Note**: A working httpz implementation can be found in the **zig-db** project at `/home/emoessner/db/zig-db/`. Key files to examine:
> - `src/http/server.zig` - Server wrapper pattern with configuration
> - `src/http/app_state.zig` - Application state management
> - `src/http/handlers.zig` - Handler patterns with httpz
> - `src/main.zig` - Simple server initialization
> - `build.zig` & `build.zig.zon` - Dependency configuration
> 
> Many patterns from this project can be directly reused for the issue tracker.

## Architecture

### Core Components

```
┌─────────────────┐     HTTP/REST      ┌─────────────────┐
│   Zig Project A │ ◄─────────────────► │                 │
├─────────────────┤                     │                 │
│   Zig Project B │ ◄─────────────────► │  Issue Tracker  │
├─────────────────┤                     │   API Server    │
│     CI/CD       │ ◄─────────────────► │    (httpz)      │
├─────────────────┤                     │                 │
│   Web Browser   │ ◄─────────────────► │                 │
└─────────────────┘                     └────────┬────────┘
                                                 │
                                        ┌────────▼────────┐
                                        │  SQLite Database │
                                        │   (issues.db)    │
                                        └─────────────────┘
```

### Technology Stack

- **Language**: Zig
- **HTTP Server**: httpz
- **Database**: SQLite (via zqlite)
- **Serialization**: std.json

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

## Server Implementation

> **Reference**: See `/home/emoessner/db/zig-db/src/http/` for working patterns:
> - `server.zig` - Enhanced server wrapper with configuration
> - `app_state.zig` - Clean AppState structure with atomic counters
> - `handlers.zig` - Handler signatures and JSON responses

### Basic Server Setup

```zig
const std = @import("std");
const httpz = @import("httpz");
const database = @import("database.zig");

// AppState pattern from zig-db/src/http/app_state.zig
const AppState = struct {
    db_pool: *database.Pool,
    api_keys: std.StringHashMap(bool),
    start_time: i64,
    request_count: std.atomic.Value(u64), // Thread-safe counter
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Initialize database pool
    const db_pool = try database.Pool.init(allocator, .{
        .path = "./issues.db",
        .min_connections = 5,
        .max_connections = 20,
    });
    defer db_pool.deinit();

    // Initialize app state
    var app_state = AppState{
        .db_pool = db_pool,
        .api_keys = std.StringHashMap(bool).init(allocator),
        .start_time = std.time.timestamp(),
    };
    defer app_state.api_keys.deinit();

    // Create server
    var server = try httpz.Server(AppState).init(allocator, .{
        .port = 8080,
        .address = "0.0.0.0",
        .request_timeout = 30000,      // 30 seconds
        .keepalive_timeout = 120000,   // 2 minutes
        .max_request_size = 1048576,   // 1MB
    }, app_state);
    defer server.deinit();

    // Set up routes
    var router = try server.router(.{
        .not_found_handler = notFoundHandler,
        .uncaught_error_handler = errorHandler,
    });
    
    // Project routes
    router.get("/api/v1/projects", handlers.getProjects, .{});
    router.get("/api/v1/projects/:id", handlers.getProject, .{});
    router.post("/api/v1/projects", handlers.createProject, .{});
    router.put("/api/v1/projects/:id", handlers.updateProject, .{});
    router.delete("/api/v1/projects/:id", handlers.deleteProject, .{});
    
    // Issue routes
    router.get("/api/v1/issues", handlers.getIssues, .{});
    router.get("/api/v1/issues/:id", handlers.getIssue, .{});
    router.post("/api/v1/projects/:pid/issues", handlers.createIssue, .{});
    router.put("/api/v1/issues/:id", handlers.updateIssue, .{});
    router.delete("/api/v1/issues/:id", handlers.deleteIssue, .{});
    
    // Special endpoints
    router.get("/api/v1/issues/ready", handlers.getReadyIssues, .{});
    router.get("/api/v1/issues/blocked", handlers.getBlockedIssues, .{});
    
    std.log.info("Issue Tracker API Server starting on port 8080", .{});
    try server.listen();
}

fn notFoundHandler(_: *httpz.Request, res: *httpz.Response, _: AppState) !void {
    try res.json(.{ 
        .@"error" = "Endpoint not found",
        .message = "The requested resource does not exist"
    }, .{ .status = 404 });
}

fn errorHandler(req: *httpz.Request, res: *httpz.Response, err: anyerror, _: AppState) void {
    std.log.err("Uncaught error on {s}: {any}", .{ req.url.path, err });
    
    res.status = 500;
    res.json(.{ 
        .@"error" = "Internal server error",
        .message = "An unexpected error occurred",
        .timestamp = std.time.timestamp(),
    }, .{}) catch {};
}
```

### Handler Implementation Examples

> **Reference**: See `/home/emoessner/db/zig-db/src/http/handlers.zig` for simpler handler patterns including:
> - Handler signatures (Note: zig-db uses `AppState, Request, Response` order)
> - JSON response patterns
> - HTML response examples
> - Atomic counter usage

```zig
const handlers = struct {
    // GET /api/v1/issues/:id
    pub fn getIssue(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
        // Extract and validate issue ID
        const issue_id_str = req.param("id") orelse {
            try res.json(.{ .@"error" = "Issue ID required" }, .{ .status = 400 });
            return;
        };
        
        const issue_id = std.fmt.parseInt(u32, issue_id_str, 10) catch {
            try res.json(.{ .@"error" = "Invalid issue ID format" }, .{ .status = 400 });
            return;
        };
        
        // Get database connection
        var conn = app_state.db_pool.acquire() catch {
            try res.json(.{ .@"error" = "Service temporarily unavailable" }, .{ .status = 503 });
            return;
        };
        defer app_state.db_pool.release(conn);
        
        // Query issue
        const issue = conn.getIssue(issue_id) catch |err| switch (err) {
            error.NotFound => {
                try res.json(.{ .@"error" = "Issue not found" }, .{ .status = 404 });
                return;
            },
            else => {
                std.log.err("Database error: {any}", .{err});
                try res.json(.{ .@"error" = "Database error" }, .{ .status = 500 });
                return;
            },
        };
        
        // Return issue with HATEOAS links
        try res.json(.{
            .id = issue.id,
            .project_id = issue.project_id,
            .issue_number = issue.issue_number,
            .title = issue.title,
            .description = issue.description,
            .status = issue.status,
            .priority = issue.priority,
            .created_at = issue.created_at,
            .updated_at = issue.updated_at,
            ._links = .{
                .self = try std.fmt.allocPrint(req.arena, "/api/v1/issues/{d}", .{issue.id}),
                .project = try std.fmt.allocPrint(req.arena, "/api/v1/projects/{d}", .{issue.project_id}),
                .comments = try std.fmt.allocPrint(req.arena, "/api/v1/issues/{d}/comments", .{issue.id}),
            },
        }, .{});
    }
    
    // POST /api/v1/projects/:pid/issues
    pub fn createIssue(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
        // Validate project ID
        const project_id_str = req.param("pid") orelse {
            try res.json(.{ .@"error" = "Project ID required" }, .{ .status = 400 });
            return;
        };
        
        const project_id = std.fmt.parseInt(u32, project_id_str, 10) catch {
            try res.json(.{ .@"error" = "Invalid project ID format" }, .{ .status = 400 });
            return;
        };
        
        // Parse request body
        const body = req.body() orelse {
            try res.json(.{ .@"error" = "Request body required" }, .{ .status = 400 });
            return;
        };
        
        const IssueInput = struct {
            title: []const u8,
            description: ?[]const u8 = null,
            priority: ?[]const u8 = "medium",
            assigned_to: ?[]const u8 = null,
            tags: ?[][]const u8 = null,
        };
        
        const input = std.json.parseFromSlice(IssueInput, req.arena, body, .{}) catch {
            try res.json(.{ .@"error" = "Invalid JSON format" }, .{ .status = 400 });
            return;
        };
        defer input.deinit();
        
        // Validate input
        if (input.value.title.len == 0) {
            try res.json(.{ .@"error" = "Title is required" }, .{ .status = 400 });
            return;
        }
        
        // Create issue in database
        var conn = app_state.db_pool.acquire() catch {
            try res.json(.{ .@"error" = "Service temporarily unavailable" }, .{ .status = 503 });
            return;
        };
        defer app_state.db_pool.release(conn);
        
        const issue_id = conn.createIssue(.{
            .project_id = project_id,
            .title = input.value.title,
            .description = input.value.description,
            .priority = input.value.priority orelse "medium",
            .assigned_to = input.value.assigned_to,
        }) catch |err| {
            std.log.err("Failed to create issue: {any}", .{err});
            try res.json(.{ .@"error" = "Failed to create issue" }, .{ .status = 500 });
            return;
        };
        
        // Return created issue
        res.status = 201;
        try res.json(.{
            .id = issue_id,
            .message = "Issue created successfully",
            ._links = .{
                .self = try std.fmt.allocPrint(req.arena, "/api/v1/issues/{d}", .{issue_id}),
            },
        }, .{});
    }
    
    // GET /api/v1/issues/ready
    pub fn getReadyIssues(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
        // Parse query parameters
        const project = req.query("project");
        const priority = req.query("priority");
        const limit = blk: {
            const limit_str = req.query("limit") orelse "20";
            break :blk std.fmt.parseInt(u32, limit_str, 10) catch 20;
        };
        
        // Validate limit
        if (limit > 100) {
            try res.json(.{ .@"error" = "Limit cannot exceed 100" }, .{ .status = 400 });
            return;
        }
        
        // Query ready issues (no blocking dependencies)
        var conn = app_state.db_pool.acquire() catch {
            try res.json(.{ .@"error" = "Service temporarily unavailable" }, .{ .status = 503 });
            return;
        };
        defer app_state.db_pool.release(conn);
        
        const issues = conn.getReadyIssues(.{
            .project_name = project,
            .priority = priority,
            .limit = limit,
        }) catch |err| {
            std.log.err("Failed to get ready issues: {any}", .{err});
            try res.json(.{ .@"error" = "Failed to retrieve issues" }, .{ .status = 500 });
            return;
        };
        
        // Format response
        try res.json(.{
            .issues = issues,
            .total = issues.len,
            ._links = .{
                .self = req.url.raw,
            },
        }, .{});
    }
};
```

### Authentication Middleware

```zig
fn authenticateRequest(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !bool {
    const auth_header = req.header("authorization") orelse {
        try res.json(.{ .@"error" = "Authentication required" }, .{ .status = 401 });
        return false;
    };
    
    if (!std.mem.startsWith(u8, auth_header, "Bearer ")) {
        try res.json(.{ .@"error" = "Invalid authorization format" }, .{ .status = 401 });
        return false;
    }
    
    const api_key = auth_header[7..];
    if (!app_state.api_keys.contains(api_key)) {
        try res.json(.{ .@"error" = "Invalid API key" }, .{ .status = 401 });
        return false;
    }
    
    return true;
}

// Use in handlers
pub fn protectedHandler(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    if (!try authenticateRequest(req, res, app_state)) return;
    
    // Handler implementation...
}
```

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
zig build run
# Or with custom options
zig build run -- --port 8080 --db ./issues.db
```

### build.zig Configuration

> **Reference**: See `/home/emoessner/db/zig-db/build.zig` for complete working example

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // Add httpz dependency (pattern from zig-db/build.zig)
    const httpz = b.dependency("httpz", .{
        .target = target,
        .optimize = optimize,
    });
    
    // Add zqlite dependency for SQLite
    const zqlite = b.dependency("zqlite", .{
        .target = target,
        .optimize = optimize,
    });
    
    const exe = b.addExecutable(.{
        .name = "zig-issue-tracker",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Add modules (exact pattern from zig-db line 101-102)
    exe.root_module.addImport("httpz", httpz.module("httpz"));
    exe.root_module.addImport("zqlite", zqlite.module("zqlite"));
    
    // Link SQLite
    exe.linkSystemLibrary("sqlite3");
    exe.linkLibC();
    
    b.installArtifact(exe);
}
```

### build.zig.zon Dependencies

> **Reference**: See `/home/emoessner/db/zig-db/build.zig.zon` for working dependencies

```zig
.{
    .name = "zig-issue-tracker",
    .version = "0.1.0",
    .dependencies = .{
        .httpz = .{
            // Exact dependency from zig-db project (tested and working)
            .url = "git+https://github.com/karlseguin/http.zig?ref=master#2924ead41db9e1986db3b4909a82185c576f6361",
            .hash = "httpz-0.0.0-PNVzrC7CBgDSMZNP7chgAczR3FyAz5Eum6Kb1_K9kmYR",
        },
        .zqlite = .{
            // Exact dependency from zig-db project
            .url = "git+https://github.com/karlseguin/zqlite.zig?ref=master#fdb6ca1e379b002dd6effe22b96dc8ee93befe5e",
            .hash = "zqlite-0.0.0-RWLaY5eAlQCBMG2E0Y8JjWkT-SZlZp39cu75CUVb2QoP",
        },
    },
}
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

## Advanced httpz Features

### CORS Support

```zig
fn setupCORS(router: *httpz.Router(AppState)) void {
    // Handle preflight requests
    router.options("/*", corsHandler, .{});
    
    // Apply CORS to all routes
    router.all("/*", addCORSHeaders, .{});
}

fn corsHandler(req: *httpz.Request, res: *httpz.Response, _: AppState) !void {
    res.headers.add("Access-Control-Allow-Origin", "*");
    res.headers.add("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
    res.headers.add("Access-Control-Allow-Headers", "Content-Type, Authorization");
    res.headers.add("Access-Control-Max-Age", "86400");
    res.status = 204;
}

fn addCORSHeaders(req: *httpz.Request, res: *httpz.Response, _: AppState) !void {
    res.headers.add("Access-Control-Allow-Origin", "*");
}
```

### Request Validation Helpers

```zig
const validation = struct {
    pub fn requireParam(req: *httpz.Request, res: *httpz.Response, name: []const u8) ?[]const u8 {
        const value = req.param(name) orelse {
            res.json(.{ .@"error" = std.fmt.allocPrint(req.arena, "{s} is required", .{name}) catch "Parameter required" }, .{ .status = 400 }) catch {};
            return null;
        };
        return value;
    }
    
    pub fn parseIntParam(req: *httpz.Request, res: *httpz.Response, name: []const u8) ?u32 {
        const str = requireParam(req, res, name) orelse return null;
        return std.fmt.parseInt(u32, str, 10) catch {
            res.json(.{ .@"error" = std.fmt.allocPrint(req.arena, "Invalid {s} format", .{name}) catch "Invalid format" }, .{ .status = 400 }) catch {};
            return null;
        };
    }
    
    pub fn validateLimit(req: *httpz.Request, default: u32, max: u32) u32 {
        const limit_str = req.query("limit") orelse return default;
        const limit = std.fmt.parseInt(u32, limit_str, 10) catch return default;
        return @min(limit, max);
    }
};
```

### Performance Monitoring

```zig
const RequestMetrics = struct {
    total_requests: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    error_count: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    
    pub fn recordRequest(self: *RequestMetrics, is_error: bool) void {
        _ = self.total_requests.fetchAdd(1, .monotonic);
        if (is_error) {
            _ = self.error_count.fetchAdd(1, .monotonic);
        }
    }
};

// Add metrics endpoint
router.get("/api/v1/metrics", getMetrics, .{});

fn getMetrics(_: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const uptime = std.time.timestamp() - app_state.start_time;
    const total = app_state.metrics.total_requests.load(.monotonic);
    const errors = app_state.metrics.error_count.load(.monotonic);
    
    try res.json(.{
        .uptime_seconds = uptime,
        .total_requests = total,
        .error_count = errors,
        .error_rate = if (total > 0) @as(f64, @floatFromInt(errors)) / @as(f64, @floatFromInt(total)) else 0,
    }, .{});
}
```

## Future Enhancements

1. **WebSocket Support** - Real-time issue updates using httpz websocket features
2. **Webhooks** - Notify external services of changes
3. **Full-Text Search** - Search across issue content using SQLite FTS5
4. **Metrics/Analytics** - Issue velocity, burndown charts
5. **Import/Export** - Migrate from GitHub Issues, Jira, etc.
6. **Web UI** - Browser-based interface served by httpz
7. **GraphQL API** - Alternative to REST for complex queries
8. **Rate Limiting** - Protect API from abuse
9. **Request Caching** - Cache common queries for performance

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

## Project Structure

```
zig-issue-tracker/
├── build.zig
├── build.zig.zon
├── src/
│   ├── main.zig           # Server entry point
│   ├── handlers.zig       # HTTP request handlers
│   ├── database.zig       # Database pool and connections
│   ├── models.zig         # Data structures
│   ├── validation.zig     # Input validation helpers
│   └── auth.zig           # Authentication middleware
├── docs/
│   └── http-api-design.md # This document
└── README.md
```

## Reusable Components from zig-db

The following components from the zig-db project can be directly adapted for the issue tracker:

1. **Build Configuration**
   - Copy exact httpz and zqlite dependencies from `build.zig.zon`
   - Use the module import pattern from `build.zig`

2. **Server Structure**
   - Adapt the server wrapper pattern from `src/http/server.zig`
   - Use the AppState pattern from `src/http/app_state.zig`
   - Copy atomic counter implementation for thread-safe metrics

3. **Handler Patterns**
   - Study handler signatures in `src/http/handlers.zig`
   - Note parameter ordering differences between projects
   - Reuse JSON response helpers

4. **Project Organization**
   - Follow the `src/http/` subdirectory structure
   - Separate concerns into focused modules

The zig-db project provides a solid foundation for httpz usage that can accelerate the issue tracker implementation.

## Conclusion

The HTTP API server approach using httpz provides a flexible, scalable solution for centralized issue tracking. httpz's simple API, efficient memory management, and built-in features like JSON handling make it an excellent choice for building REST APIs in Zig. While the centralized approach adds deployment complexity compared to embedded solutions, it enables powerful cross-project workflows and team collaboration features that justify the trade-offs for most team environments.

Key advantages of using httpz:
- Native Zig implementation with zero dependencies
- Request arena allocator for automatic memory cleanup
- Built-in JSON serialization/deserialization
- Simple routing with path parameters
- Easy error handling patterns
- Good performance characteristics

This design provides a solid foundation for building a production-ready issue tracking API server in Zig.