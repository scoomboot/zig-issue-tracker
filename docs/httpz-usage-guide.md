# httpz Usage Guide

*Comprehensive development guide for using httpz in the NFL Statistics Database*

## Table of Contents

1. [Quick Reference](#quick-reference)
2. [Basic Setup & Configuration](#basic-setup--configuration)
3. [Routing & Request Handling](#routing--request-handling)
4. [JSON APIs](#json-apis)
5. [Advanced Features](#advanced-features)
6. [Database Integration Patterns](#database-integration-patterns)
7. [Performance & Memory Management](#performance--memory-management)
8. [NFL Stats API Examples](#nfl-stats-api-examples)
9. [Troubleshooting](#troubleshooting)

---

## Quick Reference

### Common Tasks Cheat Sheet

```zig
// Server initialization
var server = try httpz.Server(void).init(allocator, .{
    .port = 8080,
    .address = "127.0.0.1",
}, {});

// Routing
var router = try server.router(.{});
router.get("/api/players", getPlayers, .{});
router.get("/api/players/:id", getPlayer, .{});
router.post("/api/players", createPlayer, .{});
router.put("/api/players/:id", updatePlayer, .{});
router.delete("/api/players/:id", deletePlayer, .{});

// JSON responses
try res.json(.{ .id = 1, .name = "Tom Brady" }, .{});
try res.json(.{ .@"error" = "Player not found" }, .{ .status = 404 });

// Path parameters
const player_id = req.param("id");

// Query parameters
const limit = req.query("limit");
const offset = req.query("offset");

// Request body parsing
const body = try req.json(PlayerData);
```

### HTTP Status Codes
- `200` - OK (successful GET, PUT)
- `201` - Created (successful POST)
- `204` - No Content (successful DELETE)
- `400` - Bad Request (invalid input)
- `404` - Not Found (resource doesn't exist)
- `500` - Internal Server Error (server error)

---

## Basic Setup & Configuration

### Simple Server (Current Pattern)

```zig
const std = @import("std");
const httpz = @import("httpz");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Create server with minimal configuration
    var server = try httpz.Server(void).init(allocator, .{
        .port = 8080,
        .address = "127.0.0.1",
    }, {});
    defer server.deinit();

    // Set up routing
    var router = try server.router(.{});
    router.get("/", handleRoot, .{});
    
    std.log.info("Server starting on port 8080", .{});
    try server.listen();
}
```

### Server with Application State

```zig
const AppState = struct {
    start_time: i64,
    request_count: std.atomic.Value(u64),
    db_pool: *DatabasePool,
};

pub fn main() !void {
    var app_state = AppState{
        .start_time = std.time.timestamp(),
        .request_count = std.atomic.Value(u64).init(0),
        .db_pool = try createDbPool(allocator),
    };

    var server = try httpz.Server(AppState).init(allocator, .{
        .port = 8080,
        .address = "0.0.0.0",
        .timeout_request = 10000,      // 10 seconds
        .timeout_keepalive = 120000,   // 2 minutes
        .max_form_count = 100,
        .max_query_count = 50,
    }, app_state);
    defer server.deinit();
}
```

### Configuration Options

```zig
const config = httpz.Config{
    .port = 8080,
    .address = "0.0.0.0",
    
    // Timeouts (milliseconds)
    .timeout_request = 30000,       // Request timeout
    .timeout_keepalive = 120000,    // Keep-alive timeout
    
    // Limits
    .max_request_size = 1048576,    // 1MB max request size
    .max_form_count = 100,          // Max form fields
    .max_query_count = 50,          // Max query parameters
    .max_multiform_count = 20,      // Max multipart form fields
    
    // Buffer sizes
    .buffer_size = 4096,            // Initial buffer size
    .max_header_count = 32,         // Max headers per request
};
```

---

## Routing & Request Handling

### Basic Routing Patterns

```zig
// Simple routes
router.get("/", handleRoot, .{});
router.get("/health", handleHealth, .{});

// Path parameters
router.get("/api/players/:id", getPlayer, .{});
router.get("/api/teams/:team_id/players/:player_id", getTeamPlayer, .{});

// Multiple HTTP methods for same path
router.get("/api/players", getPlayers, .{});
router.post("/api/players", createPlayer, .{});

// Wildcard routes (use sparingly)
router.get("/static/*", serveStatic, .{});
```

### Handler Function Signatures

```zig
// Basic handler (no state)
fn handleHealth(_: *httpz.Request, res: *httpz.Response) !void {
    try res.json(.{ .status = "healthy" }, .{});
}

// Handler with application state
fn getPlayers(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    _ = app_state.request_count.fetchAdd(1, .monotonic);
    // Implementation...
}

// Handler with custom context (advanced)
fn authenticatedHandler(req: *httpz.Request, res: *httpz.Response, ctx: *AuthContext) !void {
    if (!ctx.is_authenticated) {
        try res.json(.{ .@"error" = "Unauthorized" }, .{ .status = 401 });
        return;
    }
    // Implementation...
}
```

### Request Data Access

```zig
fn handlePlayerRequest(req: *httpz.Request, res: *httpz.Response) !void {
    // Path parameters
    const player_id_str = req.param("id");
    const player_id = std.fmt.parseInt(u32, player_id_str.?, 10) catch {
        try res.json(.{ .@"error" = "Invalid player ID" }, .{ .status = 400 });
        return;
    };

    // Query parameters with defaults
    const limit_str = req.query("limit") orelse "20";
    const limit = std.fmt.parseInt(u32, limit_str, 10) catch 20;
    
    const offset_str = req.query("offset") orelse "0";
    const offset = std.fmt.parseInt(u32, offset_str, 10) catch 0;

    // Headers
    const content_type = req.header("content-type");
    const authorization = req.header("authorization");

    // Request body
    const body_bytes = req.body();
    if (body_bytes) |bytes| {
        // Parse JSON body
        const PlayerInput = struct {
            name: []const u8,
            team_id: u32,
            position: []const u8,
        };
        
        const input = std.json.parseFromSlice(PlayerInput, req.arena, bytes, .{}) catch {
            try res.json(.{ .@"error" = "Invalid JSON" }, .{ .status = 400 });
            return;
        };
        defer input.deinit();
        
        // Use input.value.name, input.value.team_id, etc.
    }
}
```

---

## JSON APIs

### Response Patterns

```zig
// Success responses
try res.json(.{ .id = 1, .name = "Tom Brady" }, .{});                    // 200 OK
try res.json(.{ .id = 1, .name = "Tom Brady" }, .{ .status = 201 });     // 201 Created

// Error responses
try res.json(.{ .@"error" = "Player not found" }, .{ .status = 404 });
try res.json(.{ 
    .@"error" = "Validation failed", 
    .details = .{ .field = "name", .message = "Name is required" }
}, .{ .status = 400 });

// Paginated responses
try res.json(.{
    .data = players,
    .pagination = .{
        .page = 1,
        .per_page = 20,
        .total = 1247,
        .pages = 63,
    },
}, .{});

// No content response (for DELETE)
res.status = 204;
```

### Content-Type Handling

```zig
fn handlePlayerData(req: *httpz.Request, res: *httpz.Response) !void {
    const content_type = req.header("content-type") orelse "";
    
    if (std.mem.startsWith(u8, content_type, "application/json")) {
        // Handle JSON input
        const body = req.body() orelse {
            try res.json(.{ .@"error" = "Request body required" }, .{ .status = 400 });
            return;
        };
        
        // Parse JSON...
    } else if (std.mem.startsWith(u8, content_type, "application/x-www-form-urlencoded")) {
        // Handle form data
        const name = req.formValue("name");
        const team_id = req.formValue("team_id");
        // Process form data...
    } else {
        try res.json(.{ .@"error" = "Unsupported content type" }, .{ .status = 415 });
        return;
    }
}
```

### Validation Helpers

```zig
fn validatePlayerInput(input: anytype) !void {
    if (input.name.len == 0) {
        return error.NameRequired;
    }
    if (input.name.len > 50) {
        return error.NameTooLong;
    }
    if (input.team_id == 0) {
        return error.TeamIdRequired;
    }
}

fn handleValidationError(err: anytype, res: *httpz.Response) !void {
    const message = switch (err) {
        error.NameRequired => "Name is required",
        error.NameTooLong => "Name must be 50 characters or less",
        error.TeamIdRequired => "Team ID is required",
        else => "Validation failed",
    };
    
    try res.json(.{ .@"error" = message }, .{ .status = 400 });
}
```

---

## Advanced Features

### Custom Error Handlers

```zig
var server = try httpz.Server(AppState).init(allocator, .{
    .port = 8080,
}, app_state);

var router = try server.router(.{
    .not_found_handler = notFoundHandler,
    .uncaught_error_handler = uncaughtErrorHandler,
});

fn notFoundHandler(_: *httpz.Request, res: *httpz.Response, _: AppState) !void {
    try res.json(.{ 
        .@"error" = "Endpoint not found",
        .message = "The requested resource does not exist"
    }, .{ .status = 404 });
}

fn uncaughtErrorHandler(req: *httpz.Request, res: *httpz.Response, err: anyerror, _: AppState) void {
    std.log.err("Uncaught error on {s}: {any}", .{ req.url.path, err });
    
    res.status = 500;
    res.json(.{ 
        .@"error" = "Internal server error",
        .request_id = generateRequestId() // Optional: add request tracing
    }, .{}) catch {};
}
```

### Middleware (When Available)

```zig
// Logging middleware example structure
const LoggingMiddleware = struct {
    const Self = @This();
    
    pub fn init(allocator: std.mem.Allocator, config: anytype) !Self {
        return Self{};
    }
    
    pub fn execute(self: *Self, req: *httpz.Request, res: *httpz.Response, ctx: anytype) !void {
        const start_time = std.time.milliTimestamp();
        
        // Log request
        std.log.info("Request: {s} {s}", .{ req.method, req.url.path });
        
        // Continue to next handler
        try ctx.next();
        
        // Log response
        const duration = std.time.milliTimestamp() - start_time;
        std.log.info("Response: {} - {}ms", .{ res.status, duration });
    }
};

// Apply middleware
const logger = try server.middleware(LoggingMiddleware, .{});
router.middlewares = &.{logger};
```

### CORS Support

```zig
fn corsHandler(req: *httpz.Request, res: *httpz.Response) !void {
    // Handle preflight requests
    if (std.mem.eql(u8, req.method, "OPTIONS")) {
        res.headers.add("Access-Control-Allow-Origin", "*");
        res.headers.add("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
        res.headers.add("Access-Control-Allow-Headers", "Content-Type, Authorization");
        res.headers.add("Access-Control-Max-Age", "86400");
        res.status = 204;
        return;
    }
    
    // Add CORS headers to all responses
    res.headers.add("Access-Control-Allow-Origin", "*");
    res.headers.add("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
}

// Apply to all routes
router.all("/*", corsHandler, .{});
```

---

## Database Integration Patterns

### Connection Management

```zig
const AppState = struct {
    db_pool: *database.Pool,
    
    // Get database connection for request
    pub fn getDb(self: *AppState) !*database.Connection {
        return try self.db_pool.acquire();
    }
    
    pub fn releaseDb(self: *AppState, conn: *database.Connection) void {
        self.db_pool.release(conn);
    }
};

fn getPlayer(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const player_id_str = req.param("id") orelse {
        try res.json(.{ .@"error" = "Player ID required" }, .{ .status = 400 });
        return;
    };
    
    const player_id = std.fmt.parseInt(u32, player_id_str, 10) catch {
        try res.json(.{ .@"error" = "Invalid player ID" }, .{ .status = 400 });
        return;
    };
    
    var conn = app_state.getDb() catch {
        try res.json(.{ .@"error" = "Database connection failed" }, .{ .status = 500 });
        return;
    };
    defer app_state.releaseDb(conn);
    
    const player = conn.getPlayer(player_id) catch |err| switch (err) {
        error.PlayerNotFound => {
            try res.json(.{ .@"error" = "Player not found" }, .{ .status = 404 });
            return;
        },
        else => {
            std.log.err("Database error: {any}", .{err});
            try res.json(.{ .@"error" = "Database error" }, .{ .status = 500 });
            return;
        },
    };
    
    try res.json(player, .{});
}
```

### Transaction Patterns

```zig
fn createPlayerWithStats(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const input = try parsePlayerInput(req, res);
    if (input == null) return; // Error already sent
    
    var conn = app_state.getDb() catch {
        try res.json(.{ .@"error" = "Database connection failed" }, .{ .status = 500 });
        return;
    };
    defer app_state.releaseDb(conn);
    
    // Start transaction
    try conn.begin();
    errdefer conn.rollback() catch {};
    
    // Create player
    const player_id = conn.createPlayer(input.?.player_data) catch |err| {
        std.log.err("Failed to create player: {any}", .{err});
        try res.json(.{ .@"error" = "Failed to create player" }, .{ .status = 500 });
        return;
    };
    
    // Create initial stats
    conn.createPlayerStats(player_id, input.?.stats_data) catch |err| {
        std.log.err("Failed to create player stats: {any}", .{err});
        try res.json(.{ .@"error" = "Failed to create player stats" }, .{ .status = 500 });
        return;
    };
    
    // Commit transaction
    try conn.commit();
    
    try res.json(.{ .id = player_id, .message = "Player created successfully" }, .{ .status = 201 });
}
```

### Repository Pattern Integration

```zig
// Define repositories in app state
const AppState = struct {
    players: *PlayerRepository,
    teams: *TeamRepository,
    stats: *StatsRepository,
};

fn getPlayerStats(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const player_id = parsePlayerId(req, res) orelse return;
    const season = req.query("season");
    const week = req.query("week");
    
    const stats = app_state.stats.getPlayerStats(.{
        .player_id = player_id,
        .season = if (season) |s| std.fmt.parseInt(u16, s, 10) catch null else null,
        .week = if (week) |w| std.fmt.parseInt(u8, w, 10) catch null else null,
    }) catch |err| switch (err) {
        error.PlayerNotFound => {
            try res.json(.{ .@"error" = "Player not found" }, .{ .status = 404 });
            return;
        },
        else => {
            std.log.err("Failed to get player stats: {any}", .{err});
            try res.json(.{ .@"error" = "Internal error" }, .{ .status = 500 });
            return;
        },
    };
    
    try res.json(.{ .data = stats }, .{});
}
```

---

## Performance & Memory Management

### Memory Allocation Patterns

```zig
fn efficientHandler(req: *httpz.Request, res: *httpz.Response) !void {
    // Use request arena for temporary allocations
    // Memory is automatically freed after request
    var results = std.ArrayList(Player).init(req.arena);
    
    // Avoid allocations in hot paths when possible
    var buffer: [1024]u8 = undefined;
    const formatted = try std.fmt.bufPrint(buffer[0..], "Player ID: {d}", .{player_id});
    
    try res.json(.{ .message = formatted }, .{});
}
```

### Connection Pooling Best Practices

```zig
const PoolConfig = struct {
    min_connections: u32 = 5,
    max_connections: u32 = 20,
    acquire_timeout_ms: u32 = 5000,
    idle_timeout_ms: u32 = 300000, // 5 minutes
};

// Always use defer for connection cleanup
fn handlerWithDb(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    var conn = app_state.db_pool.acquire() catch {
        try res.json(.{ .@"error" = "Service temporarily unavailable" }, .{ .status = 503 });
        return;
    };
    defer app_state.db_pool.release(conn);
    
    // Use connection...
}
```

### Caching Strategies

```zig
const AppState = struct {
    cache: std.HashMap([]const u8, CacheEntry),
    cache_mutex: std.Thread.Mutex = .{},
    
    pub fn getCachedPlayer(self: *AppState, player_id: u32) ?Player {
        self.cache_mutex.lock();
        defer self.cache_mutex.unlock();
        
        var buffer: [32]u8 = undefined;
        const key = std.fmt.bufPrint(buffer[0..], "player:{d}", .{player_id}) catch return null;
        
        if (self.cache.get(key)) |entry| {
            if (entry.isValid()) {
                return entry.value;
            }
        }
        return null;
    }
};
```

---

## NFL Stats API Examples

### Players Endpoint Implementation

```zig
// GET /api/players?position=QB&team=NE&limit=20&offset=0
fn getPlayers(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    // Parse query parameters
    const position = req.query("position");
    const team = req.query("team");
    const limit = blk: {
        const limit_str = req.query("limit") orelse "20";
        break :blk std.fmt.parseInt(u32, limit_str, 10) catch 20;
    };
    const offset = blk: {
        const offset_str = req.query("offset") orelse "0";
        break :blk std.fmt.parseInt(u32, offset_str, 10) catch 0;
    };
    
    // Validate limits
    if (limit > 100) {
        try res.json(.{ .@"error" = "Limit cannot exceed 100" }, .{ .status = 400 });
        return;
    }
    
    // Build query
    const filters = PlayerFilters{
        .position = position,
        .team = team,
        .limit = limit,
        .offset = offset,
    };
    
    // Get data from repository
    const result = app_state.players.findMany(filters) catch |err| {
        std.log.err("Failed to get players: {any}", .{err});
        try res.json(.{ .@"error" = "Internal error" }, .{ .status = 500 });
        return;
    };
    
    // Return paginated response
    try res.json(.{
        .data = result.players,
        .pagination = .{
            .limit = limit,
            .offset = offset,
            .total = result.total_count,
            .has_more = (offset + limit) < result.total_count,
        },
    }, .{});
}

// GET /api/players/:id
fn getPlayer(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const player_id = parsePlayerId(req, res) orelse return;
    
    const player = app_state.players.findById(player_id) catch |err| switch (err) {
        error.PlayerNotFound => {
            try res.json(.{ .@"error" = "Player not found" }, .{ .status = 404 });
            return;
        },
        else => {
            std.log.err("Failed to get player {d}: {any}", .{ player_id, err });
            try res.json(.{ .@"error" = "Internal error" }, .{ .status = 500 });
            return;
        },
    };
    
    try res.json(.{ .data = player }, .{});
}

// GET /api/players/:id/stats?season=2024&week=1
fn getPlayerStats(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const player_id = parsePlayerId(req, res) orelse return;
    
    // Parse optional filters
    const season = blk: {
        const season_str = req.query("season") orelse break :blk null;
        break :blk std.fmt.parseInt(u16, season_str, 10) catch {
            try res.json(.{ .@"error" = "Invalid season format" }, .{ .status = 400 });
            return;
        };
    };
    
    const week = blk: {
        const week_str = req.query("week") orelse break :blk null;
        break :blk std.fmt.parseInt(u8, week_str, 10) catch {
            try res.json(.{ .@"error" = "Invalid week format" }, .{ .status = 400 });
            return;
        };
    };
    
    const stats = app_state.stats.getPlayerStats(.{
        .player_id = player_id,
        .season = season,
        .week = week,
    }) catch |err| switch (err) {
        error.PlayerNotFound => {
            try res.json(.{ .@"error" = "Player not found" }, .{ .status = 404 });
            return;
        },
        else => {
            std.log.err("Failed to get stats for player {d}: {any}", .{ player_id, err });
            try res.json(.{ .@"error" = "Internal error" }, .{ .status = 500 });
            return;
        },
    };
    
    try res.json(.{ .data = stats }, .{});
}
```

### Teams Endpoint Implementation

```zig
// GET /api/teams
fn getTeams(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const season = blk: {
        const season_str = req.query("season") orelse break :blk null;
        break :blk std.fmt.parseInt(u16, season_str, 10) catch null;
    };
    
    const teams = app_state.teams.findAll(.{ .season = season }) catch |err| {
        std.log.err("Failed to get teams: {any}", .{err});
        try res.json(.{ .@"error" = "Internal error" }, .{ .status = 500 });
        return;
    };
    
    try res.json(.{ .data = teams }, .{});
}

// GET /api/teams/:id/roster?season=2024
fn getTeamRoster(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const team_id = parseTeamId(req, res) orelse return;
    const season = blk: {
        const season_str = req.query("season") orelse "2024";
        break :blk std.fmt.parseInt(u16, season_str, 10) catch {
            try res.json(.{ .@"error" = "Invalid season format" }, .{ .status = 400 });
            return;
        };
    };
    
    const roster = app_state.players.findByTeam(team_id, season) catch |err| switch (err) {
        error.TeamNotFound => {
            try res.json(.{ .@"error" = "Team not found" }, .{ .status = 404 });
            return;
        },
        else => {
            std.log.err("Failed to get team roster: {any}", .{err});
            try res.json(.{ .@"error" = "Internal error" }, .{ .status = 500 });
            return;
        },
    };
    
    try res.json(.{ .data = roster }, .{});
}
```

### Helper Functions

```zig
fn parsePlayerId(req: *httpz.Request, res: *httpz.Response) ?u32 {
    const player_id_str = req.param("id") orelse {
        res.json(.{ .@"error" = "Player ID required" }, .{ .status = 400 }) catch {};
        return null;
    };
    
    return std.fmt.parseInt(u32, player_id_str, 10) catch {
        res.json(.{ .@"error" = "Invalid player ID format" }, .{ .status = 400 }) catch {};
        return null;
    };
}

fn parseTeamId(req: *httpz.Request, res: *httpz.Response) ?u32 {
    const team_id_str = req.param("id") orelse {
        res.json(.{ .@"error" = "Team ID required" }, .{ .status = 400 }) catch {};
        return null;
    };
    
    return std.fmt.parseInt(u32, team_id_str, 10) catch {
        res.json(.{ .@"error" = "Invalid team ID format" }, .{ .status = 400 }) catch {};
        return null;
    };
}
```

---

## Troubleshooting

### Common Issues

#### 1. Server Won't Start

```bash
# Check if port is already in use
netstat -tlnp | grep :8080
lsof -i :8080

# Use different port
var server = try httpz.Server(void).init(allocator, .{ .port = 8081 }, {});
```

#### 2. JSON Parsing Errors

```zig
// Always handle JSON parsing errors
const input = std.json.parseFromSlice(PlayerData, req.arena, body_bytes, .{}) catch |err| {
    std.log.err("JSON parse error: {any}", .{err});
    try res.json(.{ 
        .@"error" = "Invalid JSON format",
        .details = "Check request body syntax"
    }, .{ .status = 400 });
    return;
};
```

#### 3. Memory Issues

```zig
// Don't store request arena references beyond handler scope
fn badHandler(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    // DON'T DO THIS - arena memory will be freed
    const data = try req.arena.dupe(u8, "some data");
    app_state.stored_data = data; // This will become invalid!
}

// Instead, use application allocator for persistent data
fn goodHandler(req: *httpz.Request, res: *httpz.Response, app_state: AppState) !void {
    const data = try app_state.allocator.dupe(u8, "some data");
    app_state.stored_data = data; // Safe to store
}
```

#### 4. Database Connection Issues

```zig
// Always handle connection errors gracefully
var conn = app_state.db_pool.acquire() catch |err| {
    std.log.err("Database connection failed: {any}", .{err});
    try res.json(.{ 
        .@"error" = "Service temporarily unavailable",
        .retry_after = 30 
    }, .{ .status = 503 });
    return;
};
defer app_state.db_pool.release(conn);
```

### Debugging Techniques

#### Request Logging

```zig
fn loggedHandler(req: *httpz.Request, res: *httpz.Response) !void {
    const start_time = std.time.microTimestamp();
    
    std.log.info("Request: {s} {s}", .{ req.method, req.url.path });
    if (req.query_string.len > 0) {
        std.log.info("Query: {s}", .{req.query_string});
    }
    
    // Handle request...
    
    const duration = std.time.microTimestamp() - start_time;
    std.log.info("Response: {} - {d}μs", .{ res.status, duration });
}
```

#### Error Context

```zig
fn contextualErrorHandler(req: *httpz.Request, res: *httpz.Response, err: anyerror) void {
    const request_id = generateRequestId();
    
    std.log.err("Error {s} on {s} {s} - ID: {s}", .{ 
        @errorName(err), 
        req.method, 
        req.url.path,
        request_id 
    });
    
    res.status = 500;
    res.json(.{ 
        .@"error" = "Internal server error",
        .request_id = request_id  // Help users report issues
    }, .{}) catch {};
}
```

### Performance Monitoring

```zig
const RequestMetrics = struct {
    total_requests: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    error_count: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    avg_response_time: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    
    pub fn recordRequest(self: *RequestMetrics, duration_us: u64, is_error: bool) void {
        _ = self.total_requests.fetchAdd(1, .monotonic);
        if (is_error) {
            _ = self.error_count.fetchAdd(1, .monotonic);
        }
        
        // Simple moving average (basic implementation)
        const current_avg = self.avg_response_time.load(.monotonic);
        const new_avg = (current_avg + duration_us) / 2;
        _ = self.avg_response_time.compareAndSwap(current_avg, new_avg, .monotonic, .monotonic);
    }
};
```

---

## Integration with Build System

### build.zig Configuration

```zig
// Ensure httpz dependency is properly configured
const httpz = b.dependency("httpz", .{
    .target = target,
    .optimize = optimize,
});

// Add to executable imports
.imports = &.{
    .{ .name = "httpz", .module = httpz.module("httpz") },
    // ... other imports
},
```

### Development vs Production

```zig
// Development configuration
const is_development = (optimize == .Debug);

var server = try httpz.Server(AppState).init(allocator, .{
    .port = if (is_development) 8080 else 80,
    .address = if (is_development) "127.0.0.1" else "0.0.0.0",
    .timeout_request = if (is_development) 30000 else 10000,
}, app_state);
```

---

*This guide covers the essential patterns for using httpz effectively in the NFL Statistics Database project. Refer to specific sections when implementing new endpoints or debugging issues.*