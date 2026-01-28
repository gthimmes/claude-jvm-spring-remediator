# JVM Spring Remediator

A Claude Code skill that analyzes CVE vulnerabilities in Java/Kotlin Spring Boot applications, assesses actual code exposure, and implements minimal-impact remediation.

**Transform "we have a CVE" into "here's the fix, applied and verified"** - security remediation that actually gets done.

## What This Is

This is a **Claude Code skill** - it runs inside Claude Code and uses **Claude's capabilities** to analyze, assess, and fix CVE vulnerabilities in your codebase.

- **Single CVE focus** - Give it one CVE, get it fixed
- **Exposure assessment** - Determines if your code actually uses vulnerable paths
- **Latest safe patch** - Recommends latest patch version, not just minimum fix
- **Minimal-impact fixes** - Prioritizes patch updates over major version jumps
- **Actually implements the fix** - Not just snippets, real changes to your build files
- **Specific test plans** - CVE-type-specific verification steps, not generic guidance
- **Clean dependency refresh** - Ensures correct versions are pulled after changes
- **Jira-ready output** - Complete summary with test plan you can paste into your ticket

## Installation

### Step 1: Add the Marketplace

```bash
/plugin marketplace add gthimmes/claude-marketplace
```

### Step 2: Install the Plugin

```bash
/plugin install jvm-spring-remediator
```

> **Note**: If the `/remediate` slash command doesn't work immediately after installation, restart Claude Code to refresh the skill registry.

Once installed, use `/remediate CVE-XXXX-XXXXX` or ask Claude to help fix a CVE.

## Quick Start

**In Claude Code:**

```
User: /remediate CVE-2024-38816

Claude: I'll analyze CVE-2024-38816 for your project.

[Fetches CVE details from NVD]
[Analyzes your dependency tree]
[Searches codebase for exposure]
[Queries Maven Central for latest patch version]
[Verifies no new CVEs in patch range]

## CVE Analysis: CVE-2024-38816

Severity: CRITICAL (CVSS 9.8)
Affected: spring-webmvc < 5.3.31
Your Version: 5.3.27
Exposure: MEDIUM - RouterFunction usage found in WebConfig.java:45

## Remediation Plan

Strategy: Dependency Management Override
Minimum Fix: 5.3.31
Recommended: 5.3.39 (latest patch in 5.3.x line)
Risk: MINIMAL - patch update, no breaking changes

Proceed with remediation?

User: Yes

Claude: [Updates pom.xml]
[Runs clean build with dependency refresh]
[Verifies with dependency tree]
[Generates specific test plan]

Done! Here's the summary for your Jira ticket:

CVE: CVE-2024-38816
Action: Updated spring-webmvc to 5.3.39
File: pom.xml
Dependencies Refreshed: Yes

## Test Plan

### Automated Tests
./gradlew test --tests "WebConfigTest"

### Manual Verification (Path Traversal)
1. Confirm normal access works: curl http://localhost:8080/static/styles.css
2. Confirm traversal blocked: curl http://localhost:8080/static/../../../etc/passwd

### Security Scan
./gradlew dependencyCheckAnalyze
```

## What This Does

When you provide a CVE ID, Claude will:

1. **Fetch CVE Details** from NVD
   - Severity and CVSS score
   - Affected library and version ranges
   - Vulnerability description

2. **Analyze Your Dependencies**
   - Check if vulnerable library is present
   - Determine if direct or transitive
   - Map the dependency path

3. **Assess Actual Exposure**
   - Search codebase for usage of vulnerable code paths
   - Categorize exposure: HIGH, MEDIUM, LOW, or MINIMAL
   - Always recommend fixing regardless of exposure level

4. **Recommend Remediation Strategy**
   - Prioritize patch updates (lowest risk)
   - Recommend latest patch in minor line, not just minimum fix
   - Verify no new CVEs between minimum fix and latest patch
   - Fall back to minor/major only when necessary

5. **Implement the Fix** (after your approval)
   - Update pom.xml or build.gradle
   - Add dependency management overrides if needed

6. **Generate Specific Test Plan**
   - Identify affected code paths from exposure assessment
   - Provide specific test commands for your test framework
   - Include CVE-type-specific manual verification steps
   - Never generic "run your tests" guidance

