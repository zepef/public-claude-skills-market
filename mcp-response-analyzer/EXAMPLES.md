# MCP Response Analyzer - Examples

Practical examples showing how to use this skill to achieve 95-98% token savings when working with large MCP responses.

## Example 1: Finding Failed Vercel Deployments

### Scenario
You called `mcp__vercel__list_deployments` and received a ~10,100 token response with 20 deployments. You only need to find the failed ones.

### Step 1: Write Response to File
```bash
# After calling MCP tool, write response to temp file
cat > /tmp/mcp_deployments_20251025.json << 'EOF'
{
  "deployments": {
    "deployments": [
      {
        "uid": "dpl_72PE8zucbBAPjLBRfmUxvEUS3g1B",
        "state": "ERROR",
        "url": "parles-au-pcg-epw7fy4dr...",
        "created": 1761364860547,
        "meta": {
          "githubCommitMessage": "Externalize system prompts to /prompts directory",
          "githubCommitSha": "4b48fa3..."
        },
        "target": "production"
      },
      // ... 19 more deployments
    ]
  }
}
EOF
```

### Step 2: Invoke Skill to Filter
```bash
# Filter ERROR deployments only
cat /tmp/mcp_deployments_20251025.json | jq '.deployments.deployments[] | select(.state == "ERROR") | {
  id: .uid,
  state,
  url,
  created,
  commit_preview: .meta.githubCommitMessage[:100]
}' | head -20
```

### Step 3: Compact Result
**Output** (~300 tokens instead of 10,100):
```json
{
  "summary": "Found 2 ERROR deployments out of 20 total",
  "filtered_count": 2,
  "total_count": 20,
  "results": [
    {
      "id": "dpl_72PE8zucbBAPjLBRfmUxvEUS3g1B",
      "state": "ERROR",
      "url": "parles-au-pcg-epw7fy4dr...",
      "created": 1761364860547,
      "commit_preview": "Externalize system prompts to /prompts directory"
    },
    {
      "id": "dpl_5fD1ggMdAz6cBYitmAN91EhfCR5c",
      "state": "ERROR",
      "url": "parles-au-pcg-abc123...",
      "created": 1761350000000,
      "commit_preview": "Fix VAT calculator apostrophes"
    }
  ],
  "token_savings": "~9,800 tokens saved (97%)",
  "full_data": "/tmp/mcp_deployments_20251025.json",
  "next_steps": [
    "Use get_deployment_build_logs(id) to see error details",
    "Check inspector URL for visual debugging"
  ]
}
```

**Token Savings**: 10,100 → 300 tokens = **97% reduction**

---

## Example 2: Recent Production Deployments

### Scenario
Find the last 5 production deployments sorted by newest first.

### Filter Command
```bash
cat /tmp/mcp_deployments_20251025.json | jq '
  [.deployments.deployments[] | select(.target == "production")] |
  sort_by(.created) |
  reverse |
  .[:5] |
  .[] |
  {
    id: .uid,
    state,
    url,
    created,
    commit: .meta.githubCommitMessage[:80]
  }
'
```

### Output
```json
{
  "summary": "5 most recent production deployments",
  "results": [
    {
      "id": "dpl_5fD1ggMdAz6cBYitmAN91EhfCR5c",
      "state": "READY",
      "url": "parles-au-pcg.vercel.app",
      "created": 1761375000000,
      "commit": "Fix: ESLint errors + missing dependencies (react-markdown, remark-gfm)"
    },
    // ... 4 more
  ],
  "token_savings": "~9,600 tokens (95%)"
}
```

---

## Example 3: Deployments by Commit SHA

### Scenario
Find all deployments for a specific commit to track its deployment history.

### Filter Command
```bash
cat /tmp/mcp_deployments_20251025.json | jq --arg sha "4b48fa3" '
  [.deployments.deployments[] | select(.meta.githubCommitSha | startswith($sha))] |
  .[] |
  {
    id: .uid,
    state,
    target,
    readyState,
    url,
    created
  }
'
```

### Output
```json
{
  "summary": "2 deployments found for commit 4b48fa3",
  "results": [
    {
      "id": "dpl_72PE8zucbBAPjLBRfmUxvEUS3g1B",
      "state": "ERROR",
      "target": "production",
      "readyState": "BUILD_ERROR",
      "url": "parles-au-pcg-git-main-...",
      "created": 1761364860547
    },
    {
      "id": "dpl_abc123...",
      "state": "READY",
      "target": "preview",
      "readyState": "READY",
      "url": "parles-au-pcg-git-feature-...",
      "created": 1761360000000
    }
  ]
}
```

---

## Example 4: Analyzing Render Logs

### Scenario
100 log entries (~25,000 tokens) - find only ERROR level logs.

### Step 1: Write Logs to File
```bash
# After calling mcp__render__list_logs
cat > /tmp/mcp_render_logs_20251025.json << 'EOF'
{
  "logs": [
    {
      "timestamp": "2025-10-25T14:30:00Z",
      "level": "ERROR",
      "message": "Database connection failed",
      "instance": "srv-abc123"
    },
    // ... 99 more entries
  ]
}
EOF
```

### Step 2: Filter ERROR Logs
```bash
cat /tmp/mcp_render_logs_20251025.json | jq '
  [.logs[] | select(.level == "ERROR")] |
  .[:10] |
  .[] |
  {
    timestamp,
    level,
    message: .message[:100],
    instance
  }
'
```

