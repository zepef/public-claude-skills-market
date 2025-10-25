# MCP Response Analyzer - Workflow Integration

Complete integration guide for using this skill in your Claude Code sessions to optimize token usage.

## Quick Start

### 1. Detect Large MCP Response
When you see a warning like this:
```
⚠ Large MCP response (~10.1k tokens), this can fill up context quickly
```

### 2. Write Response to File
Immediately write the MCP response to a temporary file:
```bash
# Generic pattern
cat > /tmp/mcp_<tool_name>_$(date +%Y%m%d_%H%M%S).json << 'EOF'
<paste MCP response here>
EOF
```

### 3. Invoke Skill
Use the skill to analyze the file instead of keeping full response in context:
```bash
cat /tmp/mcp_<tool_name>_*.json | jq '<filter expression>'
```

### 4. Get Compact Summary
Receive 300-500 token summary instead of 10k+ token full response.

---

## Integration Patterns

### Pattern A: Single MCP Call with Large Response

**Scenario**: One MCP tool returns >5k tokens

```bash
# Step 1: Call MCP tool (response stored in variable)
RESPONSE=$(mcp__vercel__list_deployments \
  --projectId="prj_k9FgC5lQe8fsLsinMuk3Nnyce68V" \
  --teamId="team_3QFAMGjgOA1LB6ZrETFKRBBD")

# Step 2: Write to file
echo "$RESPONSE" > /tmp/mcp_deployments_$(date +%Y%m%d).json

# Step 3: Analyze with skill
cat /tmp/mcp_deployments_*.json | jq '.deployments.deployments[] | select(.state == "ERROR")'

# Step 4: Extract insights
# - Found 2 ERROR deployments
# - Need to investigate with get_deployment_build_logs()
# - Token savings: 9,800 (97%)
```

**Token Impact**:
- Without skill: 10,100 tokens consumed
- With skill: 300 tokens consumed
- **Savings: 9,800 tokens (97%)**

---

### Pattern B: Multiple MCP Calls Requiring Correlation

**Scenario**: Need to correlate data across multiple MCP calls

```bash
# Step 1: Get all deployments
mcp__vercel__list_deployments > /tmp/mcp_deployments.json

# Step 2: Get all logs
mcp__render__list_logs > /tmp/mcp_logs.json

# Step 3: Filter deployments for failed ones
FAILED_IDS=$(cat /tmp/mcp_deployments.json | jq -r '.deployments.deployments[] | select(.state == "ERROR") | .uid')

# Step 4: Correlate with logs by timestamp
for id in $FAILED_IDS; do
  CREATED=$(cat /tmp/mcp_deployments.json | jq -r --arg id "$id" '.deployments.deployments[] | select(.uid == $id) | .created')
  cat /tmp/mcp_logs.json | jq --argjson time "$CREATED" '.logs[] | select(.timestamp > ($time - 300000) and .timestamp < ($time + 300000))'
done

# Step 5: Summarize findings
# - Deployment dpl_72PE8... failed at 14:30:00
# - Logs show database connection errors at 14:29:50
# - Root cause identified
```

**Token Impact**:
- Without skill: 35,100 tokens (10k + 25k)
- With skill: 800 tokens
- **Savings: 34,300 tokens (98%)**

---

### Pattern C: Iterative Investigation Workflow

**Scenario**: Debugging requires multiple rounds of filtering

```bash
# Round 1: Broad filter - all production deployments
cat /tmp/mcp_deployments.json | jq '[.deployments.deployments[] | select(.target == "production")] | length'
# Output: 8 production deployments

# Round 2: Narrow down - production failures
cat /tmp/mcp_deployments.json | jq '[.deployments.deployments[] | select(.target == "production" and .state == "ERROR")] | length'
# Output: 2 failures

# Round 3: Time range - failures in last 24 hours
NOW=$(date +%s)000  # milliseconds
cat /tmp/mcp_deployments.json | jq --argjson now "$NOW" '[.deployments.deployments[] | select(.target == "production" and .state == "ERROR" and .created > ($now - 86400000))] | length'
# Output: 1 recent failure

# Round 4: Extract details
cat /tmp/mcp_deployments.json | jq --argjson now "$NOW" '[.deployments.deployments[] | select(.target == "production" and .state == "ERROR" and .created > ($now - 86400000))] | .[] | {id: .uid, url, commit: .meta.githubCommitMessage}'
# Output: Full details of the 1 recent failure
```

