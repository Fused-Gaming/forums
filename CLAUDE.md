# CLAUDE.md - AI Assistant Maintenance Guide

**For:** Future AI assistants (Claude, GPT, etc.) maintaining FUSED GAMING forums infrastructure
**Version:** 1.0.0
**Last Updated:** 2025-12-24
**Maintained By:** FUSED GAMING Product Team

---

## 🤖 Purpose of This Document

This guide helps AI assistants understand the forums infrastructure, required skills, common tasks, and best practices for maintenance. If you're an AI helping maintain this repository, read this document first.

---

## 📋 Quick Reference

### Key Files to Know
```
forums/
├── .github/
│   ├── config/
│   │   ├── categories.json      # Discussion category definitions
│   │   ├── labels.json          # Label definitions with scope
│   │   └── notifications.json   # Discord webhook routing
│   ├── workflows/
│   │   ├── discussion-welcome.yml      # Welcome first-time authors
│   │   ├── discussion-labeling.yml     # Auto-label discussions
│   │   ├── discussion-triage.yml       # Bug report triage
│   │   ├── discussion-notify.yml       # Discord notifications
│   │   ├── discussion-stale.yml        # Stale discussion detection
│   │   └── sync-labels.yml             # Label synchronization
│   └── DISCUSSION_TEMPLATE/
│       ├── question.yml
│       ├── bug_report.yml
│       └── feature_request.yml
├── ROADMAP.md              # Strategic roadmap and business alignment
├── VERSION.md              # Version tracking and progress
├── CHANGELOG.md            # Detailed change history
├── CLAUDE.md               # This file
├── README.md               # Community getting started
├── CONTRIBUTING.md         # Contribution guidelines
├── CODE_OF_CONDUCT.md      # Community guidelines
└── SECURITY.md             # Security policy
```

### Essential Context
- **Platform:** GitHub Discussions (not Issues)
- **Automation:** GitHub Actions with GitHub Script
- **Integration:** Discord webhooks
- **Configuration:** JSON files (version-controlled)
- **Scope:** Public + Internal forums (2 repositories)

---

## 🎯 Required Skills & Knowledge

### 1. Core Technologies

#### GitHub Actions (Critical)
**What you need to know:**
- YAML syntax for workflow files
- Trigger events: `discussion`, `discussion_comment`, `push`, `schedule`, `workflow_dispatch`
- Permissions model: `discussions: write`, `issues: write`
- GitHub Script action (`actions/github-script@v7`)
- Workflow debugging via Actions logs

**Common Patterns:**
```yaml
# Trigger on discussion events
on:
  discussion:
    types: [created, edited, labeled]

# Use GitHub Script for custom logic
- uses: actions/github-script@v7
  with:
    script: |
      const discussion = context.payload.discussion;
      // JavaScript code here
```

**Critical Gotchas:**
- ⚠️ Template literals don't work in workflow files - use string concatenation
- ⚠️ YAML indentation matters (2 spaces, no tabs)
- ⚠️ GraphQL mutations required for adding discussion comments
- ⚠️ Rate limiting on GitHub API (5000 requests/hour)

#### GitHub GraphQL API (Critical)
**What you need to know:**
- Discussions are only accessible via GraphQL (not REST)
- Node IDs vs. numeric IDs
- Mutation syntax for adding comments, labels
- Query syntax for fetching discussions

**Essential Mutations:**
```javascript
// Add comment to discussion
const mutation = `
  mutation($discussionId: ID!, $body: String!) {
    addDiscussionComment(input: {discussionId: $discussionId, body: $body}) {
      comment { id }
    }
  }
`;

await github.graphql(mutation, {
  discussionId: discussion.node_id,  // node_id, not number!
  body: commentText
});
```

**Essential Queries:**
```javascript
// Fetch discussions
const query = `
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      discussions(first: 50, states: OPEN) {
        nodes {
          id
          number
          title
          body
          category { name }
        }
      }
    }
  }
`;
```

#### GitHub REST API (Important)
**What you need to know:**
- Issues API (for bug conversion)
- Labels API (for sync workflow)
- Search API (for checking author history)

