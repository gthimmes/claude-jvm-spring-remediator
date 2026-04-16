---
name: remediate
description: Analyze CVE vulnerabilities in Java/Kotlin Spring Boot codebases, assess actual exposure, and implement minimal-impact remediation. Supports single CVE, batch scan from Dependabot/OWASP/Snyk, and automated PR creation. Use when user provides a CVE ID, asks to scan for vulnerabilities, or wants to remediate security issues.
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, Task, Agent, WebFetch, WebSearch
---

# JVM Spring CVE Remediator

A Claude Code skill that analyzes CVE vulnerabilities in Java/Kotlin Spring Boot applications, assesses actual code exposure, and implements minimal-impact remediation.

## What This Skill Does

When invoked with a CVE ID or scanner source, you (Claude) will:

0. **Scan for vulnerabilities** (scan mode) — pull alerts from Dependabot, OWASP, or Snyk; group by library; let user choose scope
1. **Fetch CVE details** from NVD and security advisories
2. **Analyze the project's dependencies** to determine if affected
3. **Assess actual exposure** by scanning the codebase for usage of vulnerable code paths
4. **Recommend remediation strategy** prioritizing minimal-impact updates
5. **Present a remediation plan** for user approval
6. **Implement the fix** by updating build files once approved
7. **Generate a specific test plan** based on affected code paths and CVE type
8. **Clean and refresh dependencies** to ensure correct versions are pulled
9. **Verify the remediation** and provide Jira-ready summary with test plan
10. **Create a pull request** (with `--create-pr`) — branch, commit, and open a PR with rich context

**Advanced Mode: Parallel Strategy Testing**
- When explicitly requested, test 4 dependency override strategies in parallel
- Each strategy runs on an isolated temporary branch with full build validation
- Compare all results and apply the cleanest working solution
- Ideal for complex dependency scenarios or when standard approach is unclear

## Input Format

The user will provide a single CVE identifier:
```
/remediate CVE-2024-38816
```

Or simply mention a CVE in conversation:
```
Can you fix CVE-2024-38816 in my project?
```

**Scan mode (batch from scanner output):**
```
/remediate scan                    # Auto-detect scanner output in project
/remediate scan dependabot         # Read from GitHub Dependabot alerts
/remediate scan owasp              # Read OWASP Dependency-Check report
/remediate scan snyk               # Read Snyk JSON output
```

**With PR creation:**
```
/remediate CVE-2024-38816 --create-pr
/remediate scan --create-pr
/remediate scan dependabot --create-pr
```

**For parallel strategy testing:**
```
I need to remediate CVE-2024-38816 affecting spring-webmvc in our Gradle project
```

Or explicitly request parallel testing:
```
/remediate CVE-2024-38816 using parallel strategies
```

Or ask for comparison of approaches:
```
Test multiple dependency override strategies for CVE-2024-38816
```

## Execution Workflow

### Phase 0: Scanner Integration (Scan Mode Only)

**This phase runs only when the user invokes `/remediate scan`.** If the user provides a single CVE ID, skip directly to Phase 1.

#### Phase 0A: Scanner Detection & Data Collection

Determine the scanner source and collect all open vulnerability alerts.

**Auto-detect (`/remediate scan`):**
Check sources in this order and use the first one that returns results:
1. Is this a GitHub repo with Dependabot enabled? Try Dependabot alerts.
2. Look for OWASP Dependency-Check report files in the project.
3. Look for Snyk report files in the project.
4. If nothing found, tell the user no scanner output was detected and suggest running one:
   ```
   No scanner output detected. You can generate one with:
   - OWASP: mvn org.owasp:dependency-check-maven:check (or ./gradlew dependencyCheckAnalyze)
   - Snyk: snyk test --json > snyk-report.json
   - Or enable Dependabot alerts on your GitHub repository
   ```

**Dependabot (`/remediate scan dependabot`):**
```bash
# Get the repo owner/name from git remote
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')

# Fetch open Dependabot alerts for the Maven/Gradle ecosystem
gh api "repos/${REPO}/dependabot/alerts" \
  --jq '[.[] | select(.state=="open") | select(.dependency.package.ecosystem=="maven" or .dependency.package.ecosystem=="gradle")] | sort_by(.security_advisory.cvss.score) | reverse'
```