7. **Clean and Refresh Dependencies**
   - Run `./gradlew clean build --refresh-dependencies` for Gradle
   - Run `mvn clean install -U` for Maven
   - Ensures correct versions are actually pulled

8. **Verify and Report**
   - Confirm new version in dependency tree
   - Provide Jira-ready summary with complete test plan

## Remediation Strategy Priority

The skill follows this order to minimize risk:

| Priority | Strategy | Risk Level | Example |
|----------|----------|------------|---------|
| A | Patch Update | MINIMAL | 5.3.27 → 5.3.39 |
| B | Minor Update | MODERATE | 5.3.27 → 5.4.15 |
| C | Major Update | HIGH | 5.3.27 → 6.1.5 |
| D | Spring Boot Update | HIGH | 2.7.12 → 2.7.18 |
| E | Version Override | MODERATE | Force transitive version |
| F | Exclusion | VARIABLE | Exclude unused transitive |

## Why Always Fix?

Even when exposure assessment shows LOW or MINIMAL risk, this skill always recommends remediation because:

- **Code evolves** - Today's unused path may be tomorrow's feature
- **Defense in depth** - Security best practice
- **Clean dependency tree** - Eliminates vulnerability entirely
- **Compliance** - Auditors want CVEs resolved, not justified

## Supported Build Systems

### Maven
- Parses `pom.xml` files
- Uses `mvn dependency:tree` for analysis
- Updates direct dependencies or adds `dependencyManagement` overrides
- Runs `mvn clean install -U` to refresh dependencies

### Gradle
- Supports Groovy and Kotlin DSL
- Uses `./gradlew dependencies` for analysis
- Updates dependencies or adds `resolutionStrategy` overrides
- Runs `./gradlew clean build --refresh-dependencies` to refresh

## Example Output (Jira-Ready)

```
## Remediation Summary

**CVE**: CVE-2024-38816
**Severity**: CRITICAL (CVSS 9.8)
**Action Taken**: Updated spring-webmvc to 5.3.39 (latest in 5.3.x, min fix was 5.3.31)
**CVE Verification**: No new CVEs in versions 5.3.31 through 5.3.39
**Files Modified**: pom.xml
**Dependencies Refreshed**: Yes

## Test Plan

### Automated Tests
./gradlew test --tests "WebConfigTest"
./gradlew test --tests "StaticResourceTest"

### Manual Verification (Path Traversal)
1. Positive test - normal access works:
   curl http://localhost:8080/static/styles.css → returns file

2. Negative test - traversal blocked:
   curl http://localhost:8080/static/../../../etc/passwd → returns 400

3. Encoded test - encoded traversal blocked:
   curl http://localhost:8080/static/..%2F..%2Fetc/passwd → returns 400

### Security Scan
./gradlew dependencyCheckAnalyze
```

## Privacy & Security

- **All analysis runs locally** within Claude Code
- **CVE data fetched from public NVD API**
- **No code sent to external services**
- **Your codebase stays private**

## Documentation

- **[skills/jvm-spring-remediator/SKILL.md](skills/jvm-spring-remediator/SKILL.md)** - Full skill instructions

## Use Cases

### For Security Teams
- Quick CVE triage and remediation
- Consistent remediation approach
- Audit-ready documentation with test plans

### For Engineering Teams
- Fix CVEs without deep dependency research
- Minimal-risk update strategies
- Specific test commands for verification

### For DevOps
- Streamlined security patching
- Predictable build file changes
- Easy integration with CI/CD

## Requirements

- **Claude Code** installed
- **Maven or Gradle** project
- **Java or Kotlin** Spring Boot application
- **Internet access** for NVD API queries

## Limitations

- Single CVE at a time (by design for focused remediation)
- Requires valid, parseable build files
- Internet access needed for CVE data
- Complex version conflicts may need manual review

## Get Started

```bash
# Step 1: Add the marketplace
/plugin marketplace add gthimmes/claude-jvm-spring-remediator

# Step 2: Install the plugin
/plugin install jvm-spring-remediator

# Step 3: Use it
/remediate CVE-2024-38816
```

**Turn CVE alerts into completed fixes with specific test plans.**

---

Built as a Claude Code plugin - Analyzes exposure, recommends latest safe patch, implements fix, generates test plan