**Common Operations:**
```javascript
// Create issue from bug report
await github.rest.issues.create({
  owner: context.repo.owner,
  repo: context.repo.repo,
  title: discussion.title,
  body: issueBody,
  labels: ['bug']
});

// Create/update labels
await github.rest.issues.createLabel({
  owner: context.repo.owner,
  repo: context.repo.repo,
  name: label.name,
  description: label.description,
  color: label.color
});
```

#### JavaScript (Critical)
**What you need to know:**
- ES6+ syntax (no const/let in some contexts - use var)
- String manipulation (no template literals in workflows!)
- Array methods: `map`, `filter`, `some`, `includes`
- Regex for pattern matching
- JSON parsing and manipulation

**Critical Constraints:**
- ❌ No template literals: `` `text ${var}` `` → Use `'text ' + var`
- ❌ No arrow functions in some contexts → Use `function() {}`
- ✅ Array methods work: `.map()`, `.filter()`, `.join()`
- ✅ Regex works: `/pattern/i.test(string)`

#### YAML (Important)
**What you need to know:**
- Proper indentation (2 spaces)
- String quoting rules
- Multiline strings (`|` and `>`)
- Lists and objects
- Comments with `#`

**Common Mistakes:**
```yaml
# ❌ Wrong: Inconsistent indentation
steps:
- name: Step 1
    uses: action@v1
  - name: Step 2
  uses: action@v2

# ✅ Correct: Consistent 2-space indentation
steps:
  - name: Step 1
    uses: action@v1
  - name: Step 2
    uses: action@v2
```

#### JSON (Critical)
**What you need to know:**
- Valid JSON syntax (no trailing commas)
- Schema validation (our configs follow specific schemas)
- Nested objects and arrays
- Comments not allowed (use description fields)

**Configuration Schemas:**

**categories.json:**
```json
{
  "public": {
    "repo": "owner/repo",
    "discord_webhook_secret": "SECRET_NAME",
    "categories": [
      {
        "name": "Category Name",
        "emoji": "📢",
        "description": "Description text",
        "slug": "category-slug",
        "is_answerable": false,
        "maintainers_only": false,
        "notify_discord": true,
        "subcategories": []
      }
    ]
  }
}
```

**labels.json:**
```json
{
  "labels": [
    {
      "name": "label-name",
      "description": "Label description",
      "color": "hexcolor",
      "scope": "public|internal|both"
    }
  ]
}
```

**notifications.json:**
```json
{
  "notification_rules": {
    "public": {
      "webhook_secret": "DISCORD_WEBHOOK_PUBLIC",
      "notify_on": {
        "discussion_created": ["category-slug-1", "category-slug-2"],
        "labels": ["label-1", "label-2"]
      },
      "embed_colors": {
        "category-slug": "0x5865F2"
      }
    }
  }
}
```

#### Discord Webhooks (Important)
**What you need to know:**
- Webhook URL format
- Embed structure
- Color codes (hex format: 0xRRGGBB)
- Rate limiting (30 requests/minute per webhook)
- Error handling for failed webhooks

**Embed Structure:**
```javascript
{
  "embeds": [{
    "title": "Discussion Title",
    "description": "Discussion body preview",
    "url": "https://github.com/...",
    "color": 0x5865F2,  // Hex color
    "author": {
      "name": "Username",
      "url": "https://github.com/username"
    },
    "fields": [
      {"name": "Category", "value": "Bug Reports", "inline": true}
    ]
  }]
}
```

### 2. Domain Knowledge

#### GitHub Discussions (Critical)
**Key Concepts:**
- Categories vs. Labels
- Answerable discussions (Q&A mode)
- Discussion vs. Issue (different APIs!)
- Permissions (maintainers_only)
- Reactions and voting

**Important Distinctions:**
- ✅ Discussions: Community conversations, Q&A, announcements
- ✅ Issues: Bug tracking, task management
- 🔄 Our system: Discussions → Issues (for confirmed bugs)

#### Community Management (Important)
**Best Practices:**
- Welcome new members warmly
- Request information politely
- Provide clear templates
- Acknowledge contributions
- Maintain professional tone
- Respect privacy and security