Extract from each alert:
- CVE ID: `.security_advisory.cve_id`
- Severity: `.security_advisory.severity`
- CVSS score: `.security_advisory.cvss.score`
- Package name: `.dependency.package.name`
- Current version: `.dependency.scope` and manifest path
- Fixed version: `.security_advisory.vulnerabilities[].first_patched_version.identifier`
- Advisory summary: `.security_advisory.summary`

**OWASP Dependency-Check (`/remediate scan owasp`):**
```bash
# Look for report files
find . -name "dependency-check-report.json" -o -name "dependency-check-report.xml" 2>/dev/null
```

If JSON report found, parse it:
- CVE IDs: `dependencies[].vulnerabilities[].name`
- Severity: `dependencies[].vulnerabilities[].cvssv3.baseSeverity` (or cvssv2)
- CVSS score: `dependencies[].vulnerabilities[].cvssv3.baseScore`
- Package: `dependencies[].fileName` and `dependencies[].packages[].id`
- Description: `dependencies[].vulnerabilities[].description`

If only XML report found, parse the equivalent XML structure.

If no report file exists, offer to run the scan:
```bash
# Maven
mvn org.owasp:dependency-check-maven:check -Dformat=JSON

# Gradle (if plugin configured)
./gradlew dependencyCheckAnalyze
```

**Snyk (`/remediate scan snyk`):**
```bash
# Look for existing report files
find . -name "snyk-report.json" -o -name "snyk-results.json" -o -name "snyk-test.json" 2>/dev/null
```

If report found, parse it:
- CVE IDs: `vulnerabilities[].identifiers.CVE[]`
- Severity: `vulnerabilities[].severity`
- CVSS score: `vulnerabilities[].cvssScore`
- Package: `vulnerabilities[].packageName`
- Current version: `vulnerabilities[].version`
- Fixed version: `vulnerabilities[].fixedIn[]`
- Description: `vulnerabilities[].title`

If no report file exists and `snyk` CLI is available, offer to run:
```bash
snyk test --json > snyk-report.json
```

#### Phase 0B: Triage, Deduplication & Prioritization

After collecting all CVEs from the scanner:

1. **Filter to JVM/Spring dependencies only** — ignore non-Java vulnerabilities if the scanner reports them

2. **Group CVEs by library** — multiple CVEs may affect the same library and be fixed by one upgrade:
   ```
   spring-webmvc: [CVE-2024-38816, CVE-2024-38819]  → one upgrade fixes both
   jackson-databind: [CVE-2024-12345]                 → single CVE
   ```

3. **Determine group severity** — use the highest severity CVE in each group

4. **Sort groups by severity** — CRITICAL first, then HIGH, MEDIUM, LOW

5. **Present summary table** to the user:

```
## Scan Results: {N} CVEs Found ({M} Libraries)

| # | Library | CVEs | Highest Severity | Current | Suggested Action |
|---|---------|------|-----------------|---------|-----------------|
| 1 | spring-webmvc | CVE-2024-38816, CVE-2024-38819 | CRITICAL (9.8) | 5.3.27 | Upgrade (fixes both) |
| 2 | jackson-databind | CVE-2024-12345 | HIGH (7.5) | 2.14.1 | Upgrade |
| 3 | snakeyaml | CVE-2024-11111 | HIGH (7.2) | 1.33 | Upgrade |
| 4 | commons-io | CVE-2024-99999 | MEDIUM (5.3) | 2.11.0 | Upgrade |

**Options:**
- "all" — remediate everything
- "all critical" or "all high" — remediate by severity threshold
- "1,2" — remediate specific libraries by number
- "skip 4" — remediate all except specific libraries
```

#### Phase 0C: Batch Execution Loop

After the user selects which libraries to remediate:

1. **For each selected library group**, run the standard Phases 1-9 workflow:
   - Use the highest-severity CVE ID for NVD lookup in Phase 1
   - In Phase 4 (version selection), verify the recommended version fixes ALL CVEs in the group
   - Track all changes and test plans across iterations

