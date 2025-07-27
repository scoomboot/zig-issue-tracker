# Ready to Work Issues

**Current Focus**: Issues with no dependencies or completed dependencies that can be started immediately.

## 🟢 No Dependencies - Start Immediately

### Critical Priority (Memory Safety Issues)

*No critical memory safety issues at this time.*

**Note**: All memory safety issues (#050-#054) have been resolved from a code perspective. Issue #053 remains flagged by zig-tooling due to a tooling limitation with ownership transfer patterns, but the memory management is correct and safe.

### High Priority

- **#001**: Implement CSV parsing for player statistics
  - **Component**: src/csv_parser.zig (to be created)
  - **Status**: In Progress
  - **Dependencies**: None
  - **Details**: Parse offensive, defensive, and kicking stats from CSV files into structured data

- **#034**: Implement comprehensive server configuration
  - **Component**: src/http/config.zig, src/http/server.zig
  - **Dependencies**: None (Phase 1 completed)
  - **Details**: Add timeout configuration, request limits, buffer optimization


- **#057**: Add comprehensive HTTP module test coverage
  - **Component**: tests/http/, src/http/
  - **Dependencies**: None
  - **Details**: Create test coverage for 8 HTTP modules with missing tests

### Low Priority

- **#014**: Document SQLite3 development library prerequisites
  - **Component**: Documentation (README.md, setup guides)
  - **Dependencies**: None
  - **Details**: Ensure SQLite3 development libraries are clearly documented as prerequisite

## 🟢 All Dependencies Completed - Ready to Start

### High Priority

- **#013**: Set up connection pool management
  - **Component**: src/database/pool.zig
  - **Dependencies**: Issue #010 (✅ completed)
  - **Details**: Pool config (min: 5, max: 20), connection lifecycle, performance pragmas, health checks
  - **Next**: Enables database middleware (#026) and repository pattern (#020)

### Medium Priority

- **#058**: Integrate zig-tooling checks into CI pipeline
  - **Component**: CI configuration, build.zig
  - **Dependencies**: Issue #055 (✅ completed)
  - **Details**: Add automated zig-tooling checks to build pipeline with quality gates and failure reporting
  - **Next**: Enables automated code quality enforcement


## 🔄 Next Wave (1 Dependency Away)

*Issues that become available after completing current work*

### After #001 (CSV Parser) - Needs completion
- **#015**: Implement streaming CSV parser with performance optimizations [#011/#012 ✅ completed]

### After #034 (HTTP Config)
- **#035**: Add environment-based configuration
- **#037**: Implement custom error handlers
- **#041**: Implement CORS support

### After #013 (Pool) - Needs completion  
- **#020**: Implement generic repository pattern (also needs #011 ✅)

## 📊 Priority Recommendations

### Start These Now (High Priority)
1. **Continue #001** - CSV parsing foundation (in progress)

### High Priority (Parallel Development)
2. **Start #034** - HTTP server config (independent track)
3. **Start #057** - HTTP test coverage (fills major testing gap)

### Quick Wins
4. **Complete #013** - Connection pool (enables repository layer)
5. **Address #014** - Documentation (low effort)
6. **Consider #058** - CI integration (medium priority, now available)

### Impact Chain
- **#001** → #015 → #016/#017 (import pipeline) [#012 ✅ completed]
- **#034** → #035/#037/#041 (HTTP features)
- **#013** → #020 → #021-#025 (repository layer)
- **#055** ✅ → #058 (CI integration now available)

---

*This file tracks issues from ISSUES.md that are immediately actionable. Check dependencies before starting work.*