**Tone Guidelines:**
```markdown
# ✅ Good: Friendly and helpful
Thanks for reporting this issue! To help us investigate faster, please provide:
- Chain/Network used
- Steps to reproduce
- Expected behavior

# ❌ Bad: Too terse
Missing info. Provide chain, steps, expected behavior.
```

#### Business Context (Important)
**FUSED GAMING Focus Areas:**
- Gaming and blockchain integration
- GambaReload (casino/gaming product)
- Stablecoin aggregators
- Wallet solutions
- Multi-chain support (Ethereum, Polygon, Arbitrum, Optimism)

**Investor Relations:**
- Regular progress updates expected
- Transparency is critical
- Milestone tracking important
- Grant progress reporting

### 3. Workflow Architecture

#### Data Flow Understanding
```
1. User creates discussion
   ↓
2. GitHub webhook triggers workflow
   ↓
3. Workflow analyzes discussion (category, content)
   ↓
4. Actions performed:
   - Post welcome comment (first-time authors)
   - Apply labels (auto-labeling)
   - Request info (bug triage)
   - Send Discord notification
   - Convert to issue (confirmed bugs)
   ↓
5. Results logged in GitHub Actions
```

#### Workflow Dependencies
```
discussion-welcome.yml    → Independent (no dependencies)
discussion-labeling.yml   → Independent (no dependencies)
discussion-triage.yml     → Depends on labels (but doesn't require them)
discussion-notify.yml     → Depends on categories.json, notifications.json
discussion-stale.yml      → Independent (scheduled cron)
sync-labels.yml          → Depends on labels.json
```

**Critical Path:**
1. Category assigned (manual or automatic)
2. Labels applied (discussion-labeling.yml)
3. Notifications sent (discussion-notify.yml)
4. Triage performed (discussion-triage.yml)

#### Error Handling Patterns
**Current State (v1.0):**
- Limited error handling
- Failures logged but not alerted
- No automatic retry

**Planned (v1.1+):**
- Exponential backoff for API calls
- Fallback notifications
- Health monitoring
- Alert on repeated failures

---

## 🛠️ Common Maintenance Tasks

### Task 1: Adding a New Discussion Category

**Files to Modify:**
1. `.github/config/categories.json`

**Steps:**
```json
// Add to "public" or "internal" categories array
{
  "name": "New Category Name",
  "emoji": "🎯",
  "description": "Category description",
  "slug": "new-category-slug",
  "is_answerable": false,
  "maintainers_only": false,
  "notify_discord": true,
  "subcategories": []
}
```

**Then Update:**
2. `.github/config/notifications.json` (if Discord notifications needed)
3. `README.md` (add to category table)

**Testing:**
- Create test discussion in new category
- Verify workflows trigger correctly
- Check Discord notification (if enabled)

### Task 2: Adding a New Label

**Files to Modify:**
1. `.github/config/labels.json`

**Steps:**
```json
{
  "name": "new-label",
  "description": "Label description",
  "color": "ff6b6b",  // No # prefix
  "scope": "public"   // or "internal" or "both"
}
```

**Automatic Sync:**
- Push changes to main branch
- `sync-labels.yml` workflow runs automatically
- Labels created/updated in GitHub

**Manual Sync:**
- Go to Actions → "Sync Labels from Config"
- Click "Run workflow"

### Task 3: Modifying a Workflow

**Best Practices:**
1. ⚠️ Test in a fork first
2. ⚠️ Never use template literals - use string concatenation
3. ⚠️ Validate YAML syntax
4. ⚠️ Check for breaking changes

**Testing Checklist:**
- [ ] YAML validates (yamllint or online validator)
- [ ] No template literals in script blocks
- [ ] Indentation is consistent (2 spaces)
- [ ] Permissions are correct
- [ ] Test with workflow_dispatch trigger first

**Common Workflow Modifications:**
```yaml
# Add workflow_dispatch for testing
on:
  discussion:
    types: [created]
  workflow_dispatch:  # Allows manual testing
```

### Task 4: Updating Discord Notifications