2. **Accumulate results** across all library groups:
   - List of all files modified
   - Combined test plan covering all remediations
   - Any failures or issues encountered

3. **At the end, produce a consolidated Jira summary**:

```
## Batch Remediation Summary (for Jira)

**Date**: {date}
**Scanner**: {Dependabot/OWASP/Snyk}
**CVEs Remediated**: {N}
**Libraries Updated**: {M}

| Library | Old Version | New Version | CVEs Fixed | Severity |
|---------|-------------|-------------|------------|----------|
| spring-webmvc | 5.3.27 | 5.3.39 | CVE-2024-38816, CVE-2024-38819 | CRITICAL |
| jackson-databind | 2.14.1 | 2.14.3 | CVE-2024-12345 | HIGH |

**Files Modified**: pom.xml, build.gradle
**Dependencies Refreshed**: Yes

## Combined Test Plan
{Consolidated test commands and manual verification steps from all remediations}

## Security Scan
{Command to re-run the original scanner to confirm all CVEs resolved}
```

4. **If `--create-pr` flag is present**, proceed to Phase 10.

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

**CRITICAL: You must check ALL modules and ALL configurations, not just the main module.**

  1. First, find every module that has the vulnerable dependency in ANY configuration:

  \`\`\`bash
  ./gradlew allDependencyInsight --dependency {library-name} 2>&1 | grep -E "(commons-fileupload|Project |configuration )"
  \`\`\`

  If `allDependencyInsight` is not available, iterate over every module:

  \`\`\`bash
  for module in $(./gradlew projects --quiet | grep "Project" | sed "s/.*'\(.*\)'/\1/"); do
    for config in runtimeClasspath testRuntimeClasspath; do
      ./gradlew ${module}:dependencyInsight --dependency {library-name} --configuration ${config} 2>&1
    done
  done
  \`\`\`

  2. Record EVERY module + configuration where the vulnerable version appears.
  3. Your remediation must address ALL of them, not just the first one you find.

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

### Phase 4A: Alternative Parallel Strategy Testing (Advanced)

**When to use this approach:**
- User explicitly requests parallel strategy testing
- User wants to compare multiple override approaches
- Standard strategy selection is unclear or risky
- Project has complex dependency constraints

**Parallel Strategy Testing Workflow:**

When a user says "I need to remediate [CVE-ID] affecting [LIBRARY] in our [Gradle/Maven] project" and wants to test multiple approaches in parallel:

1. **Do NOT try approaches sequentially.** Instead, immediately create a TaskCreate plan with 4 parallel strategies to test simultaneously.

2. **The 4 parallel strategies to test:**
   - **Strategy 1**: Override the version property in `ext`/`extra` block
   - **Strategy 2**: Use a `dependencyManagement` block with `force` (Maven) or `strictly()` (Gradle)
   - **Strategy 3**: Use `resolutionStrategy.eachDependency` (Gradle) or `<dependency>` override (Maven)
   - **Strategy 4**: Use dependency constraints with `strictly()` (Gradle) or enforcer plugin (Maven)

3. **For each strategy, spawn a parallel sub-agent using the Task tool that:**
   - Creates the change on a temporary Git branch named `remediate-[CVE-ID]-strategy-[1-4]`
   - Implements the specific dependency override approach
   - Runs the full build via Bash (`./gradlew build` or `mvn clean install`)
   - Checks the resolved dependency tree to confirm the vulnerable version is gone
   - Reports success or failure with the exact error message if it fails
   - Cleans up and returns to the original branch when done

4. **After all agents complete:**
   - Compare results from all 4 strategies
   - Identify which strategies succeeded and which failed
   - Select the cleanest working solution (prioritize simplicity and maintainability)
   - Apply the winning solution to the main branch
   - Show the user the diff of the final applied change
   - Clean up temporary branches

**Example implementation:**

```
User: I need to remediate CVE-2024-38816 affecting spring-webmvc in our Gradle project using parallel strategies