### Output
```json
{
  "summary": "Found 5 ERROR logs out of 100 total",
  "results": [
    {
      "timestamp": "2025-10-25T14:30:00Z",
      "level": "ERROR",
      "message": "Database connection failed",
      "instance": "srv-abc123"
    },
    // ... 4 more errors
  ],
  "token_savings": "~24,600 tokens (98%)"
}
```

**Token Savings**: 25,000 → 400 tokens = **98% reduction**

---

## Example 5: Supabase Projects by Organization

### Scenario
50 projects (~15,000 tokens) - find projects in a specific organization.

### Filter Command
```bash
cat /tmp/mcp_supabase_projects_20251025.json | jq --arg org "org_abc123" '
  [.projects[] | select(.organization_id == $org)] |
  .[] |
  {
    id,
    name,
    status,
    region,
    created_at
  }
'
```

### Output
```json
{
  "summary": "3 projects found in organization org_abc123",
  "results": [
    {
      "id": "prj_k9FgC5lQe8fsLsinMuk3Nnyce68V",
      "name": "parles-au-pcg-db",
      "status": "ACTIVE_HEALTHY",
      "region": "us-east-1",
      "created_at": "2025-09-15T10:00:00Z"
    }
  ],
  "token_savings": "~14,500 tokens (97%)"
}
```

---

## Token Savings Comparison Table

| Scenario | Original Tokens | Filtered Tokens | Savings | Reduction % |
|----------|----------------|-----------------|---------|-------------|
| 20 Vercel deployments → 2 ERROR | 10,100 | 300 | 9,800 | 97% |
| 100 Render logs → 5 errors | 25,000 | 400 | 24,600 | 98% |
| 50 Supabase projects → 3 matching | 15,000 | 500 | 14,500 | 97% |
| 30 Vercel logs → text filter | 8,000 | 250 | 7,750 | 97% |
| 15 Render services → type filter | 6,000 | 350 | 5,650 | 94% |

**Average Savings**: **95-98% token reduction**

---

## Common jq Patterns

### Filter by Single Field
```bash
jq '.items[] | select(.state == "ERROR")'
```

### Filter by Multiple Conditions
```bash
jq '.items[] | select(.state == "ERROR" and .target == "production")'
```

### Filter by Date Range
```bash
jq --argjson start 1761300000000 --argjson end 1761400000000 '
  .items[] | select(.created >= $start and .created <= $end)
'
```

### Sort and Limit
```bash
jq '[.items[]] | sort_by(.created) | reverse | .[:5]'
```

### Extract Specific Fields
```bash
jq '.items[] | {id, state, url}'
```

### Nested Field Access
```bash
jq '.items[] | {id, commit: .meta.githubCommitMessage[:100]}'
```

### Count Matching Items
```bash
jq '[.items[] | select(.state == "ERROR")] | length'
```

---

## Workflow Integration

### Pattern 1: MCP → File → Skill → Action
```bash
# 1. Call MCP tool
mcp__vercel__list_deployments > /tmp/deployments.json

# 2. Invoke skill to filter
cat /tmp/deployments.json | jq '.deployments.deployments[] | select(.state == "ERROR")'

# 3. Take action on filtered results
for id in $(cat filtered.json | jq -r '.id'); do
  mcp__vercel__get_deployment_build_logs --id=$id
done
```

### Pattern 2: Iterative Refinement
```bash
# First pass - broad filter
cat /tmp/deployments.json | jq '.deployments.deployments[] | select(.target == "production")'

# Second pass - narrow down
cat /tmp/deployments.json | jq '.deployments.deployments[] | select(.target == "production" and .state == "ERROR")'

# Final extraction
cat /tmp/deployments.json | jq '.deployments.deployments[] | select(.target == "production" and .state == "ERROR") | {id, url, created}'
```

---

## Cleanup

### List Temporary Files
```bash
ls -lh /tmp/mcp_*.json
```

### Remove Old Files
```bash
# Remove files older than 1 day
find /tmp -name "mcp_*.json" -mtime +1 -delete

# Remove all MCP temp files
rm /tmp/mcp_*.json
```

### Verify Cleanup
```bash
ls -lh /tmp/mcp_*.json 2>/dev/null || echo "All cleaned up!"
```

---

## Best Practices

1. **Descriptive Filenames**: Use timestamps and tool names
   ```bash
   /tmp/mcp_vercel_deployments_20251025_143000.json
   /tmp/mcp_render_logs_20251025_150000.json
   ```

2. **Preserve Original Data**: Skill reads from file, doesn't modify it
   ```bash
   # Keep original
   cat original.json | jq '.filter' > filtered.json
   ```

3. **Progressive Filtering**: Start broad, narrow down iteratively
   ```bash
   # Step 1: All production
   # Step 2: Production + ERROR
   # Step 3: Production + ERROR + last 7 days
   ```

4. **Calculate Savings**: Show token efficiency gains
   ```bash
   ORIGINAL=$(cat /tmp/mcp_response.json | wc -c)
   FILTERED=$(cat filtered.json | wc -c)
   echo "Savings: $(( (ORIGINAL - FILTERED) * 100 / ORIGINAL ))%"
   ```

5. **Clean Up After Session**: Remove temp files when done
   ```bash
   rm /tmp/mcp_*.json
   ```
