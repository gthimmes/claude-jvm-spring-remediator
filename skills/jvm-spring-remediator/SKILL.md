---
name: remediate
description: Analyze a CVE vulnerability in Java/Kotlin Spring Boot codebases, assess actual exposure, and implement minimal-impact remediation. Use when user provides a CVE ID to analyze and fix.
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, Task, WebFetch, WebSearch
---

# JVM Spring CVE Remediator

A Claude Code skill that analyzes CVE vulnerabilities in Java/Kotlin Spring Boot applications, assesses actual code exposure, and implements minimal-impact remediation.

## What This Skill Does

When invoked with a CVE ID, you (Claude) will:

1. **Fetch CVE details** from NVD and security advisories
2. **Analyze the project's dependencies** to determine if affected
3. **Assess actual exposure** by scanning the codebase for usage of vulnerable code paths
4. **Recommend remediation strategy** prioritizing minimal-impact updates
5. **Present a remediation plan** for user approval
6. **Implement the fix** by updating build files once approved
7. **Clean and refresh dependencies** to ensure correct versions are pulled
8. **Verify the remediation** by checking the updated dependency tree

## Input Format

The user will provide a single CVE identifier:
```
/remediate CVE-2024-38816
```

Or simply mention a CVE in conversation:
```
Can you fix CVE-2024-38816 in my project?
```

## Execution Workflow

### Phase 1: CVE Data Gathering

Fetch vulnerability details from the National Vulnerability Database:

```
URL: https://services.nvd.nist.gov/rest/json/cves/2.0?cveId={CVE-ID}
```

Extract and present:
- **CVSS Score** and severity (CRITICAL/HIGH/MEDIUM/LOW)
- **Description** of the vulnerability
- **Affected library** and version ranges
- **CWE classification** (type of vulnerability)
- **Fixed versions** if available

Also search for additional context:
- Spring Security Advisories (for Spring-related CVEs)
- GitHub Security Advisories
- Library release notes

### Phase 2: Project Dependency Analysis

Identify the build system and analyze dependencies:

**For Maven projects:**
```bash
# Find pom.xml files
find . -name "pom.xml" 2>/dev/null

# Get dependency tree
mvn dependency:tree -DoutputType=text

# Search for specific library
mvn dependency:tree -Dincludes=*:{artifactId}*
```

**For Gradle projects:**
```bash
# Find build files
find . -name "build.gradle" -o -name "build.gradle.kts" 2>/dev/null

# Get dependency tree
./gradlew dependencies --configuration runtimeClasspath

# Search for specific library
./gradlew dependencyInsight --dependency {library-name}
```

Determine:
- Is the vulnerable library present?
- What version is currently used?
- Is it a direct or transitive dependency?
- What is the dependency path?

### Phase 3: Exposure Assessment

**IMPORTANT**: Always assess exposure, but always offer the remediation option regardless of exposure level. Code changes over time and a non-exposed vulnerability today could become exposed tomorrow.

Search the codebase for actual usage of vulnerable functionality:

```bash
# Search for imports of vulnerable package
grep -r "import.*{vulnerable.package}" --include="*.java" --include="*.kt" src/

# Search for usage of vulnerable classes/methods
grep -r "{VulnerableClass}" --include="*.java" --include="*.kt" src/

# Check Spring configurations if relevant
grep -r "@Enable{Feature}" --include="*.java" --include="*.kt" src/
```

Categorize exposure level:
- **HIGH**: Vulnerable code directly used in REST endpoints or user-facing code
- **MEDIUM**: Vulnerable code used in application logic
- **LOW**: Vulnerable dependency present but specific vulnerable code path not used
- **MINIMAL**: Dependency is transitive and vulnerable functionality not invoked

**Always present findings like this:**
```
Exposure Assessment: [HIGH/MEDIUM/LOW/MINIMAL]

[Explanation of what was found or not found]

RECOMMENDATION: Even though exposure is [level], I recommend remediating this vulnerability because:
- Code may change in the future and begin using vulnerable paths
- Defense in depth is a security best practice
- It eliminates the vulnerability from your dependency tree entirely
```

### Phase 4: Remediation Strategy Selection

Follow this priority order for selecting the safest remediation:

#### Strategy A: Patch Version Update (LOWEST RISK)
- Same major.minor version, increment patch only (x.y.Z)
- Example: `5.3.27` → `5.3.31`
- Criteria: Remediates CVE, introduces zero new CVEs, no breaking changes
- Testing: Smoke tests + affected area regression