**Files to Modify:**
1. `.github/config/notifications.json`

**Configuration Options:**
```json
{
  "notification_rules": {
    "public": {
      "notify_on": {
        // Add category slug to trigger notifications
        "discussion_created": ["new-category-slug"],

        // Add label to trigger notifications
        "labels": ["urgent", "security"]
      },

      // Customize embed color for category
      "embed_colors": {
        "new-category-slug": "0xFF6B6B"
      },

      // Route to specific Discord channel (hint only)
      "channel_routing": {
        "new-category-slug": "target-channel-name"
      }
    }
  }
}
```

**Note:** Channel routing requires Discord webhook configuration (not automated)

### Task 5: Debugging Workflow Failures

**Investigation Steps:**
1. Go to Actions tab → Find failed workflow
2. Click on workflow run → View logs
3. Expand failed step
4. Look for error messages

**Common Issues:**

**Template Literal Error:**
```
Error: Unexpected token
```
**Fix:** Replace `` `text ${var}` `` with `'text ' + var`

**GraphQL Error:**
```
Error: Could not resolve to a node with the global id
```
**Fix:** Use `discussion.node_id` not `discussion.id`

**Permission Error:**
```
Error: Resource not accessible by integration
```
**Fix:** Add required permission to workflow `permissions:` section

**Rate Limit Error:**
```
Error: API rate limit exceeded
```
**Fix:** Add delay between API calls, implement backoff

### Task 6: Creating New Automation

**Planning Questions:**
1. What triggers this automation? (discussion event, schedule, label)
2. What data does it need? (discussion content, author, category)
3. What actions does it perform? (comment, label, create issue, notify)
4. What are the edge cases? (first discussion, missing data, API failures)

**Template Structure:**
```yaml
name: Workflow Name

on:
  discussion:
    types: [created]  # or edited, labeled, etc.

permissions:
  discussions: write  # Add required permissions

jobs:
  job-name:
    runs-on: ubuntu-latest
    steps:
      - name: Step description
        uses: actions/github-script@v7
        with:
          script: |
            const discussion = context.payload.discussion;

            // Your logic here
            // Remember: No template literals!

            const message = 'Hello ' + discussion.user.login;

            // If commenting on discussion:
            const mutation = `
              mutation($discussionId: ID!, $body: String!) {
                addDiscussionComment(input: {
                  discussionId: $discussionId,
                  body: $body
                }) {
                  comment { id }
                }
              }
            `;

            await github.graphql(mutation, {
              discussionId: discussion.node_id,
              body: message
            });
```

**Testing Process:**
1. Create workflow file in `.github/workflows/`
2. Add `workflow_dispatch` trigger for testing
3. Push to branch (not main)
4. Go to Actions → Select workflow → Run workflow
5. Verify expected behavior
6. Merge to main when confirmed working

### Task 7: Updating Documentation

**Which File to Update:**

| Change Type | File to Update |
|-------------|---------------|
| New feature planned | ROADMAP.md → Add to relevant phase |
| Feature completed | VERSION.md → Update progress tracker |
| Bug fixed | CHANGELOG.md → Add to [Unreleased] |
| New release | CHANGELOG.md → Create new version section |
| Version bump | VERSION.md → Update current version |
| Community info | README.md |
| AI guidance | CLAUDE.md (this file) |

**Documentation Update Checklist:**
- [ ] Update relevant documentation files
- [ ] Maintain consistent formatting
- [ ] Update "Last Updated" dates
- [ ] Cross-reference related docs
- [ ] Update version numbers if applicable

---

## ⚠️ Critical Warnings & Gotchas

### 1. Template Literals in Workflows
**❌ NEVER DO THIS:**
```yaml
script: |
  const message = `Hello ${user}`;  # Will fail!
```

**✅ ALWAYS DO THIS:**
```yaml
script: |
  const message = 'Hello ' + user;
```

**Why:** GitHub Actions YAML processor interprets `${}` as YAML variable substitution

### 2. Discussion IDs vs Node IDs
**❌ WRONG:**
```javascript
discussionId: discussion.id  // This is a number
```