**Token Impact**:
- Without skill: 10,100 tokens × 4 rounds = 40,400 tokens
- With skill: 300 tokens total (file read once)
- **Savings: 40,100 tokens (99%)**

---

## MCP Tool-Specific Workflows

### Vercel Deployment Investigation

```bash
# 1. Get deployments
mcp__vercel__list_deployments > /tmp/vercel_deployments.json

# 2. Find errors
ERROR_IDS=$(cat /tmp/vercel_deployments.json | jq -r '.deployments.deployments[] | select(.state == "ERROR") | .uid')

# 3. Get build logs for each error
for id in $ERROR_IDS; do
  echo "=== $id ===" >> /tmp/build_logs.txt
  mcp__vercel__get_deployment_build_logs --id="$id" >> /tmp/build_logs.txt
done

# 4. Analyze build logs
cat /tmp/build_logs.txt | grep -i "error\|failed\|exception"

# 5. Summarize
echo "Found $(echo $ERROR_IDS | wc -w) failed deployments"
```

### Render Service Monitoring

```bash
# 1. Get all services
mcp__render__list_services > /tmp/render_services.json

# 2. Filter by status
cat /tmp/render_services.json | jq '[.services[] | select(.status != "RUNNING")] | .[] | {id, name, status}'

# 3. Get logs for non-running services
NON_RUNNING=$(cat /tmp/render_services.json | jq -r '.services[] | select(.status != "RUNNING") | .id')

for service in $NON_RUNNING; do
  mcp__render__list_logs --resource="$service" > "/tmp/logs_${service}.json"
  cat "/tmp/logs_${service}.json" | jq '[.logs[] | select(.level == "ERROR")] | .[:5]'
done
```

### Supabase Database Analysis

```bash
# 1. List all projects
mcp__supabase__list_projects > /tmp/supabase_projects.json

# 2. Filter by region
cat /tmp/supabase_projects.json | jq '[.projects[] | select(.region == "us-east-1")] | .[] | {id, name, status}'

# 3. Get tables for specific project
PROJECT_ID=$(cat /tmp/supabase_projects.json | jq -r '.projects[0].id')
mcp__supabase__list_tables --projectId="$PROJECT_ID" > /tmp/supabase_tables.json

# 4. Analyze table sizes
cat /tmp/supabase_tables.json | jq '[.tables[]] | sort_by(.size) | reverse | .[:10] | .[] | {name, size, schema}'
```

---

## Session Lifecycle Integration

### Session Start
```bash
# Check for existing temp files from previous sessions
ls -lh /tmp/mcp_*.json

# Review what was being investigated
cat /tmp/mcp_deployments_*.json | jq '.deployments.deployments[] | select(.state == "ERROR") | {id: .uid, created}'
```

### During Session
```bash
# Write large responses to files
mcp_tool_call > /tmp/mcp_$(date +%s).json

# Analyze incrementally
cat /tmp/mcp_*.json | jq '<progressive filters>'

# Keep context clean - only load filtered results
```

### Session End
```bash
# Optional: Clean up temp files
rm /tmp/mcp_*.json

# Or: Keep for next session
# Archive important analyses
mv /tmp/mcp_deployments_*.json ~/analyses/deployment_investigation_$(date +%Y%m%d).json
```

---

## Token Budget Management

### Strategy 1: Pre-emptive File Writing
**Rule**: If MCP response might be >5k tokens, write to file immediately

```bash
# Before calling MCP tool, estimate response size
# Deployments: ~500 tokens each
# Logs: ~250 tokens each
# Projects: ~300 tokens each

# If count × token_size > 5000, use file strategy
if [ $DEPLOYMENT_COUNT -gt 10 ]; then
  mcp__vercel__list_deployments > /tmp/deployments.json
else
  mcp__vercel__list_deployments  # Load directly
fi
```

### Strategy 2: Context Window Monitoring
```bash
# Check context usage
# If > 75%, aggressively use file strategy

# Green zone (0-75%): Normal operations
# Yellow zone (75-85%): Start using files for >3k responses
# Red zone (85%+): Use files for ALL MCP calls
```