#### Strategy B: Minor Version Update (MODERATE RISK)
- Same major, increment minor (x.Y.z)
- Example: `5.3.27` → `5.4.15`
- Criteria: Patch unavailable, minor version fixes CVE
- Testing: Full regression testing

#### Strategy C: Major Version Update (HIGH RISK)
- Increment major version (X.y.z)
- Example: `5.3.27` → `6.1.5`
- Criteria: Only option to remediate, requires careful analysis
- Testing: Full QA cycle + compatibility verification

#### Strategy D: Spring Boot Version Update
- Update Spring Boot parent/BOM version
- Criteria: CVE in Spring-managed dependency, Spring Boot update remediates
- Testing: Full application regression

#### Strategy E: Version Override
- Override transitive dependency version
- Use Maven `<dependencyManagement>` or Gradle `resolutionStrategy`
- Criteria: CVE in transitive dependency, parent compatible with newer version

#### Strategy F: Dependency Exclusion
- Exclude vulnerable transitive dependency if not used
- Criteria: Vulnerable library is transitive and not directly used

**For each strategy, verify:**
1. Query Maven Central for available versions
2. Check if target version remediates the CVE
3. Check if target version introduces any new CVEs
4. Review release notes for breaking changes

Maven Central API:
```
https://search.maven.org/solrsearch/select?q=g:{groupId}+AND+a:{artifactId}&rows=50&wt=json
```

### Phase 5: Present Remediation Plan

Present the plan clearly and ask for approval:

```
## CVE Remediation Plan

### Vulnerability
- **CVE ID**: CVE-XXXX-XXXXX
- **Severity**: CRITICAL (CVSS 9.8)
- **Description**: [Brief description]
- **Affected Library**: groupId:artifactId
- **Current Version**: X.Y.Z
- **Your Exposure**: [HIGH/MEDIUM/LOW/MINIMAL] - [Brief explanation]

### Recommended Fix
- **Strategy**: Patch Version Update
- **Target Version**: A.B.C
- **Risk Level**: MINIMAL
- **Breaking Changes**: None expected

### What I Will Do
1. Update [pom.xml/build.gradle] to change version from X.Y.Z to A.B.C
2. [If dependencyManagement override] Add version override in dependency management section
3. Run clean dependency refresh to pull new versions
4. Verify the update with dependency tree command

### After the Fix
You should:
- Run your test suite
- Perform a security scan to confirm remediation
- Deploy to staging for verification

**Do you want me to proceed with this remediation?**
```

### Phase 6: Implement the Fix

**Only proceed after user approval.**

Make the actual changes to build files:

**For Maven (pom.xml):**
```xml
<!-- Direct dependency version update -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>5.3.31</version>  <!-- Updated from 5.3.27 -->
</dependency>

<!-- OR dependency management override for transitive -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>5.3.31</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**For Gradle (build.gradle.kts):**
```kotlin
// Direct dependency version update
implementation("org.springframework:spring-webmvc:5.3.31")

// OR resolution strategy for transitive
configurations.all {
    resolutionStrategy {
        force("org.springframework:spring-webmvc:5.3.31")
    }
}
```

Use the Edit tool to make precise changes to the build files. Preserve formatting and comments.

### Phase 7: Clean and Refresh Dependencies

**IMPORTANT**: After updating build files, always run a clean dependency refresh to ensure the correct versions are pulled.

**For Gradle projects:**
```bash
# Clean build and refresh dependencies
./gradlew clean build --refresh-dependencies
```

If the full build takes too long or fails due to unrelated issues, at minimum run:
```bash
# Just refresh dependencies without full build
./gradlew --refresh-dependencies dependencies
```

**For Maven projects:**
```bash
# Clean and update dependencies
mvn clean install -U
```

If the full build is not needed, at minimum run:
```bash
# Force update of dependencies
mvn dependency:purge-local-repository -DreResolve=true
mvn dependency:resolve
```

### Phase 8: Verify and Report

After the clean dependency refresh:

1. Run dependency verification to confirm the new version:
```bash
# Maven
mvn dependency:tree -Dincludes=*:{artifactId}*

# Gradle
./gradlew dependencyInsight --dependency {library-name}
```

2. Provide a summary for the user's Jira ticket:

```
## Remediation Summary (for Jira)