Claude: I'll test 4 parallel dependency override strategies for CVE-2024-38816.

[Uses TaskCreate to create 4 tasks]

[Spawns 4 parallel Task tool invocations, one for each strategy]

Task 1: Test ext property override
Task 2: Test dependencyManagement with strictly()
Task 3: Test resolutionStrategy.eachDependency
Task 4: Test dependency constraints

[Each agent creates temp branch, applies fix, runs build, checks tree]

[After all complete]

## Results Summary

Strategy 1 (ext property): ✅ SUCCESS - Build passed, spring-webmvc 5.3.39 confirmed
Strategy 2 (strictly()): ✅ SUCCESS - Build passed, spring-webmvc 5.3.39 confirmed
Strategy 3 (eachDependency): ❌ FAILED - Conflict with Spring Boot BOM
Strategy 4 (constraints): ✅ SUCCESS - Build passed, spring-webmvc 5.3.39 confirmed

## Recommendation

Applying Strategy 1 (ext property override) - cleanest and most maintainable approach.

[Shows diff of changes]
[Applies to main branch]
[Cleans up temp branches]
```

**Key principles for parallel testing:**
- Always run all 4 strategies in parallel, never sequentially
- Each strategy gets its own isolated temporary branch
- Each agent reports detailed success/failure with exact error messages
- Compare all results before selecting the winner
- Prioritize simplicity and Spring Boot compatibility when choosing winner
- Always clean up temporary branches after completion

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
 **CRITICAL: Verify the old vulnerable version appears NOWHERE in the entire project.**

  1. For every module + configuration identified in Phase 2, re-run dependencyInsight and confirm:
     - The new version resolves correctly
     - The old version does NOT appear without an upgrade arrow (->)

  2. Specifically search for any remaining references to the old version:

  \`\`\`bash
  ./gradlew :module:dependencyInsight --dependency {library-name} --configuration testRuntimeClasspath 2>&1 | grep "{old-version}"
  \`\`\`

  If the old version still appears without "-> {new-version}", the remediation is INCOMPLETE. Do not proceed to the summary.

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

### Phase 10: Pull Request Creation (when `--create-pr` is used)

**This phase runs only when the user includes `--create-pr` in the command or explicitly asks for a PR during remediation.** If the flag is not present, skip this phase.

#### 10.1 Create a Branch

If currently on the main/default branch, create a remediation branch:

```bash
# Single CVE
git checkout -b remediate/CVE-2024-38816

# Batch scan mode — use date to avoid conflicts
git checkout -b remediate/security-scan-$(date +%Y-%m-%d)
```

If the user is already on a feature branch, use the current branch.

#### 10.2 Stage and Commit Changes

Stage only the build files that were modified during remediation:

**Single CVE commit:**
```bash
# Stage modified build files
git add pom.xml  # or build.gradle / build.gradle.kts

# Commit with structured message
git commit -m "fix(security): remediate CVE-2024-38816 in spring-webmvc

Updated spring-webmvc from 5.3.27 to 5.3.39 (latest patch in 5.3.x line).
Minimum fix version was 5.3.31. Verified no new CVEs in range.

Severity: CRITICAL (CVSS 9.8)
Exposure: MEDIUM
Strategy: Dependency management override"
```

**Batch scan commit:**
```bash
git add pom.xml build.gradle build.gradle.kts 2>/dev/null

git commit -m "fix(security): remediate {N} CVEs across {M} libraries

- spring-webmvc: 5.3.27 → 5.3.39 (fixes CVE-2024-38816, CVE-2024-38819)
- jackson-databind: 2.14.1 → 2.14.3 (fixes CVE-2024-12345)
- snakeyaml: 1.33 → 1.33.2 (fixes CVE-2024-11111)
- commons-io: 2.11.0 → 2.11.1 (fixes CVE-2024-99999)

Scanner: {Dependabot/OWASP/Snyk}
All changes verified via dependency tree analysis."
```

#### 10.3 Push and Create PR

Push the branch and create a pull request using `gh`:

```bash
git push -u origin HEAD
```

**Single CVE PR:**
```bash
gh pr create \
  --title "fix(security): remediate CVE-{ID} in {library}" \
  --body "$(cat <<'EOF'