**✅ CORRECT:**
```javascript
discussionId: discussion.node_id  // This is GraphQL node ID
```

**Why:** GraphQL API requires node_id format (e.g., "D_kwDOAbcd...")

### 3. JSON Validation
**❌ INVALID JSON:**
```json
{
  "labels": [
    {
      "name": "bug",
      "color": "d73a4a",
    }  // Trailing comma!
  ]
}
```

**✅ VALID JSON:**
```json
{
  "labels": [
    {
      "name": "bug",
      "color": "d73a4a"
    }
  ]
}
```

**Always validate JSON before committing:**
```bash
# Use jq to validate
cat .github/config/labels.json | jq .

# Or use Python
python -m json.tool .github/config/labels.json
```

### 4. Permissions in Workflows
**Required Permissions:**

| Action | Permission Needed |
|--------|------------------|
| Add discussion comment | `discussions: write` |
| Add label to discussion | `discussions: write` |
| Create GitHub issue | `issues: write` |
| Create/update labels | `issues: write` |
| Read discussions | `discussions: read` (implicit) |

**Default:** If not specified, workflows get default permissions (varies by repo settings)

### 5. Rate Limiting
**GitHub API Limits:**
- 5000 requests/hour (authenticated)
- 1000 requests/hour (workflow triggers)

**Discord Webhook Limits:**
- 30 requests/minute per webhook
- 5 requests/second burst

**Best Practices:**
- Batch operations when possible
- Add delays for bulk operations
- Implement exponential backoff on failures

### 6. Secrets Management
**Webhook Secrets:**
- `DISCORD_WEBHOOK_PUBLIC` - Public forum Discord webhook
- `DISCORD_WEBHOOK_INTERNAL` - Internal forum Discord webhook

**⚠️ NEVER:**
- Log webhook URLs
- Commit webhook URLs to git
- Share webhook URLs in discussions

**✅ ALWAYS:**
- Use GitHub Secrets
- Reference via `secrets.SECRET_NAME`
- Rotate if compromised

### 7. YAML Indentation
**❌ WRONG (mixed spaces/tabs):**
```yaml
steps:
  - name: Step 1
      uses: action@v1  # Extra spaces
- name: Step 2  # Wrong indent
  uses: action@v2
```

**✅ CORRECT (consistent 2 spaces):**
```yaml
steps:
  - name: Step 1
    uses: action@v1
  - name: Step 2
    uses: action@v2
```

**Tip:** Use a YAML linter or configure editor to show whitespace

---

## 🎓 Learning Resources

