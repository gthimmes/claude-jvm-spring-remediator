# JVM Spring Remediator

A Claude Code skill that analyzes CVE vulnerabilities in Java/Kotlin Spring Boot applications, assesses actual code exposure, and implements minimal-impact remediation.

**Transform "we have a CVE" into "here's the fix, applied and verified"** - security remediation that actually gets done.

## What This Is

This is a **Claude Code skill** - it runs inside Claude Code and uses **Claude's capabilities** to analyze, assess, and fix CVE vulnerabilities in your codebase.

- **Single CVE focus** - Give it one CVE, get it fixed
- **Exposure assessment** - Determines if your code actually uses vulnerable paths
- **Minimal-impact fixes** - Prioritizes patch updates over major version jumps
- **Actually implements the fix** - Not just snippets, real changes to your build files
- **Clean dependency refresh** - Ensures correct versions are pulled after changes
- **Jira-ready output** - Concise summary you can paste into your ticket

## Installation

### Step 1: Add the Marketplace

```bash
/plugin marketplace add gthimmes/claude-jvm-spring-remediator
```

### Step 2: Install the Plugin

```bash
/plugin install jvm-spring-remediator
```

Once installed, Claude will automatically use this skill when you mention CVE remediation.

## Quick Start

**In Claude Code:**

```
User: /jvm-spring-remediator CVE-2024-38816

Claude: I'll analyze CVE-2024-38816 for your project.

[Fetches CVE details from NVD]
[Analyzes your dependency tree]
[Searches codebase for exposure]

## CVE Analysis: CVE-2024-38816

Severity: CRITICAL (CVSS 9.8)
Affected: spring-webmvc < 5.3.31
Your Version: 5.3.27
Exposure: MEDIUM - RouterFunction usage found in WebConfig.java

## Remediation Plan

Strategy: Dependency Management Override
Target: 5.3.31 (patch update, minimal risk)

Proceed with remediation?

User: Yes

Claude: [Updates pom.xml]
[Runs clean build with dependency refresh]
[Verifies with dependency tree]

Done! Summary for Jira:

CVE: CVE-2024-38816
Action: Updated spring-webmvc to 5.3.31
File: pom.xml
Dependencies Refreshed: Yes
Next: Run tests, security scan, deploy to staging
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
   - Fall back to minor/major only when necessary
   - Check target version for new CVEs

5. **Implement the Fix** (after your approval)
   - Update pom.xml or build.gradle
   - Add dependency management overrides if needed

6. **Clean and Refresh Dependencies**
   - Run `./gradlew clean build --refresh-dependencies` for Gradle
   - Run `mvn clean install -U` for Maven
   - Ensures correct versions are actually pulled

7. **Verify and Report**
   - Confirm new version in dependency tree
   - Provide Jira-ready summary

## Remediation Strategy Priority

The skill follows this order to minimize risk:

| Priority | Strategy | Risk Level | Example |
|----------|----------|------------|---------|
| A | Patch Update | MINIMAL | 5.3.27 → 5.3.31 |
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
**Action Taken**: Added dependency management override for spring-webmvc 5.3.31
**Files Modified**: pom.xml
**Dependencies Refreshed**: Yes - ran clean build with dependency refresh
**Verification**: Dependency tree confirms new version

**Next Steps**:
- Run test suite
- Security scan to confirm remediation
- Deploy to staging
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
- Audit-ready documentation

### For Engineering Teams
- Fix CVEs without deep dependency research
- Minimal-risk update strategies
- Clear verification steps

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
/jvm-spring-remediator CVE-2024-38816
```

**Turn CVE alerts into completed fixes.**

---

Built as a Claude Code plugin - Analyzes exposure, recommends strategy, implements the fix, refreshes dependencies