## Security Remediation

### Vulnerability
| Field | Value |
|-------|-------|
| CVE | {CVE-ID} |
| Severity | {SEVERITY} (CVSS {score}) |
| Library | {groupId}:{artifactId} |
| Previous Version | {old-version} |
| Fixed Version | {new-version} |
| Exposure | {level} - {explanation} |

### What Changed
- Updated `{library}` from {old-version} to {new-version} via {strategy}
- Version {new-version} is the latest patch in the {minor}.x line (minimum fix: {min-fix-version})
- Verified no new CVEs exist in versions {min-fix-version} through {new-version}

### Test Plan

**Automated Tests:**
\`\`\`bash
{specific test commands from Phase 7}
\`\`\`

**Manual Verification ({CVE Type} - {CWE-ID}):**
{CVE-type-specific verification steps from Phase 7}

**Security Scan:**
\`\`\`bash
{scanner command to confirm remediation}
\`\`\`

### Verification
- [x] Dependency tree confirms new version
- [x] Old version no longer present in any module/configuration
- [x] Clean build with dependency refresh succeeded

---
*Remediated by [jvm-spring-remediator](https://github.com/gthimmes/claude-jvm-spring-remediator)*
EOF
)" \
  --label "security,dependencies"
```

Add severity-specific label:
```bash
# Add severity label based on CVSS
gh pr edit --add-label "critical"   # CVSS >= 9.0
gh pr edit --add-label "high"       # CVSS >= 7.0
gh pr edit --add-label "medium"     # CVSS >= 4.0
gh pr edit --add-label "low"        # CVSS < 4.0
```

**Batch scan PR:**
```bash
gh pr create \
  --title "fix(security): remediate {N} CVEs across {M} libraries" \
  --body "$(cat <<'EOF'
## Security Remediation — Batch Scan

**Scanner**: {Dependabot/OWASP/Snyk}
**Date**: {date}
**CVEs Remediated**: {N}
**Libraries Updated**: {M}

### Vulnerabilities Fixed

| Library | Old Version | New Version | CVEs Fixed | Severity |
|---------|-------------|-------------|------------|----------|
| spring-webmvc | 5.3.27 | 5.3.39 | CVE-2024-38816, CVE-2024-38819 | CRITICAL |
| jackson-databind | 2.14.1 | 2.14.3 | CVE-2024-12345 | HIGH |
| snakeyaml | 1.33 | 1.33.2 | CVE-2024-11111 | HIGH |
| commons-io | 2.11.0 | 2.11.1 | CVE-2024-99999 | MEDIUM |

### What Changed
{Bullet list of each library update with strategy used}

### Combined Test Plan

**Automated Tests:**
\`\`\`bash
{consolidated test commands for all affected code paths}
\`\`\`

**Manual Verification:**
{consolidated CVE-type-specific steps, grouped by type}

**Security Scan (re-run original scanner):**
\`\`\`bash
{command to re-run the scanner and confirm all CVEs resolved}
\`\`\`

### Verification
- [x] Dependency tree confirms all new versions
- [x] Old versions no longer present in any module/configuration
- [x] Clean build with dependency refresh succeeded
- [x] {N}/{N} CVEs resolved

---
*Remediated by [jvm-spring-remediator](https://github.com/gthimmes/claude-jvm-spring-remediator)*
EOF
)" \
  --label "security,dependencies"
```

#### 10.4 Report PR URL

After creating the PR, display the URL to the user:

```
## Pull Request Created

**PR**: {URL returned by gh pr create}
**Branch**: remediate/CVE-2024-38816
**Labels**: security, dependencies, critical

The PR includes the full remediation summary, test plan, and verification checklist.
```

**Important notes for PR creation:**
- Always ask for confirmation before pushing and creating the PR
- If `gh` CLI is not available or not authenticated, inform the user and provide the branch/commit for manual PR creation
- If labels don't exist in the repository, `gh pr edit --add-label` may fail silently — this is acceptable
- Never force-push; if the branch already exists, inform the user and ask how to proceed

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

