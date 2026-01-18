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
7. **Generate a specific test plan** based on affected code paths and CVE type
8. **Clean and refresh dependencies** to ensure correct versions are pulled
9. **Verify the remediation** and provide Jira-ready summary with test plan

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

### Version Selection Process

**IMPORTANT**: Always recommend the latest patch version within the compatible minor version line, not just the minimum fix version. This ensures the user gets all available security fixes and bug fixes.

**Follow this process:**

1. **Identify the minimum fix version** from CVE data (e.g., "fixed in 5.3.31")

2. **Determine the current minor version line** from the project (e.g., if using 5.3.27, the line is 5.3.x)

3. **Query Maven Central for all available versions** in that minor line:
   ```
   https://search.maven.org/solrsearch/select?q=g:{groupId}+AND+a:{artifactId}&rows=100&wt=json
   ```

4. **Find the latest patch version** in the compatible minor line (e.g., 5.3.39 if that's the newest 5.3.x)

5. **Verify no new CVEs exist** between the minimum fix and latest patch:
   - Query NVD for CVEs affecting versions between minimum fix and latest
   - Check GitHub Security Advisories for the library
   - If new CVEs exist in the latest patch, find the newest version without CVEs

6. **Recommend the latest safe patch version**, not just the minimum fix

**Example:**
```
CVE-2024-38816 is fixed in spring-webmvc 5.3.31
Project uses spring-webmvc 5.3.27
Latest available in 5.3.x line: 5.3.39

Process:
1. Minimum fix: 5.3.31
2. Latest patch: 5.3.39
3. Check CVEs for 5.3.31 through 5.3.39 → None found
4. Recommend: 5.3.39 (not 5.3.31)

Rationale: "Recommending 5.3.39 (latest in 5.3.x line) rather than
minimum fix 5.3.31 to include all subsequent security and bug fixes.
Verified no new CVEs in versions 5.3.31 through 5.3.39."
```

### Version Verification Checklist

For each recommended version, verify:
1. ✅ Remediates the original CVE
2. ✅ Is the latest patch in the compatible minor version line
3. ✅ No new CVEs introduced between minimum fix and recommended version
4. ✅ No known breaking changes from current version
5. ✅ Compatible with project's Spring Boot version

Maven Central API:
```
https://search.maven.org/solrsearch/select?q=g:{groupId}+AND+a:{artifactId}&rows=100&wt=json
```

NVD API (to check for CVEs in a version range):
```
https://services.nvd.nist.gov/rest/json/cves/2.0?virtualMatchString=cpe:2.3:a:{vendor}:{product}:*
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
- **Minimum Fix Version**: A.B.C (first version that fixes CVE)
- **Recommended Version**: A.B.D (latest patch in A.B.x line)
- **Risk Level**: MINIMAL
- **Breaking Changes**: None expected

### Version Selection Rationale
- Minimum fix is A.B.C, but A.B.D is the latest in the A.B.x line
- Verified no new CVEs exist in versions A.B.C through A.B.D
- Recommending latest patch to include all subsequent security and bug fixes

### What I Will Do
1. Update [pom.xml/build.gradle] to change version from X.Y.Z to A.B.D
2. [If dependencyManagement override] Add version override in dependency management section
3. Run clean dependency refresh to pull new versions
4. Verify the update with dependency tree command
5. Generate a specific test plan based on your exposure

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

### Phase 7: Generate Test Plan

After implementing the fix, generate a **specific test plan** based on the exposure assessment and CVE type. Do not provide generic guidance - tailor the test plan to the actual affected code paths.

#### 7.1 Identify Affected Code Paths

Based on the exposure assessment from Phase 3, list the specific:
- Classes and methods that use the vulnerable functionality
- REST endpoints that could trigger vulnerable code
- Configuration classes that enable vulnerable features
- Integration points with the affected library

#### 7.2 Map Existing Test Coverage

Search for existing tests that cover the affected functionality:

```bash
# Find tests related to affected classes
grep -r "{AffectedClassName}" --include="*Test.java" --include="*Test.kt" --include="*Spec.kt" src/test/

# Find integration tests for affected endpoints
grep -r "{endpoint-path}" --include="*Test.java" --include="*IT.java" src/test/

# Find tests that use the vulnerable library directly
grep -r "import.*{vulnerable.package}" --include="*Test.java" --include="*Test.kt" src/test/
```

#### 7.3 Generate Test Commands

Provide specific test commands based on the project's test framework:

**For Gradle projects:**
```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests "{AffectedClassNameTest}"

# Run tests with specific tag/category
./gradlew test -Pinclude-tags="security"

# Run integration tests
./gradlew integrationTest
```

**For Maven projects:**
```bash
# Run all tests
mvn test

# Run specific test class
mvn test -Dtest={AffectedClassNameTest}

# Run integration tests
mvn verify -Pintegration-tests

# Run with security profile
mvn test -Psecurity-tests
```

#### 7.4 CVE-Type-Specific Verification

Generate manual verification steps based on the CVE type:

**Path Traversal (CWE-22):**
```
1. Test that path traversal attempts are blocked:
   - Try accessing: GET /api/files/../../../etc/passwd
   - Try accessing: GET /static/..%2F..%2F..%2Fetc/passwd
   - Verify 400 Bad Request or sanitized path response
2. Review logs for path traversal attempt logging
3. Verify no file system access outside allowed directories
```

**Deserialization (CWE-502):**
```
1. Test with malicious payload if safe to do so in test environment
2. Verify type filtering/allowlisting is active
3. Check that untrusted input sources have deserialization controls
4. Monitor for unexpected class instantiation in logs
```

**Resource Leak / Cleanup (CWE-404, CWE-459):**
```
1. Monitor temp directory before and after error scenarios:
   - Linux/Mac: watch -n 1 'ls -la /tmp | grep {pattern} | wc -l'
   - Windows: dir %TEMP% /s | find /c "{pattern}"
2. Trigger error conditions that previously caused leaks
3. Verify resources are cleaned up after errors
4. Check memory usage doesn't grow unexpectedly
```

**Injection (CWE-89, CWE-79, CWE-77):**
```
1. Test with injection payloads appropriate to the type:
   - SQL: ' OR '1'='1'; DROP TABLE--
   - XSS: <script>alert('xss')</script>
   - Command: ; cat /etc/passwd
2. Verify input is sanitized or rejected
3. Check output encoding is applied
4. Review parameterized query usage
```

**Denial of Service (CWE-400):**
```
1. Test with large/malformed input that previously caused issues
2. Monitor CPU/memory during stress scenarios
3. Verify timeouts and limits are enforced
4. Check that error handling doesn't amplify the issue
```

**Authentication/Authorization (CWE-287, CWE-863):**
```
1. Test access with invalid/expired credentials
2. Verify privilege escalation attempts are blocked
3. Check session handling after fix
4. Test with edge cases (null user, empty roles, etc.)
```

#### 7.5 Present Test Plan

Include the test plan in the remediation output:

```
## Test Plan

### Automated Tests
Run these commands to verify the fix doesn't break existing functionality:

\`\`\`bash
# Run affected test classes
./gradlew test --tests "WebConfigTest"
./gradlew test --tests "FileControllerTest"

# Run integration tests for affected endpoints
./gradlew integrationTest --tests "*FileUpload*"
\`\`\`

### Manual Verification (Path Traversal)
Since this CVE is a path traversal vulnerability, verify the fix with:

1. **Positive test**: Confirm normal file access still works
   - `curl http://localhost:8080/files/report.pdf` → Should return file

2. **Negative test**: Confirm path traversal is blocked
   - `curl http://localhost:8080/files/../../../etc/passwd` → Should return 400

3. **Encoded test**: Confirm encoded traversal is blocked
   - `curl http://localhost:8080/files/..%2F..%2Fetc/passwd` → Should return 400

### Security Scan
After tests pass, run security scan to confirm remediation:
\`\`\`bash
./gradlew dependencyCheckAnalyze
# OR
mvn org.owasp:dependency-check-maven:check
\`\`\`
```

### Phase 8: Clean and Refresh Dependencies

**IMPORTANT**: After updating build files and generating the test plan, run a clean dependency refresh to ensure the correct versions are pulled.

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

### Phase 9: Verify and Report

After the clean dependency refresh:

1. Run dependency verification to confirm the new version:
```bash
# Maven
mvn dependency:tree -Dincludes=*:{artifactId}*

# Gradle
./gradlew dependencyInsight --dependency {library-name}
```

2. Provide a summary for the user's Jira ticket that includes the test plan:

```
## Remediation Summary (for Jira)

**CVE**: CVE-XXXX-XXXXX
**Severity**: CRITICAL
**Action Taken**: Updated {library} from {old-version} to {new-version}
**Version Selection**: {new-version} is latest patch in {minor}.x line (minimum fix was {min-fix-version})
**CVE Verification**: Confirmed no new CVEs in versions {min-fix-version} through {new-version}
**Files Modified**: pom.xml (or build.gradle)
**Dependencies Refreshed**: Yes - ran clean build with dependency refresh
**Verification**: Dependency tree confirms new version

## Test Plan

### Automated Tests
{Specific test commands for affected code paths}

### Manual Verification ({CVE Type})
{CVE-type-specific verification steps}

### Security Scan
{Security scan command to confirm remediation}
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

### Recommend Latest Safe Patch
- Always recommend the latest patch in the compatible minor version line, not just the minimum fix
- Verify no new CVEs exist between the minimum fix and the latest patch
- Explain the version selection rationale to the user

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

### Generate Specific Test Plans
- Never provide generic "run your tests" guidance
- Identify specific test classes and commands based on affected code paths
- Include CVE-type-specific manual verification steps (path traversal, deserialization, etc.)
- Provide concrete verification commands the team can execute

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
[Queries Maven Central for latest versions]
[Verifies no CVEs in newer patch versions]

## CVE Analysis: CVE-2024-38816

**Severity**: CRITICAL (CVSS 9.8)
**Description**: Path traversal vulnerability in Spring Framework WebMvc.fn and WebFlux.fn
**Affected**: spring-webmvc < 5.3.31, < 6.0.15, < 6.1.3

**Your Project**:
- Current version: spring-webmvc 5.3.27 (via Spring Boot 2.7.12)
- Dependency type: Transitive (through spring-boot-starter-web)
- Exposure: MEDIUM - Found RouterFunction usage in WebConfig.java:45

**Recommendation**: Even with medium exposure, I recommend fixing this to eliminate the vulnerability entirely.

## Remediation Plan

**Strategy**: Dependency Management Override (lowest risk)
**Minimum Fix Version**: 5.3.31
**Recommended Version**: 5.3.39 (latest patch in 5.3.x line)
**Risk**: MINIMAL - patch update, no breaking changes

**Version Selection Rationale**:
- Minimum fix is 5.3.31, but 5.3.39 is the latest in the 5.3.x line
- Verified no new CVEs in versions 5.3.31 through 5.3.39
- Recommending latest patch to include all subsequent security and bug fixes

**What I'll do**:
1. Add spring-webmvc 5.3.39 to dependencyManagement in pom.xml
2. Run clean dependency refresh
3. Verify with dependency tree
4. Generate specific test plan for path traversal verification

Proceed with remediation?

User: Yes

Claude: [Uses Edit tool to update pom.xml]
[Runs ./gradlew clean build --refresh-dependencies]
[Runs dependency verification]

Done! Here's the summary for your Jira ticket:

**CVE**: CVE-2024-38816
**Severity**: CRITICAL
**Action**: Added dependency management override for spring-webmvc 5.3.39 (latest patch in 5.3.x line, minimum fix was 5.3.31)
**Files Modified**: pom.xml
**Dependencies Refreshed**: Yes
**Verified**: Dependency tree confirms 5.3.39, no new CVEs in version range

## Test Plan

### Automated Tests
Run these commands to verify the fix doesn't break existing functionality:

\`\`\`bash
# Run tests for affected code
./gradlew test --tests "WebConfigTest"
./gradlew test --tests "StaticResourceTest"
\`\`\`

### Manual Verification (Path Traversal)
Since this CVE is a path traversal vulnerability (CWE-22), verify the fix with:

1. **Positive test**: Confirm normal static resource access still works
   - `curl http://localhost:8080/static/styles.css` → Should return file

2. **Negative test**: Confirm path traversal is blocked
   - `curl http://localhost:8080/static/../../../etc/passwd` → Should return 400

3. **Encoded test**: Confirm encoded traversal is blocked
   - `curl http://localhost:8080/static/..%2F..%2Fetc/passwd` → Should return 400

### Security Scan
\`\`\`bash
./gradlew dependencyCheckAnalyze
\`\`\`
```
