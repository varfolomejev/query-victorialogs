# Query VictoriaLogs - Codex Submission

## Skill Overview

**Name:** query-victorialogs
**Version:** 1.0.0
**Author:** ihorvarfolomeiev
**License:** MIT
**Status:** Stable, Production-Ready

## Description

A Claude Code skill that enables natural language querying and analysis of logs from VictoriaLogs instances using LogsQL via HTTP API. This skill provides safe, intelligent log investigation capabilities with built-in safeguards to prevent server overload.

## Key Features

### 1. **Safe Time Range Management**
- Enforces maximum 3-hour query windows to prevent server crashes
- Automatic two-step approach for date-based searches (hits → query)
- Default to recent time ranges (5m-1h) with explicit `_time` filters

### 2. **Full LogsQL Support**
- **Filters:** word search, phrase search, regex, field filters, stream filters, time filters
- **Pipes:** fields, unpack_json, stats, sort, limit, filter, top, extract_regexp
- **Stats Functions:** count, sum, avg, min, max, median, quantile, rate, count_uniq, uniq_values

### 3. **Multi-Environment Support**
- DEV/PROD environment switching via project CLAUDE.md
- Automatic environment selection (defaults to DEV)
- Per-project configuration

### 4. **Comprehensive HTTP API Coverage**
- `/select/logsql/query` - Fetch log entries
- `/select/logsql/hits` - Get hit counts/histograms
- `/select/logsql/stats_query` - Aggregate statistics
- `/select/logsql/field_names` - Discover available fields
- `/select/logsql/field_values` - Explore field values
- `/select/logsql/streams` - List log streams

## Use Cases

### Development & Debugging
- "search logs for errors in last 5 minutes"
- "find logs matching user ID u-12345"
- "tail dev logs for connection refused"

### Incident Investigation
- "check what happened on March 20 around the deployment"
- "find all timeout errors in the last hour"
- "show me authentication failures in prod"

### Performance Analysis
- "get error rate per service in the last hour"
- "what are the slowest API endpoints today"
- "count 500 errors by service"

### Discovery & Exploration
- "what log streams are available"
- "show me all field names in recent logs"
- "what values does the 'level' field have"

## Technical Architecture

### File Structure
```
query-victorialogs/
├── skill.json              # Metadata for CCPM registry (Claude Code)
├── .skillfish.json         # Metadata for Codex compatibility
├── SKILL.md                # Main prompt instructions
├── README.md               # User-facing documentation
├── LICENSE                 # MIT license
├── CODEX.md                # This file - codex submission
├── references/
│   ├── logsql-reference.md # Complete LogsQL syntax reference
│   └── http-api.md         # HTTP API endpoint documentation
└── scripts/
    ├── vl-query.sh         # Helper script for queries
    └── vl-hits.sh          # Helper script for hit counts
```

### Dependencies
- **External:** curl (standard on all Unix-like systems)
- **Environment:** VL_DEV_URL, VL_PROD_URL (configured per-project)

### Configuration

Users add VictoriaLogs endpoints to their project's `CLAUDE.md`:

```markdown
## VictoriaLogs

- **DEV**: `http://<your-dev-host>:9428`
- **PROD**: `http://<your-prod-host>:9428`

Default to DEV unless user explicitly says "prod" or "production".
```

## Safety & Best Practices

### Critical Safety Features
1. **Time Range Limits:** Maximum 3-hour query window enforced
2. **Explicit Time Filters:** All queries must include `_time` filter
3. **Two-Step Date Search:** Uses hits endpoint first to locate relevant time windows
4. **Default to Recent:** Defaults to short time ranges (5m-1h) when unspecified

### Query Optimization
- Places fastest filters (word, phrase) first
- Uses slowest filters (regexp) last
- Includes reasonable `limit` values (100-500)
- Leverages `| collapse_nums` to group similar messages
- Uses `| unpack_json` for structured logs

## Installation

### Via CCPM (Recommended)
```bash
ccpm install ihorvarfolomeiev/query-victorialogs
```

### Manual Installation
```bash
git clone https://github.com/ihorvarfolomeiev/query-victorialogs.git \
  ~/.claude/skills/query-victorialogs
```

## Real-World Usage Examples

### Example 1: Error Investigation
**User:** "search logs for errors in last 5 minutes"

**Claude executes:**
```bash
curl -s "$VL_DEV_URL/select/logsql/query" \
  -H 'Accept: application/stream+json' \
  -H 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
  --data-urlencode "query=_time:5m error" \
  -d 'limit=500' \
  --insecure
```

### Example 2: Date-Based Search (Unknown Time)
**User:** "check what happened on March 20"

**Step 1 - Claude finds time windows:**
```bash
curl -s "$VL_DEV_URL/select/logsql/hits" \
  -H 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
  --data-urlencode "query=*" \
  -d 'step=3600000ms' \
  -d 'start=2026-03-20T00:00:00Z' \
  -d 'end=2026-03-20T23:59:59Z' \
  --insecure