### When to Use Parallel Strategy Testing
Use the parallel strategy testing approach (Phase 4A) when:
- User explicitly requests testing multiple approaches
- User says "test parallel strategies" or "compare approaches"
- Standard remediation encounters unexpected conflicts
- Project has complex Spring Boot BOM interactions
- You're uncertain which override method will work best

Do NOT use parallel testing by default - it's more resource-intensive. Use the standard workflow (Phase 4) unless there's a clear reason to test multiple strategies.

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

### Scan Mode: Group and Deduplicate
- Always group CVEs by library before presenting to the user
- One version bump that fixes 3 CVEs is better than 3 separate changes
- Present the summary table and let the user choose scope before starting work
- In batch mode, track cumulative results and produce a single consolidated summary at the end

### PR Creation: Rich Context for Reviewers
- Include enough detail in the PR body that a reviewer can approve without additional research
- Always include: CVE ID, severity, exposure assessment, version rationale, test plan, verification checklist
- Use structured tables for readability
- For batch PRs, list all CVEs in a summary table and consolidate test plans
- Always confirm with the user before pushing and creating the PR
- If `gh` CLI is not available, provide the commit and branch info so the user can create the PR manually

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
- **No scanner output found**: Suggest running a scanner (OWASP, Snyk) or enabling Dependabot
- **`gh` CLI not available**: For Dependabot integration or PR creation, inform user to install/authenticate `gh` CLI and provide manual alternatives
- **Dependabot alerts API returns empty**: Check if Dependabot is enabled on the repo, or if all alerts are already resolved
- **PR creation fails**: Provide the branch name and commit so the user can create the PR manually
- **Scanner report format unrecognized**: Ask user to verify the report was generated correctly and provide the file path

## Example Session

### Standard Remediation

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

### Parallel Strategy Testing

```
User: I need to remediate CVE-2024-38816 affecting spring-webmvc in our Gradle project using parallel strategies

Claude: I'll test 4 parallel dependency override strategies for CVE-2024-38816.

[Creates 4 tasks with TaskCreate]

Task 1: Test ext property override (branch: remediate-CVE-2024-38816-strategy-1)
Task 2: Test dependencyManagement with strictly() (branch: remediate-CVE-2024-38816-strategy-2)
Task 3: Test resolutionStrategy.eachDependency (branch: remediate-CVE-2024-38816-strategy-3)
Task 4: Test dependency constraints (branch: remediate-CVE-2024-38816-strategy-4)

[Spawns 4 parallel Task tool invocations]
[Each agent creates temp branch, applies fix, runs build, checks tree]

## Parallel Testing Results

✅ **Strategy 1: ext property override**
- Branch: remediate-CVE-2024-38816-strategy-1
- Build: PASSED
- Verification: spring-webmvc 5.3.39 confirmed in dependency tree
- Approach: Simple and maintainable

✅ **Strategy 2: dependencyManagement with strictly()**
- Branch: remediate-CVE-2024-38816-strategy-2
- Build: PASSED
- Verification: spring-webmvc 5.3.39 confirmed
- Approach: Explicit version enforcement

❌ **Strategy 3: resolutionStrategy.eachDependency**
- Branch: remediate-CVE-2024-38816-strategy-3
- Build: FAILED
- Error: Conflict with Spring Boot BOM - version downgraded back to 5.3.27
- Reason: BOM takes precedence over resolution strategy in this project

✅ **Strategy 4: dependency constraints**
- Branch: remediate-CVE-2024-38816-strategy-4
- Build: PASSED
- Verification: spring-webmvc 5.3.39 confirmed
- Approach: Modern Gradle constraints API

## Recommendation

Applying **Strategy 1 (ext property override)** to main branch.

**Rationale**:
- Cleanest and most maintainable approach
- Compatible with Spring Boot BOM
- Easy to understand and modify later
- Least verbose syntax

## Changes Applied

\`\`\`diff
diff --git a/build.gradle.kts b/build.gradle.kts
index 1234567..abcdefg 100644
--- a/build.gradle.kts
+++ b/build.gradle.kts
@@ -1,3 +1,7 @@
+ext {
+    set("spring-webmvc.version", "5.3.39")
+}
+
 plugins {
     id("org.springframework.boot") version "2.7.12"
 }
\`\`\`

[Cleans up temporary branches]

Done! spring-webmvc updated to 5.3.39 using ext property override.
```