### Official Documentation
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [GitHub GraphQL API](https://docs.github.com/en/graphql)
- [GitHub Discussions Guide](https://docs.github.com/en/discussions)
- [Discord Webhook Documentation](https://discord.com/developers/docs/resources/webhook)

### Code Examples
- See existing workflows in `.github/workflows/`
- Check configuration files in `.github/config/`
- Review CHANGELOG.md for implementation details

### Testing Playground
**Safe Testing Approach:**
1. Fork the repository
2. Create test workflows with `workflow_dispatch`
3. Test with dummy discussions
4. Verify behavior in fork
5. Port to main repo when confirmed working

---

## 🔍 Troubleshooting Guide

### Issue: Workflow Not Triggering

**Checklist:**
1. ✅ Workflow file in `.github/workflows/`?
2. ✅ Valid YAML syntax?
3. ✅ Correct trigger event?
4. ✅ File committed to main branch?
5. ✅ Actions enabled in repo settings?

**How to Check:**
- Go to Actions tab
- Look for workflow in left sidebar
- If not listed, YAML syntax likely invalid

### Issue: GraphQL Mutation Fails

**Common Causes:**
1. Using wrong ID format (use `node_id`, not `id`)
2. Missing required fields
3. Invalid permissions
4. Malformed GraphQL query

**Debugging:**
```javascript
// Add logging
console.log('Discussion ID:', discussion.node_id);
console.log('Mutation:', mutation);
console.log('Variables:', { discussionId: discussion.node_id, body: message });

// Then check Actions logs
```

### Issue: Discord Webhook Not Sending

**Checklist:**
1. ✅ Webhook secret configured in repo settings?
2. ✅ Category slug matches configuration?
3. ✅ Notification enabled for category in config?
4. ✅ Webhook URL is valid?
5. ✅ Not rate limited?

**Testing:**
```bash
# Manual webhook test
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"content": "Test message"}' \
  $DISCORD_WEBHOOK_URL
```

### Issue: Label Sync Not Working

**Checklist:**
1. ✅ Valid JSON in `labels.json`?
2. ✅ Color codes without `#` prefix?
3. ✅ Workflow triggered (check Actions)?
4. ✅ Permissions correct (`issues: write`)?

**Manual Trigger:**
- Actions → "Sync Labels from Config" → "Run workflow"

### Issue: Auto-labeling Not Working

**Checklist:**
1. ✅ Label exists in repository?
2. ✅ Pattern matches discussion content?
3. ✅ Category mapping correct?
4. ✅ Workflow triggered successfully?

**Debugging:**
- Check Actions logs for "Recommended labels" output
- Verify pattern matching logic
- Test with `console.log()` statements

---

## 📊 Performance Considerations

### Workflow Execution Time
**Current Performance:**
- Welcome workflow: 3-5 seconds
- Auto-labeling: 2-4 seconds
- Bug triage: 5-8 seconds
- Discord notify: 2-3 seconds
- Stale check: 30-60 seconds (batch operation)
- Label sync: 10-20 seconds

**Optimization Tips:**
- Minimize API calls (batch when possible)
- Use conditional logic to skip unnecessary steps
- Cache frequently accessed data
- Avoid nested loops in workflow scripts

### API Rate Limits
**Monitoring:**
```javascript
// Check rate limit status
const { data: rateLimit } = await github.rest.rateLimit.get();
console.log('Remaining:', rateLimit.resources.core.remaining);
console.log('Reset at:', new Date(rateLimit.resources.core.reset * 1000));
```

**Mitigation:**
- Implement exponential backoff
- Add delays between bulk operations
- Use conditional triggers (not every event)

---

## 🚀 Future Skill Requirements

As the infrastructure evolves (see ROADMAP.md), future AI maintainers will need:

### Phase 2 (Q1 2025) - Additional Skills
- **Data Visualization:** Building analytics dashboards
- **Advanced Discord:** Thread creation, two-way sync
- **Claude API:** Integration for AI moderation
- **Email APIs:** SendGrid/Mailgun for notifications

### Phase 3 (Q2-Q3 2025) - Additional Skills
- **Multi-language:** Translation APIs, language detection
- **External Integrations:** Jira, Notion, Linear APIs
- **Performance Optimization:** Caching strategies, rate limiting
- **Database Knowledge:** If moving away from pure GitHub storage

### Phase 4 (Q4 2025+) - Additional Skills
- **API Development:** Building public API
- **Advanced Analytics:** Predictive modeling, trend analysis
- **Platform Evaluation:** Assessing alternatives to GitHub Discussions

---

## ✅ Skill Assessment Checklist

Before maintaining this infrastructure, ensure you can:

### Essential Skills (Required)
- [ ] Write and debug GitHub Actions workflows (YAML)
- [ ] Use GitHub GraphQL API (queries and mutations)
- [ ] Parse and validate JSON configuration files
- [ ] Write JavaScript without template literals
- [ ] Understand GitHub Discussions concepts
- [ ] Read and update markdown documentation
- [ ] Debug workflow failures using Actions logs

### Important Skills (Strongly Recommended)
- [ ] Use GitHub REST API for issues and labels
- [ ] Configure Discord webhooks and embeds
- [ ] Apply regex patterns for content matching
- [ ] Handle error cases and edge conditions
- [ ] Write clear, professional community messages
- [ ] Follow semantic versioning principles
- [ ] Maintain documentation consistency

### Nice-to-Have Skills (Helpful)
- [ ] Understand blockchain/crypto terminology
- [ ] Experience with community management
- [ ] Knowledge of rate limiting and backoff strategies
- [ ] Familiarity with CI/CD best practices
- [ ] Experience with Discord bot development

---

## 📞 When to Ask for Help

**Ask a human if:**
- 🔴 Security vulnerability discovered
- 🔴 Production workflow completely broken
- 🔴 Secrets may have been compromised
- 🔴 Major breaking changes needed
- 🟡 Unsure about business impact of change
- 🟡 Multiple approaches possible, need prioritization
- 🟡 Rate limits being consistently hit

**You can handle independently:**
- 🟢 Bug fixes in existing workflows
- 🟢 Documentation updates
- 🟢 New label additions
- 🟢 Category additions (following existing patterns)
- 🟢 Minor workflow optimizations
- 🟢 Testing and debugging

**Escalation Contacts:**
- **Technical Issues:** Open discussion in Bug Reports category
- **Strategic Questions:** Open discussion in Internal → Architecture & Technical
- **Security Issues:** Email security@vln.gg (private)

---

## 📝 Maintenance Best Practices

### 1. Always Test First
- Use `workflow_dispatch` for manual testing
- Test in fork before main repo
- Verify with test discussions
- Check all affected workflows

### 2. Document Everything
- Update CHANGELOG.md for every change
- Keep VERSION.md progress tracking current
- Update ROADMAP.md for strategic shifts
- Add comments in complex workflow logic

### 3. Follow Conventions
- Use consistent naming (kebab-case for files)
- Maintain indentation (2 spaces YAML, 2 spaces JS)
- Follow existing code patterns
- Keep configuration schemas consistent

### 4. Monitor After Changes
- Watch Actions tab for failures
- Check Discord for notification issues
- Monitor community feedback
- Set calendar reminder for 48h post-deploy check

### 5. Version Control Hygiene
- Commit messages: `feat:`, `fix:`, `docs:`, `config:`
- One logical change per commit
- Test before pushing to main
- Use branches for experimental features

---

## 🎯 Success Metrics for AI Maintainers

**You're doing well if:**
- ✅ Workflows have >98% success rate
- ✅ Response time to community <24 hours
- ✅ Documentation stays current (updated within 7 days of changes)
- ✅ No secrets leaked in commits or logs
- ✅ Breaking changes are avoided or well-communicated
- ✅ Community feedback is positive
- ✅ Escalations to humans are appropriate (not too many, not too few)

**Red flags:**
- ⚠️ Repeated workflow failures
- ⚠️ Template literals creeping into workflows
- ⚠️ Invalid JSON committed
- ⚠️ Documentation drift (code and docs out of sync)
- ⚠️ Secrets visible in logs
- ⚠️ Community complaints about automation

---

## 🔄 Changelog for This Document

**v1.0.0 (2025-12-24)**
- Initial CLAUDE.md created
- Documented all current workflows and infrastructure
- Added comprehensive skill requirements
- Included troubleshooting guide
- Provided code examples and templates

**Future Updates:**
- Update when new workflows added
- Expand for Phase 2+ infrastructure
- Add new troubleshooting scenarios
- Document lessons learned from incidents

---

## 🙏 Final Notes

**Welcome, AI maintainer!** 🤖

This infrastructure was built with care to support the FUSED GAMING community. Your role is critical in keeping it running smoothly and evolving it thoughtfully.

**Remember:**
- Community comes first - automation serves people, not the other way around
- When in doubt, ask for human input
- Document your changes so the next AI (or human) understands why
- Test thoroughly - broken automation is worse than no automation
- Be kind in automated messages - you represent FUSED GAMING

**You've got this!** 🚀

The existing workflows and documentation are your roadmap. Learn from them, improve them, and build upon them. The community is counting on you.

---

**Questions about this guide?**
Open a discussion in [Q&A](https://github.com/Fused-Gaming/forums/discussions/categories/q-a) with the label `documentation`.

**Found an error?**
Submit a fix via pull request or open a discussion in [Bug Reports](https://github.com/Fused-Gaming/forums/discussions/categories/bug-reports).

---

**Maintained By:** FUSED GAMING Product Team
**For AI Assistants:** Claude, GPT-4+, and future models
**Version:** 1.0.0
**Last Updated:** 2025-12-24