**CVE**: CVE-XXXX-XXXXX
**Severity**: CRITICAL
**Action Taken**: Updated {library} from {old-version} to {new-version}
**Files Modified**: pom.xml (or build.gradle)
**Dependencies Refreshed**: Yes - ran clean build with dependency refresh
**Verification**: Dependency tree confirms new version

**Next Steps**:
- Run test suite
- Security scan to confirm remediation
- Deploy to staging
```

## Key Principles

### Always Offer Remediation
Even if exposure assessment shows LOW or MINIMAL risk, always recommend and offer to implement the fix. Explain:
- Code evolves and may use vulnerable paths in the future
- Defense in depth is security best practice
- Eliminates vulnerability from dependency tree

### Minimize Impact
- Prefer patch updates over minor updates
- Prefer minor updates over major updates
- Avoid introducing breaking changes when possible
- Check for new CVEs in target versions

### Be Specific
- Provide exact version numbers
- Show exact file changes
- Give precise verification commands

### Actually Do the Work
- Don't just show snippets - use Edit tool to make real changes
- Run clean dependency refresh after changes
- Verify the changes worked
- Provide clear next steps

### Keep Output Jira-Friendly
- Concise summary suitable for ticket updates
- Clear action taken and files modified
- Specific next steps for the team

## Common Libraries Reference

| Short Name | Full Coordinates |
|------------|------------------|
| spring-core | org.springframework:spring-core |
| spring-web | org.springframework:spring-web |
| spring-webmvc | org.springframework:spring-webmvc |
| spring-security | org.springframework.security:spring-security-core |
| jackson-databind | com.fasterxml.jackson.core:jackson-databind |
| log4j | org.apache.logging.log4j:log4j-core |
| snakeyaml | org.yaml:snakeyaml |
| commons-io | commons-io:commons-io |
| tomcat | org.apache.tomcat.embed:tomcat-embed-core |
| netty | io.netty:netty-all |

## Spring Boot Compatibility

When considering updates, check Spring Boot compatibility:

| Spring Boot | Spring Framework | Java |
|-------------|------------------|------|
| 2.7.x | 5.3.x | 8+ |
| 3.0.x | 6.0.x | 17+ |
| 3.1.x | 6.0.x | 17+ |
| 3.2.x | 6.1.x | 17+ |

## Error Handling

If issues occur:

- **CVE not found**: Verify CVE ID format, check if recently published
- **Library not in project**: Confirm with user, check all modules
- **No safe version available**: Explain options (exclusion, alternative library, accept risk)
- **Build file parse error**: Ask user to verify build file is valid
- **Dependency refresh fails**: Report the error, suggest manual verification

## Example Session

```
User: /remediate CVE-2024-38816

Claude: I'll analyze CVE-2024-38816 for your project.

[Fetches CVE data from NVD]
[Analyzes dependency tree]
[Searches codebase for exposure]

## CVE Analysis: CVE-2024-38816

**Severity**: CRITICAL (CVSS 9.8)
**Description**: Path traversal vulnerability in Spring Framework WebMvc.fn and WebFlux.fn
**Affected**: spring-webmvc < 5.3.31, < 6.0.15, < 6.1.3

**Your Project**:
- Current version: spring-webmvc 5.3.27 (via Spring Boot 2.7.12)
- Dependency type: Transitive (through spring-boot-starter-web)
- Exposure: MEDIUM - Found RouterFunction usage in WebConfig.java

**Recommendation**: Even with medium exposure, I recommend fixing this to eliminate the vulnerability entirely.

## Remediation Plan

**Strategy**: Dependency Management Override (lowest risk)
**Target Version**: 5.3.31
**Risk**: MINIMAL - patch update, no breaking changes

**What I'll do**:
1. Add spring-webmvc 5.3.31 to dependencyManagement in pom.xml
2. Run clean dependency refresh
3. Verify with dependency tree

Proceed with remediation?

User: Yes

Claude: [Uses Edit tool to update pom.xml]
[Runs ./gradlew clean build --refresh-dependencies]
[Runs dependency verification]

Done! Here's the summary for your Jira ticket:

**CVE**: CVE-2024-38816
**Severity**: CRITICAL
**Action**: Added dependency management override for spring-webmvc 5.3.31
**File Modified**: pom.xml
**Dependencies Refreshed**: Yes
**Verified**: Dependency tree confirms 5.3.31

**Next Steps**:
- Run test suite
- Security scan to confirm
- Deploy to staging
```