```

**Step 2 - Claude queries relevant hours:**
```bash
curl -s "$VL_DEV_URL/select/logsql/query" \
  -H 'Accept: application/stream+json' \
  -H 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
  --data-urlencode "query=*" \
  -d 'limit=500' \
  -d 'start=2026-03-20T14:00:00Z' \
  -d 'end=2026-03-20T16:00:00Z' \
  --insecure
```

### Example 3: Aggregation & Stats
**User:** "get error rate per service in the last hour"

**Claude executes:**
```bash
curl -s "$VL_DEV_URL/select/logsql/stats_query" \
  -H 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
  --data-urlencode "query=_time:1h error | stats by (service) count() as error_count | sort by (error_count desc)" \
  --insecure
```

## Quality Assurance

### Testing Checklist
- [x] Basic query functionality
- [x] Time range enforcement (3-hour max)
- [x] Two-step date search (hits → query)
- [x] Multi-environment switching (DEV/PROD)
- [x] All HTTP API endpoints
- [x] LogsQL filters and pipes
- [x] Stats functions
- [x] Field/stream discovery
- [x] Error handling
- [x] Documentation completeness

### Known Limitations
1. **SSL Verification Disabled:** Uses `--insecure` flag due to common VictoriaLogs deployment patterns
2. **Time Range Restriction:** Maximum 3 hours per query (intentional safety limit)
3. **Server Dependency:** Requires accessible VictoriaLogs instance

## Comparison to Alternatives

### vs. Manual curl Commands
- ✅ Natural language interface
- ✅ Built-in safety limits
- ✅ Automatic two-step date search
- ✅ Environment management
- ✅ LogsQL syntax reference always available

### vs. VictoriaLogs Web UI
- ✅ Integrated into development workflow
- ✅ Scriptable and repeatable
- ✅ Context-aware (knows your project setup)
- ✅ Can chain with other Claude Code skills

### vs. Other Log Tools (Elasticsearch, Loki, etc.)
- ⚡ Optimized specifically for VictoriaLogs
- ⚡ LogsQL-native (not generic query language)
- ⚡ Lightweight (no heavy client installations)

## Target Users

### Primary Audience
- **DevOps Engineers** managing VictoriaLogs deployments
- **Backend Developers** debugging microservices
- **SREs** investigating incidents
- **QA Engineers** validating log output

### Prerequisites
- Access to VictoriaLogs instance (DEV or PROD)
- Basic understanding of log investigation concepts
- Claude Code CLI installed

## Support & Contribution

### Documentation
- Comprehensive README.md
- Inline examples in SKILL.md
- Full LogsQL reference included
- HTTP API documentation included

### Community
- GitHub Issues: https://github.com/ihorvarfolomeiev/query-victorialogs/issues
- Pull Requests welcome
- MIT License (permissive, commercial-friendly)

## Maintenance Commitment

- **Status:** Actively maintained
- **Updates:** Security patches within 48h
- **Feature Requests:** Reviewed weekly
- **Breaking Changes:** Will follow semantic versioning

## Marketing Summary

**One-liner:** Natural language log querying for VictoriaLogs with built-in safety and intelligence.

**Value Proposition:** Stop fighting with curl commands and complex LogsQL syntax. Ask Claude Code in plain English to investigate your logs, and get intelligent, safe queries that won't crash your server.

**Differentiator:** The only Claude Code skill with two-step date search and built-in time range protection for VictoriaLogs.

## Publication Checklist

- [x] skill.json with complete metadata
- [x] SKILL.md with detailed instructions
- [x] README.md with user documentation
- [x] LICENSE (MIT)
- [x] CODEX.md (this file)
- [x] Reference documentation
- [x] Helper scripts
- [x] Real-world examples
- [x] Safety documentation
- [x] Installation instructions
- [x] Configuration guide
- [ ] GitHub repository published
- [ ] CCPM registry submission
- [ ] Version 1.0.0 git tag

## Next Steps for Publication

1. **Create GitHub Repository**
   ```bash
   cd /Users/ihorvarfolomeiev/.claude/skills/query-victorialogs
   git remote add origin https://github.com/ihorvarfolomeiev/query-victorialogs.git
   git push -u origin main
   ```

2. **Create Release Tag**
   ```bash
   git tag -a v1.0.0 -m "Initial release - VictoriaLogs query skill"
   git push origin v1.0.0
   ```

3. **Submit to CCPM Registry**
   - Follow CCPM submission guidelines
   - Include skill.json in submission
   - Reference GitHub repository

4. **Announce**
   - Claude Code community channels
   - VictoriaLogs community
   - DevOps communities

---

**Ready for Codex:** ✅ Yes
**Recommended:** Suitable for public registry
**Quality Rating:** Production-ready, fully documented, tested