### Strategy 3: Incremental Loading
```bash
# Load only what's needed, when needed

# Instead of loading all 20 deployments:
cat /tmp/deployments.json | jq '.deployments.deployments | length'  # Just count

# When specific deployment needed:
cat /tmp/deployments.json | jq --arg id "$TARGET_ID" '.deployments.deployments[] | select(.uid == $id)'  # Load one
```

---

## Error Handling

### File Not Found
```bash
# Check if file exists before reading
if [ -f /tmp/mcp_deployments.json ]; then
  cat /tmp/mcp_deployments.json | jq '.deployments.deployments[] | select(.state == "ERROR")'
else
  echo "Error: MCP response file not found. Re-run MCP tool and save to file."
fi
```

### Invalid JSON
```bash
# Validate JSON before processing
if jq empty /tmp/mcp_response.json 2>/dev/null; then
  echo "Valid JSON"
else
  echo "Invalid JSON - check file format"
  cat /tmp/mcp_response.json | head -20  # Debug
fi
```

### Empty Results
```bash
# Handle no matches gracefully
RESULTS=$(cat /tmp/mcp_deployments.json | jq '[.deployments.deployments[] | select(.state == "ERROR")] | length')

if [ "$RESULTS" -eq 0 ]; then
  echo "No ERROR deployments found. Available states:"
  cat /tmp/mcp_deployments.json | jq '[.deployments.deployments[].state] | unique'
fi
```

### Missing Fields
```bash
# Check schema before filtering
FIELDS=$(cat /tmp/mcp_response.json | jq 'keys')
echo "Available fields: $FIELDS"

# Use alternative field names if needed
cat /tmp/mcp_response.json | jq '.items // .results // .data'
```

---

## Performance Optimization

### Parallel Processing
```bash
# Process multiple files in parallel
for file in /tmp/mcp_*.json; do
  (cat "$file" | jq '<filter>' > "${file}.filtered") &
done
wait
```

### Streaming Large Files
```bash
# For very large JSON files (>100MB)
jq -c '.items[]' /tmp/large_mcp_response.json | while read item; do
  echo "$item" | jq 'select(.state == "ERROR")'
done
```

### Caching Common Queries
```bash
# Cache frequently used filters
cat /tmp/mcp_deployments.json | jq '.deployments.deployments[] | select(.state == "ERROR")' > /tmp/cached_errors.json

# Reuse cache
cat /tmp/cached_errors.json | jq '.[] | {id: .uid, url}'
```

---

## Best Practices Summary

1. **Threshold**: Write to file if MCP response > 5k tokens
2. **Naming**: Use descriptive filenames with timestamps
3. **Preservation**: Keep original data, filter to new files
4. **Iteration**: Progressively narrow filters
5. **Cleanup**: Remove temp files after session or when no longer needed
6. **Documentation**: Comment your jq filters for reusability
7. **Validation**: Check JSON validity before processing
8. **Error Handling**: Gracefully handle missing fields/empty results

---

## Integration Checklist

- [ ] Detect large MCP response (>5k tokens)
- [ ] Write response to `/tmp/mcp_<tool>_<timestamp>.json`
- [ ] Verify file was written correctly (`cat /tmp/mcp_*.json | jq .`)
- [ ] Apply filters to extract relevant data
- [ ] Calculate and report token savings
- [ ] Use filtered results for next actions
- [ ] Clean up temp files when done
- [ ] Document significant findings for future reference

---

## Troubleshooting

### Issue: jq command not found
```bash
# Install jq
sudo apt-get install jq  # Debian/Ubuntu
brew install jq          # macOS
```

### Issue: Permission denied writing to /tmp
```bash
# Check permissions
ls -ld /tmp

# Use alternative directory
mkdir -p ~/.claude/mcp-responses
cat > ~/.claude/mcp-responses/deployments.json
```

### Issue: File too large to display
```bash
# Check file size
ls -lh /tmp/mcp_response.json

# Stream processing instead
jq -c '.items[]' /tmp/mcp_response.json | head -10
```

### Issue: Nested JSON difficult to navigate
```bash
# Explore structure first
cat /tmp/mcp_response.json | jq 'keys'
cat /tmp/mcp_response.json | jq '.deployments | keys'
cat /tmp/mcp_response.json | jq '.deployments.deployments[0]'

# Build filter incrementally
```