### Scanner Integration (Batch Mode)

```
User: /remediate scan dependabot

Claude: I'll fetch open Dependabot alerts for your repository.

[Runs gh api to fetch Dependabot alerts]
[Filters to Maven/Gradle ecosystem]
[Groups CVEs by library]

## Scan Results: 5 CVEs Found (3 Libraries)

| # | Library | CVEs | Highest Severity | Current | Suggested Action |
|---|---------|------|-----------------|---------|-----------------|
| 1 | spring-webmvc | CVE-2024-38816, CVE-2024-38819 | CRITICAL (9.8) | 5.3.27 | Upgrade (fixes both) |
| 2 | jackson-databind | CVE-2024-12345 | HIGH (7.5) | 2.14.1 | Upgrade |
| 3 | commons-io | CVE-2024-99999, CVE-2024-99998 | MEDIUM (5.3) | 2.11.0 | Upgrade (fixes both) |

Remediate all? Or select specific libraries (e.g., "1,2" or "all critical")?

User: all critical and high

Claude: Remediating libraries #1 (spring-webmvc) and #2 (jackson-databind).

### Remediating spring-webmvc (2 CVEs)...

[Runs standard Phases 1-9 for spring-webmvc]
[Fetches CVE details, analyzes dependencies, assesses exposure]
[Recommends 5.3.39, verifies it fixes both CVEs]
[Updates pom.xml, refreshes dependencies, verifies]

✅ spring-webmvc: 5.3.27 → 5.3.39 (fixes CVE-2024-38816 and CVE-2024-38819)

### Remediating jackson-databind (1 CVE)...

[Runs standard Phases 1-9 for jackson-databind]
[Fetches CVE details, analyzes dependencies, assesses exposure]
[Recommends 2.14.3, verifies fix]
[Updates pom.xml, refreshes dependencies, verifies]

✅ jackson-databind: 2.14.1 → 2.14.3 (fixes CVE-2024-12345)

## Batch Remediation Summary (for Jira)

**Scanner**: Dependabot
**CVEs Remediated**: 3
**Libraries Updated**: 2

| Library | Old Version | New Version | CVEs Fixed | Severity |
|---------|-------------|-------------|------------|----------|
| spring-webmvc | 5.3.27 | 5.3.39 | CVE-2024-38816, CVE-2024-38819 | CRITICAL |
| jackson-databind | 2.14.1 | 2.14.3 | CVE-2024-12345 | HIGH |

**Files Modified**: pom.xml
**Dependencies Refreshed**: Yes

## Combined Test Plan

### Automated Tests
./gradlew test --tests "WebConfigTest"
./gradlew test --tests "JacksonConfigTest"

### Manual Verification
1. Path Traversal (spring-webmvc): Test traversal attempts are blocked
2. Deserialization (jackson-databind): Verify type filtering active

### Security Scan
./gradlew dependencyCheckAnalyze
```

### PR Creation

```
User: /remediate CVE-2024-38816 --create-pr

Claude: I'll analyze CVE-2024-38816 and create a PR with the fix.

[Runs standard Phases 1-9]
[Fetches CVE data, analyzes dependencies, assesses exposure]
[Recommends spring-webmvc 5.3.39]
[Implements fix, refreshes dependencies, verifies]

Remediation complete. Ready to create PR.

I'll create a branch, commit the changes, and open a PR. Proceed?

User: Yes

Claude: [Creates branch remediate/CVE-2024-38816]
[Commits pom.xml changes with structured message]
[Pushes branch and creates PR via gh]

## Pull Request Created

**PR**: https://github.com/your-org/your-repo/pull/142
**Branch**: remediate/CVE-2024-38816
**Labels**: security, dependencies, critical

The PR includes the full remediation summary, test plan, and verification checklist.
```
