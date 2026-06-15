# PSSW OGH POM Generator Pipeline
## Comprehensive Use Case Documentation

**Platform:** AAVA Analytics Platform (`aava-dlt-staging.avateam.io`)  
**Pipeline ID:** 4510  
**Status:** ✅ Validated on Public GitHub — HP GHE Migration Pending  
**Last Updated:** 2026-06-11  
**Prepared by:** AAVA Analytics Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Solution Architecture](#3-solution-architecture)
4. [Pipeline Components](#4-pipeline-components)
5. [Storage Design](#5-storage-design)
6. [Agent Specifications](#6-agent-specifications)
7. [Tool Specification — XAMLTimestampedReportWriter](#7-tool-specification--xamltimestampedreportwriter)
8. [Inter-Agent Schema](#8-inter-agent-schema)
9. [GitHub Write Logic](#9-github-write-logic)
10. [Validated Run Evidence](#10-validated-run-evidence)
11. [Error Reference](#11-error-reference)
12. [HP GitHub Enterprise Migration](#12-hp-github-enterprise-migration)
13. [Key Learnings and Architectural Decisions](#13-key-learnings-and-architectural-decisions)
14. [Open Items and Roadmap](#14-open-items-and-roadmap)

---

## 1. Executive Summary

The PSSW OGH POM Generator pipeline is a multi-agent AI automation system built on the AAVA platform. It analyses XAML UI definition files from the OMEN Gaming Hub Windows application, extracts UI element metadata, generates Java Page Object Model (POM) classes for test automation, identifies elements missing `AutomationId` attributes, and writes structured reports to a GitHub repository.

The pipeline processes one XAML file per execution and produces a four-section report committed to GitHub under a UTC-timestamped folder structure. Each run is fully isolated — re-running after a developer code fix produces a new report in a new folder, enabling direct before/after comparison.

**Validated output for `PrimeBarControl.xaml`:**
- 7 POM methods generated with `Constants.ACCESSIBILITY_ID` locator strategy
- 10 elements identified without `AutomationId` (HIGH/MEDIUM/LOW priority)
- 100% accuracy score, PASS WITH WARNINGS verdict
- Report committed to `omen-reports/reports/2026_06_11_05_40/PrimeBarControl.txt`

---

## 2. Problem Statement

The HP OMEN Gaming Hub Windows application is built in WPF/XAML. Test automation requires Page Object Models that map UI elements to their `AutomationId` attributes. The manual process of:

1. Reading XAML files from the source repository
2. Identifying which elements have `AutomationId` and which do not
3. Writing POM Java classes for automatable elements
4. Reporting gaps to the development team for remediation

was time-consuming, error-prone, and produced no historical record of automation coverage changes over time.

**Secondary problem:** Earlier pipeline versions appended all results to a single `xaml_automation_report.json` file. This caused:

| Problem | Impact |
|---|---|
| Report grows indefinitely | File size increases with every run |
| Deduplication complexity | Silent skips when re-running same XAML after code fixes |
| No run comparison | Developer cannot diff before/after a code change |
| Merge conflicts | Concurrent writes to same file cause PUT 409 errors |

---

## 3. Solution Architecture

```
Developer / Orchestrator
        |
        | XAML file path + repo config
        v
┌─────────────────────────────────────────┐
│           AAVA Pipeline 4510            │
│                                         │
│  Agent 3: XAML Automation ID Analyzer   │
│  (GPT-4.1, temp=0.1, ID: 1084)         │
│         |                               │
│         | JSON: file_name + report_content
│         v                               │
│  Agent 4: XAMLTimestampedReportWriter   │
│  (Claude Sonnet via Bedrock)            │
│         |                               │
└─────────|───────────────────────────────┘
          |
          | GitHub Contents API PUT (Bearer auth)
          v
┌─────────────────────────────────────────┐
│  omen-reports (GitHub repository)       │
│                                         │
│  reports/                               │
│  └── 2026_06_11_05_40/                 │
│      └── PrimeBarControl.txt            │
└─────────────────────────────────────────┘
```

### Design Principles

- **One file per invocation** — Agent 4 writes exactly one `.txt` file per execution
- **No read before write** — No GET, no SHA fetch, no deduplication logic
- **Immutable runs** — Once a timestamped folder is written, it is never modified
- **UTC alignment** — Timestamp format `YYYY_MM_DD_HH_MM` matches HP Databricks pipeline conventions
- **Passthrough integrity** — Agent 4 passes `report_content` to the tool unchanged; no LLM interpretation of report data

---

## 4. Pipeline Components

### 4.1 Component Inventory

| Component | Type | ID | Model | Status |
|---|---|---|---|---|
| XAML Automation ID Analyzer | Agent | 1084 | GPT-4.1 (temp=0.1) | ✅ Live |
| XAMLTimestampedReportWriter | Agent | TBD | Claude Sonnet (Bedrock) | 🔧 Register in pipeline 4510 |
| XAMLTimestampedReportWriter | Tool | 1349 (staging) | — | ✅ Validated |
| omen-reports | GitHub repo | — | anthropologenie (public) | ✅ Active |

### 4.2 Pipeline Execution Flow

```
Step 1  Receive XAML file path and repository configuration
Step 2  Agent 3 fetches XAML content from GitHub via GitHubFileFetcher
Step 3  Agent 3 parses XAML structure — extracts elements, AutomationIds, bindings
Step 4  Agent 3 generates four-section report (Sections 1-4)
Step 5  Agent 3 produces JSON handoff: { file_name, report_content }
Step 6  Agent 4 receives JSON from Agent 3
Step 7  Agent 4 invokes XAMLTimestampedReportWriter tool
Step 8  Tool generates UTC timestamp (YYYY_MM_DD_HH_MM)
Step 9  Tool constructs path: reports/<timestamp>/<filename>.txt
Step 10 Tool base64-encodes report_content
Step 11 Tool PUTs to GitHub Contents API (no SHA, new file)
Step 12 GitHub creates folder and file, returns commit SHA
Step 13 Tool returns SUCCESS JSON with file_path and commit_sha
Step 14 Agent 4 returns exact tool response — no modification
```

---

## 5. Storage Design

### 5.1 Repository Structure

```
omen-reports/
└── reports/
    └── 2026_06_10_12_00/           ← Run 1 (first validation)
    │   └── PrimeBarControl.txt
    └── 2026_06_11_05_40/           ← Run 2 (full report validation)
    │   └── PrimeBarControl.txt
    └── 2026_06_11_09_15/           ← Future run (post developer fix)
        └── PrimeBarControl.txt
        └── HomeControl.txt
        └── SignInV2.txt
```

### 5.2 Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Folder name | `YYYY_MM_DD_HH_MM` UTC 24-hour | `2026_06_11_05_40` |
| File name | XAML basename + `.txt` | `PrimeBarControl.txt` |
| Commit message | `automated: xaml report <filename> <timestamp>` | `automated: xaml report PrimeBarControl.txt 2026_06_11_05_40` |

### 5.3 Why This Design Replaces report.json

```
OLD: Single file append pattern
GET report.json → parse → deduplicate → append → PUT
Problem: grows forever, merge conflicts, no run isolation

NEW: Timestamped folder pattern
Generate timestamp → construct path → PUT new file
Result: zero dedup logic, full history, natural diff between runs
```

### 5.4 Re-run Behaviour

| Scenario | Behaviour | Result |
|---|---|---|
| Re-run same XAML, different minute | New folder created | Two folders, two reports |
| Re-run same XAML, same minute | PUT returns 422 (file exists) | FAILED response, no data loss |
| Re-run after developer code fix | New folder with updated report | Both reports preserved for comparison |
| Developer diffs two runs | `git diff reports/2026_06_10_12_00/ reports/2026_06_11_05_40/` | Clean diff of POM and gap changes |

---

## 6. Agent Specifications

### 6.1 Agent 3 — XAML Automation ID Analyzer

| Parameter | Value |
|---|---|
| Agent ID | 1084 |
| Model | GPT-4.1 |
| Temperature | 0.1 (Precise) |
| Tool | GitHubFileFetcher |
| Input | `file_path`, `repo_name`, `repo_url` |
| Output | `{ file_name, report_content }` |

**Validated on:** `TooltipControl.xaml`, `SignInV2.xaml`, `PrimeBarControl.xaml`

**Known behaviour:** When all elements are missing `AutomationId` (e.g. `SignInV2.xaml`), Section 1 (POM class) is empty. Section 2 (Gap Report) contains all elements. This is correct — the gap report is the deliverable when no automatable elements exist.

---

### 6.2 Agent 4 — XAMLTimestampedReportWriter

**AAVA Registration Fields:**

**Agent Name**
```
XAMLTimestampedReportWriter
```

**Agent Details**
```
Agent 4 in the PSSW OGH POM Generator pipeline (ID: 4510). Receives completed XAML
automation analysis output from Agent 3 and writes it as a timestamped .txt report
to the omen-reports GitHub repository. One file per invocation. No read. No dedup. No append.
```

**Practice Area**
```
PSSW OGH Automation Testing
```

**Good At**
```
Invoking XAMLTimestampedReportWriter tool with exact input passthrough, GitHub file
creation via Contents API, deterministic JSON output without LLM interpretation
```

**Knowledge Base**
```
(leave empty — no KB attached)
```

**Guardrails**
```
Do not modify report_content before passing to the tool.
Do not generate any report content, POM code, or gap analysis.
Do not add text outside the JSON response from the tool.
Do not create a SUCCESS response yourself — it must come from the tool.
Do not expose github_token in any output.
Do not retry on failure.
```

**Agent Role**
```
You are Agent 4 of the PSSW OGH POM Generator pipeline. You have exactly one
responsibility: invoke the XAMLTimestampedReportWriter tool with every field from
your input passed through unchanged, and return the tool's exact JSON response.
```

**Goal**
```
Invoke the XAMLTimestampedReportWriter tool and write the XAML automation report as
a .txt file to the omen-reports GitHub repository under a UTC-timestamped folder in
the format reports/YYYY_MM_DD_HH_MM/filename.txt. Return the tool's structured JSON
response containing status, file_path, commit_sha, and report_timestamp_utc.
```

**Back Story**
```
Built as Agent 4 of the PSSW OGH POM Generator pipeline to replace the append-based
XAMLReportWriter pattern. The original design wrote all results into a single growing
JSON file, which prevented run-to-run comparison after code fixes and caused
deduplication complexity. This agent implements the architect-approved timestamped
folder strategy: each run creates a new UTC-named folder, each XAML file produces one
isolated .txt report, and developers can compare any two runs by folder timestamp.
Aligns with HP Databricks UTC storage conventions (YYYY_MM_DD_HH_MM, 24-hour, no seconds).
```

**Description (Behaviour)**
```
You are Agent 4 of the PSSW OGH POM Generator pipeline.

Input contains:
- github_owner
- github_repo
- github_token
- github_base_url
- branch
- file_name
- report_content
- run_timestamp_utc

Input originates from Agent 3 (XAML Automation Analyzer).
report_content already contains the completed report:
- SECTION 1: Generated Java POM Class
- SECTION 2: Automation Gap Report
- SECTION 3: Automation Priority Report
- SECTION 4: POM Validation & Consistency Review

Rules:
1. Always invoke the XAMLTimestampedReportWriter tool.
2. Pass every input field to the tool unchanged.
3. Do not modify report_content.
4. Do not summarize report_content.
5. Do not generate POM code.
6. Do not generate automation gap reports.
7. Do not generate priority reports.
8. Do not generate validation reports.
9. Do not create timestamps yourself.
10. run_timestamp_utc format is YYYY_MM_DD_HH_MM (5 segments, no seconds).
    Example: 2026_06_10_12_00. Pass it unchanged to the tool.
11. Return the exact JSON response produced by the tool.
12. Tool invocation is mandatory.
13. Do not create synthetic SUCCESS responses.
14. A SUCCESS response may only be returned if received directly from the tool.
15. Do not expose github_token in any response.
16. Do not retry failed writes.
17. Do not add explanations, markdown, reasoning, or conversational text.

Process:
Step 1: Receive input.
Step 2: Invoke XAMLTimestampedReportWriter with all fields passed unchanged.
Step 3: Return the exact tool response.

Output JSON only.
```

**Expected Output**
```
Return the exact JSON response returned by the XAMLTimestampedReportWriter tool.
No explanations. No markdown. No reasoning. No conversational text.

SUCCESS example:
{"status": "SUCCESS", "file_path": "reports/2026_06_10_12_00/PrimeBarControl.txt",
"commit_sha": "2598fb968b56153b2495e9c315c718cab9dd38ce", "report_timestamp_utc": "2026_06_10_12_00"}

FAILED example:
{"status": "FAILED", "file_path": "reports/2026_06_10_12_00/PrimeBarControl.txt",
"error": "GitHub PUT failed: HTTP 422 — ...", "report_timestamp_utc": "2026_06_10_12_00"}

Return JSON only.
```

**LLM Configuration**

| Setting | Value | Reason |
|---|---|---|
| AI Engine | Claude (Bedrock) | Consistent with pipeline |
| Model | Claude Sonnet | Sufficient for passthrough; Haiku may truncate |
| Temperature | Precise | Zero creative work — deterministic passthrough only |
| Top P | Narrow | Pairs with Precise for maximum determinism |
| Max Iteration | Single run | Tool handles failure; retry risks 422 on same path |
| Max RPM | Team testing | Upgrade to Production after orchestrator validated |
| Max Execution Time | Balanced | Tool has 30s internal timeout; needs overhead |

---

## 7. Tool Specification — XAMLTimestampedReportWriter

### 7.1 Registration

| Field | Value |
|---|---|
| Tool Name | XAMLTimestampedReportWriter |
| Tool Class | XAMLTimestampedReportWriter |
| Tool Description | Writes XAML automation reports into GitHub using timestamped folders. Creates one report file per execution. No append. No update. No deduplication. |

### 7.2 Input Schema

| Field | Type | Required | Description |
|---|---|---|---|
| `github_owner` | string | Yes | GitHub username or organisation |
| `github_repo` | string | Yes | Repository name |
| `github_token` | string | Yes | PAT with repo write access — staging only |
| `github_base_url` | string | No (default: `https://api.github.com`) | API base URL — set to GHE URL for enterprise |
| `branch` | string | Yes | Target branch (e.g. `main`) |
| `file_name` | string | Yes | XAML filename including `.xaml` extension |
| `report_content` | string | Yes | Full report text — Sections 1 through 4 |
| `run_timestamp_utc` | string | No | UTC timestamp `YYYY_MM_DD_HH_MM` — if absent, tool generates own |

### 7.3 Validation Rules

| Field | Rule |
|---|---|
| `github_owner`, `github_repo`, `branch` | Regex `^[A-Za-z0-9_.\-]+$` — no path traversal |
| `file_name` | Must end with `.xaml` (case-insensitive) |
| `github_base_url` | Must start with `https://` |
| `run_timestamp_utc` | Must match `^\d{4}_\d{2}_\d{2}_\d{2}_\d{2}$` (5 segments) OR `^\d{4}_\d{2}_\d{2}_\d{2}_\d{2}_\d{2}$` (6 segments — auto-truncated) |

### 7.4 Output Schema

**Success**
```json
{
  "status": "SUCCESS",
  "file_path": "reports/2026_06_11_05_40/PrimeBarControl.txt",
  "commit_sha": "4a7dad8668b0884d2b0419336954b4f0dee70f8a",
  "report_timestamp_utc": "2026_06_11_05_40"
}
```

**Failure**
```json
{
  "status": "FAILED",
  "file_path": "reports/2026_06_11_05_40/PrimeBarControl.txt",
  "error": "GitHub PUT failed. HTTP 422. {\"message\":\"sha wasn't supplied\"}",
  "report_timestamp_utc": "2026_06_11_05_40"
}
```

### 7.5 Full Tool Code (Staging — Pydantic v2)

```python
import requests
import base64
import json
from datetime import datetime, timezone
from typing import Optional, Type
from pydantic import BaseModel, Field, field_validator
from crewai.tools import BaseTool
import re


class XAMLTimestampedReportWriterSchema(BaseModel):
    """
    Pydantic v2 compatible schema.
    Staging version: github_token passed as input field.
    Production version: retrieve via AVASecret.getValue("GITHUB_TOKEN").
    """

    github_owner: str = Field(...)
    github_repo: str = Field(...)
    github_token: str = Field(...)
    github_base_url: str = Field(default="https://api.github.com")
    branch: str = Field(...)
    file_name: str = Field(...)
    report_content: str = Field(...)
    run_timestamp_utc: Optional[str] = Field(default=None)

    @field_validator("github_owner", "github_repo", "branch", mode="before")
    @classmethod
    def validate_github_fields(cls, v, info):
        field_name = info.field_name
        if not v or not str(v).strip():
            raise ValueError(f"{field_name} must not be empty.")
        if "../" in v or v.startswith("/") or v.startswith("\\"):
            raise ValueError(f"{field_name} contains invalid path traversal sequence.")
        if not re.match(r"^[A-Za-z0-9_.\-]+$", v):
            raise ValueError(f"{field_name} contains invalid characters.")
        return v

    @field_validator("file_name", mode="before")
    @classmethod
    def validate_file_name(cls, v):
        if not str(v).lower().endswith(".xaml"):
            raise ValueError("file_name must end with .xaml")
        return v

    @field_validator("github_base_url", mode="before")
    @classmethod
    def validate_base_url(cls, v):
        if not str(v).startswith("https://"):
            raise ValueError("github_base_url must start with https://")
        return str(v).rstrip("/")

    @field_validator("run_timestamp_utc", mode="before")
    @classmethod
    def validate_timestamp(cls, v):
        if v is not None:
            v = str(v).strip()
            # Accept YYYY_MM_DD_HH_MM (5 segments)
            if re.match(r"^\d{4}_\d{2}_\d{2}_\d{2}_\d{2}$", v):
                return v
            # Accept YYYY_MM_DD_HH_MM_SS (6 segments) — truncate seconds
            if re.match(r"^\d{4}_\d{2}_\d{2}_\d{2}_\d{2}_\d{2}$", v):
                return v[:16]
            raise ValueError(
                "run_timestamp_utc must be in YYYY_MM_DD_HH_MM format. "
                "Example: 2026_06_10_12_00"
            )
        return v


class XAMLTimestampedReportWriter(BaseTool):

    name: str = "XAMLTimestampedReportWriter"
    description: str = (
        "Writes XAML automation reports into GitHub using timestamped folders. "
        "Creates one report file per execution. "
        "No append. No update. No deduplication."
    )
    args_schema: Type[BaseModel] = XAMLTimestampedReportWriterSchema

    def _run(
        self,
        github_owner: str,
        github_repo: str,
        github_token: str,
        branch: str,
        file_name: str,
        report_content: str,
        github_base_url: str = "https://api.github.com",
        run_timestamp_utc: Optional[str] = None,
    ) -> str:

        timestamp = ""
        report_file_path = ""

        try:
            # Step 1: Timestamp
            if run_timestamp_utc:
                timestamp = run_timestamp_utc
            else:
                timestamp = datetime.now(timezone.utc).strftime("%Y_%m_%d_%H_%M")

            # Step 2: Filename conversion
            base_name = file_name[:-5]
            txt_file_name = f"{base_name}.txt"

            # Step 3: GitHub path
            report_file_path = f"reports/{timestamp}/{txt_file_name}"

            # Step 4: Encode
            encoded_content = base64.b64encode(
                report_content.encode("utf-8")
            ).decode("utf-8")

            # Step 5: Payload — no sha field (new file creation)
            payload = {
                "message": f"automated: xaml report {txt_file_name} {timestamp}",
                "content": encoded_content,
                "branch": branch
            }

            # Step 6: PUT
            url = (
                f"{github_base_url}"
                f"/repos/{github_owner}/{github_repo}"
                f"/contents/{report_file_path}"
            )
            headers = {
                "Authorization": f"Bearer {github_token}",
                "Accept": "application/vnd.github.v3+json",
                "Content-Type": "application/json"
            }
            response = requests.put(
                url, headers=headers, data=json.dumps(payload), timeout=30
            )

            # Step 7: Success
            if response.status_code in (200, 201):
                try:
                    commit_sha = response.json().get("commit", {}).get("sha")
                    return json.dumps({
                        "status": "SUCCESS",
                        "file_path": report_file_path,
                        "commit_sha": commit_sha,
                        "report_timestamp_utc": timestamp
                    })
                except Exception:
                    return json.dumps({
                        "status": "SUCCESS",
                        "file_path": report_file_path,
                        "commit_sha": None,
                        "report_timestamp_utc": timestamp,
                        "warning": "File created but commit SHA could not be extracted."
                    })

            # Step 8: Failure
            try:
                error_body = response.text[:500]
            except Exception:
                error_body = "(response body unavailable)"

            return json.dumps({
                "status": "FAILED",
                "file_path": report_file_path,
                "error": f"GitHub PUT failed. HTTP {response.status_code}. {error_body}",
                "report_timestamp_utc": timestamp
            })

        except requests.exceptions.Timeout:
            return json.dumps({
                "status": "FAILED",
                "file_path": report_file_path,
                "error": "GitHub request timed out after 30 seconds.",
                "report_timestamp_utc": timestamp
            })

        except requests.exceptions.ConnectionError:
            return json.dumps({
                "status": "FAILED",
                "file_path": report_file_path,
                "error": "GitHub connection error. Verify github_base_url and network access.",
                "report_timestamp_utc": timestamp
            })

        except Exception as e:
            return json.dumps({
                "status": "FAILED",
                "file_path": report_file_path,
                "error": str(e),
                "report_timestamp_utc": timestamp
            })
```

---

## 8. Inter-Agent Schema

### 8.1 Agent 3 → Agent 4 Handoff

Agent 3 produces this JSON. Agent 4 receives it as `{{input}}`.

```json
{
  "file_name": "PrimeBarControl.xaml",
  "report_content": "# SECTION 1: Generated Java POM Class\n\npackage com.hp.omen.automation.pages;\n..."
}
```

### 8.2 Agent 4 → Tool Handoff

Agent 4 constructs this payload from its input plus pipeline config:

```json
{
  "github_owner": "anthropologenie",
  "github_repo": "omen-reports",
  "github_token": "github_pat_...",
  "github_base_url": "https://api.github.com",
  "branch": "main",
  "file_name": "PrimeBarControl.xaml",
  "report_content": "# SECTION 1...",
  "run_timestamp_utc": "2026_06_11_05_40"
}
```

### 8.3 Report Content Structure (Sections 1–4)

```
# SECTION 1: Generated Java POM Class
  Java class extending ElementBase
  Methods using Constants.ACCESSIBILITY_ID locator strategy
  One method per element with AutomationId

# SECTION 2: Automation Gap Report
  Table of elements missing AutomationId
  Columns: element_type, x_name, section, purpose, binding_context, issue

# SECTION 3: Automation Priority Report
  HIGH:   Interactive elements (Button) missing AutomationId
  MEDIUM: Dynamic indicators (Image, ContentControl) missing AutomationId
  LOW:    Static/structural elements (Grid, Label) missing AutomationId

# SECTION 4: POM Validation & Consistency Review
  Validation table: POM Methods, Gap Report, Priority Classification, Framework Compliance
  Accuracy score formula and breakdown
  Warnings (e.g. duplicate AutomationId)
  Final verdict: PASS / PASS WITH WARNINGS / FAIL
```

---

## 9. GitHub Write Logic

### 9.1 Why No SHA Is Required

GitHub's Contents API requires a `sha` field only when **updating an existing file**. The timestamped folder design guarantees every PUT targets a path that has never existed — `reports/<new_timestamp>/<filename>.txt`. New file creation does not require a SHA. Omitting it is correct, not an oversight.

### 9.2 API Specification

| Parameter | Value |
|---|---|
| Method | `PUT` |
| Public GitHub endpoint | `https://api.github.com/repos/{owner}/{repo}/contents/{path}` |
| HP GHE endpoint | `https://github.azc.ext.hp.com/api/v3/repos/{owner}/{repo}/contents/{path}` |
| Auth header | `Authorization: Bearer {token}` (not `token` — deprecated) |
| Content-Type | `application/json` |
| Body: message | `automated: xaml report {filename}.txt {timestamp}` |
| Body: content | Base64-encoded UTF-8 report content |
| Body: branch | `main` |
| Body: sha | **Omit** — new file only |
| Timeout | 30 seconds |

### 9.3 422 Error Handling

A 422 response with `"sha" wasn't supplied` means the file already exists at that path. This happens when:
- The same `run_timestamp_utc` is reused across two runs
- Two runs occur within the same minute with no `run_timestamp_utc` provided

**Resolution:** pass a different timestamp or omit `run_timestamp_utc` to let the tool generate a fresh one.

---

## 10. Validated Run Evidence

### 10.1 Confirmed Commits

```
commit 4a7dad8668b0884d2b0419336954b4f0dee70f8a  ← Full report validation
Author: anthropologenie
Date:   Thu Jun 11 11:10:53 2026 +0530
    automated: xaml report PrimeBarControl.txt 2026_06_11_05_40

commit 2598fb968b56153b2495e9c315c718cab9dd38ce  ← First full report
Author: anthropologenie
Date:   Wed Jun 10 15:30:57 2026 +0530
    automated: xaml report PrimeBarControl.txt 2026_06_10_12_00

commit 94b7147f1e41c91a8b474680bc5ba2bb34fe5065  ← Hello World test
Author: anthropologenie
Date:   Wed Jun 10 15:27:28 2026 +0530
    automated: xaml report PrimeBarControl.txt 2026_06_10_12_00_00
```

### 10.2 Validated Files

| File | Run | Sections | POM Methods | Gap Elements | Result |
|---|---|---|---|---|---|
| `TooltipControl.xaml` | Early pipeline test | 1–4 | Partial | Present | ✅ Committed |
| `SignInV2.xaml` | Agent 3 validation | 2–4 only | 0 (no AutomationIds) | 7 | ✅ Gap path confirmed |
| `PrimeBarControl.xaml` | Full pipeline validation | 1–4 | 7 | 10 | ✅ Both paths confirmed |

### 10.3 PrimeBarControl POM — Validated Methods

```java
public WebElement tutorialButton()          // AutomationId: TutorialButton
public WebElement tutorialBarButton()       // AutomationId: TutorialBarButton
public WebElement notificationsBarButton()  // AutomationId: NotificationsBarButton
public WebElement settingsButton()          // AutomationId: SettingsButton
public WebElement settingsButtonImage()     // AutomationId: SettingsButton (duplicate — warning raised)
public WebElement playerMenuBarButton()     // AutomationId: PlayerMenuBarButton
public WebElement playerMenuStatus()        // AutomationId: PlayerMenuStatus
```

> **⚠ Warning:** `SettingsButton` AutomationId is shared by a Button and an Image element. POM correctly generates two distinct methods, but runtime ambiguity is possible. Dev team should assign unique AutomationIds.

---

## 11. Error Reference

| Error | HTTP Code | Root Cause | Resolution |
|---|---|---|---|
| `ERR-5000 — An unexpected error occurred` | 500 (AAVA) | Pydantic v1 `@validator` with `field` kwarg — dies at class load time in Pydantic v2 runtime | Replace `@validator` + `field` with `@field_validator` + `@classmethod` + `info` |
| `"sha" wasn't supplied` | 422 (GitHub) | File already exists at target path — same timestamp reused | Use different `run_timestamp_utc` or omit to auto-generate |
| `run_timestamp_utc must be in YYYY_MM_DD_HH_MM format` | Validation | 6-segment timestamp passed (includes seconds) | Validator now auto-truncates 6-segment to 5-segment |
| `GitHub PUT failed. HTTP 403` | 403 (GitHub) | PAT missing `repo` write scope or expired | Re-generate PAT with `repo` scope |
| `GitHub PUT failed. HTTP 404` | 404 (GitHub) | Wrong `github_base_url` — missing `/api/v3/` prefix on GHE | Set `github_base_url` to `https://github.azc.ext.hp.com/api/v3` |
| `GitHub request timed out` | — | GitHub API unresponsive | Retry with fresh `run_timestamp_utc` |

---

## 12. HP GitHub Enterprise Migration

### 12.1 What Changes

| Parameter | Staging (Public GitHub) | Production (HP GHE) |
|---|---|---|
| `github_base_url` | `https://api.github.com` | `https://github.azc.ext.hp.com/api/v3` |
| `github_owner` | `anthropologenie` | `PSSW` |
| `github_repo` | `omen-reports` | Confirm with HP team |
| `branch` | `main` | Confirm GHE default branch |
| `github_token` | Public GitHub PAT | HP Enterprise-scoped PAT — access request required |
| XAML source | `Vamsi2359/ContosoAir_Automation` | `PSSW/win-app-omen` |
| XAML branch | `master` | `develop` |

### 12.2 What Does Not Change

- All tool code
- Agent prompts
- Report format (Sections 1–4)
- Folder naming convention
- Pipeline ID (4510)
- Agent 3 logic

### 12.3 GHE-Specific Risks

| Risk | Detail | Mitigation |
|---|---|---|
| `/api/v3/` prefix | GHE requires this prefix; public GitHub omits it. Forgetting it returns 404 even with valid credentials. | Validate URL in agent config before first run |
| PAT scope | HP enterprise PATs require explicit provisioning — not self-service | Raise access request to HP IT before migration sprint |
| File path depth | `PrimeBarControl.xaml` is at `Source/Features/Services/Homepage/Prime/Module/Views/PrimeBarControl.xaml` — 7 levels deep | Git Trees API with `recursive=1` required for discovery; Contents API fetch uses full path string |

### 12.4 Migration Validation Steps

```
1. Provision HP GHE enterprise PAT with repo:read on PSSW/win-app-omen
                                         repo:write on PSSW/omen-reports
2. Update github_base_url to https://github.azc.ext.hp.com/api/v3
3. Update github_owner to PSSW
4. Run tool playground test with file_name=PrimeBarControl.xaml, report_content="hello world"
5. Confirm 201 response and commit in PSSW/omen-reports
6. Run full pipeline on PrimeBarControl.xaml from win-app-omen
7. Confirm output matches public GitHub validation run exactly
```

---

## 13. Key Learnings and Architectural Decisions

### 13.1 Pydantic v2 Compatibility — `field` → `info`

The `ERR-5000` error on AAVA was caused by Pydantic v1 `@validator` syntax with the `field` keyword argument. AAVA's runtime uses Pydantic v2 where `field` is not available — use `info` instead. This exception fires at class definition time, making every tool execution fail before any code runs.

```python
# WRONG — Pydantic v1
@validator("github_owner")
def validate(cls, v, field):
    raise ValueError(f"{field.name} is invalid")

# CORRECT — Pydantic v2
@field_validator("github_owner", mode="before")
@classmethod
def validate(cls, v, info):
    raise ValueError(f"{info.field_name} is invalid")
```

### 13.2 SHA Discipline

GitHub Contents API PUT requires `sha` when **updating** an existing file. For **new** files, `sha` must be **omitted**. Including it on a new path causes a 422. The timestamped folder design eliminates SHA management entirely — every path is always new.

### 13.3 Bearer vs Token Auth Header

`Authorization: token {pat}` is deprecated since 2021. HP GHE 3.x may reject it. Always use `Authorization: Bearer {pat}`.

### 13.4 Timestamp Format — 5 Segments Not 6

`YYYY_MM_DD_HH_MM` (no seconds). The validator accepts 6-segment input and auto-truncates to maintain compatibility when callers include seconds, but the canonical format is 5 segments. The Expected Output field in the AAVA agent prompt must show 5-segment examples — showing 6-segment examples trains the LLM to generate the wrong format.

### 13.5 No Knowledge Base on Passthrough Agents

Agent 4 has zero creative work to do. Attaching a knowledge base gives the LLM material to reason from, increasing the chance it modifies `report_content` or adds prose around the JSON response. Passthrough agents should always have an empty knowledge base.

---

## 14. Open Items and Roadmap

### 14.1 Immediate Actions

| Item | Owner | Priority |
|---|---|---|
| Register XAMLTimestampedReportWriter as Agent 4 in pipeline 4510 | Karthik / AAVA team | P1 |
| Wire Agent 3 → Agent 4 handoff in live pipeline | Karthik / AAVA team | P1 |
| Run full Agent 3 → Agent 4 live execution on AAVA | AAVA team | P1 |
| Confirm `reports/` folder exists in omen-reports repo | AAVA team | P1 |

### 14.2 Sprint 2

| Item | Owner | Priority |
|---|---|---|
| Provision HP GHE enterprise PAT (PSSW/omen-reports write, PSSW/win-app-omen read) | HP IT / Vamsi | P1 |
| Confirm production omen-reports repo name on HP GHE | Chandan / HP team | P1 |
| First GHE run: PrimeBarControl.xaml on win-app-omen | AAVA team | P1 |
| Build Git Trees discovery agent for win-app-omen XAML inventory | AAVA team | P2 |
| Wire discovery agent as Agent 1 in pipeline 4510 | AAVA team | P2 |
| 10-file batch test on GHE | AAVA team | P2 |

### 14.3 Sprint 3

| Item | Owner | Priority |
|---|---|---|
| External orchestration loop (FastAPI wrapper — no AAVA REST API) | AAVA team | P1 |
| Sequential queue + polling across all XAML files | AAVA team | P1 |
| File-level exception isolation | AAVA team | P2 |
| Scale test: 50+ files from win-app-omen | AAVA team | P2 |
| Production migration: AVASecret for github_token (remove from schema) | AAVA team | P2 |
| Deduplication explicit test — re-run same file same timestamp | AAVA team | P3 |

### 14.4 Out of Scope (This Pipeline)

The following capabilities are explicitly excluded from the current pipeline scope and belong to future phases:

- RCA generation
- Repository contextualization
- Knowledge Base retrieval and vector search
- Knowledge Graph integration
- Aggregated analytics dashboard (XAMLReportWriter handles this separately)

---

*End of Document — PSSW OGH POM Generator Pipeline v1.0*