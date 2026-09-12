# Instruction Architecture: CLAUDE.md vs Skills vs Hooks

## Executive Summary

This project demonstrates three distinct configuration layers, each serving a different purpose:

| Layer | Purpose | When to Use | Example |
|-------|---------|------------|---------|
| **CLAUDE.md** | Project context & shared understanding | Codebase info, team guidelines, decision log | "This repo enforces AUD tickets for legacy changes" |
| **Skills** | Domain-specific review frameworks | Activate on request for guided analysis | `/payment-risk-review` for fee calculations |
| **Hooks** | Automated enforcement gates | Pre/post-tool validation & logging | Block legacy edits without ticket, log PII mentions |

---

## Detailed Breakdown

### 1. CLAUDE.md (Project Documentation)
**Purpose**: Document codebase architecture, decision history, and team norms.

**Belongs here**:
- Architecture overview: "Project One is a payments API, Project Two handles claims"
- Team conventions: "All refunds require PR review by payment-team"
- Decision log: "Why we chose settling T+1 for India vs T+2 globally"
- Constraints: "This is regulated financial code; read payment-risk-review before editing fees"
- Setup instructions: "pytest requires Python 3.10+"

**Does NOT belong here**:
- Automated enforcement (use hooks instead)
- Domain analysis frameworks (use skills instead)
- Rules that need to block tool execution (use hooks)

**Example content**:
```markdown
# Payment Processing API

## Architecture
- src/fees.py: Fee calculation (1.10% + Rs 2.50 flat)
- src/settlement/: T+1 India, T+2 elsewhere
- src/legacy/: FROZEN. Changes require AUD ticket.

## Regulated Compliance
All fee changes must be reviewed via /payment-risk-review.
Decimal rounding is audit-critical; see test_fees.py for boundary cases.
```

---

### 2. Skills (Review & Analysis Frameworks)
**Purpose**: Provide guided analysis patterns that activate on request or via slash commands.

**Belongs here**:
- Domain-specific review focus: "Check decimal rounding for payments"
- Analysis checklist: "PII: no raw emails/phones in logs"
- Files to inspect first: "Audit src/fees.py, src/settlement/rules.py, tests/"
- Boundary test guidance: "Test rounding on 99, 100, 123.45 amounts"
- When to use: "Use this for any payment/fee code"

**Does NOT belong here**:
- Tool execution gates (use hooks)
- Automatic file-write warnings (use hooks)
- Blocking behavior (use hooks)

**Three-tier skill structure**:

1. **Root-level skills** (future): Global, apply to entire workspace
   - Example: "code-security-review" for all projects

2. **Project-level skills**: `/payment-risk-review`, `/claim-schema-check`
   - Location: `project-two-claims-api/.claude/skills/`
   - Applies to: whole project
   - **Project One has no root-level skills** (payments are in files, not reviewed via skill yet)
   - **Project Two has two skills**: payment-risk-review, claim-schema-check

3. **Nested skills**: `settlement-guard` under `src/settlement/.claude/skills/`
   - Location: `project-one-payments-api/src/settlement/.claude/skills/settlement-guard/`
   - Applies to: files under `src/settlement/` only
   - Purpose: Settlement-specific safety checks (idempotency, approval thresholds)
   - **New in Project Two**: `pii-guard` under `src/.claude/skills/pii-guard/`
     - Applies to: files under `src/` (project-two-claims-api/src/**)
     - Purpose: Check claim processing code for PII logging risks

---

### 3. Hooks (Automated Enforcement & Logging)
**Purpose**: Automatically validate or log tool usage at specific lifecycle points.

**Belongs here**:
- PreToolUse gates: Block dangerous edits (e.g., legacy without ticket)
- PostToolUse logging: Record every file edit for audit trails
- File-write warnings: Alert on PII keywords (email, phone)
- Environment validation: "Check Python version before running tests"

**Does NOT belong here**:
- Analysis guidance (use skills)
- Codebase documentation (use CLAUDE.md)
- Optional review steps (use skills)

**Hook lifecycle**:

1. **PreToolUse** (fires BEFORE tool executes):
   - `block_legacy_without_ticket.py` in Project One
   - Denies Edit/Write to `src/legacy/*` unless text contains `AUD-\d+` ticket
   - Blocks at tool invocation time; user sees permission error

2. **PostToolUse** (fires AFTER tool succeeds):
   - `audit_file_change.py` in Project One
     - Logs every Edit/Write to `.claude/audit/file_changes.jsonl`
     - Records: timestamp, tool name, file path, action (modified/created)
   - `pii_warning.py` in Project Two
     - Scans file edits for "email" or "phone" keywords
     - Writes warning to `.claude/warnings/pii_warnings.log`
     - Alerts on stderr: `⚠️  PII Warning: ...`

---

## Current Configuration Summary

### Project One (project-one-payments-api)
```
.claude/
  settings.json
    PreToolUse: block_legacy_without_ticket.py
      → Blocks src/legacy/ edits without AUD-* ticket
    PostToolUse: audit_file_change.py
      → Logs all edits to .claude/audit/file_changes.jsonl
  hooks/
    block_legacy_without_ticket.py
    audit_file_change.py
  skills/
    (none at root)
src/settlement/.claude/skills/
  settlement-guard/SKILL.md
    → Nested skill: validates settlement idempotency, thresholds
```

### Project Two (project-two-claims-api)
```
.claude/
  settings.json
    PostToolUse: pii_warning.py
      → Logs PII keyword warnings to .claude/warnings/pii_warnings.log
  hooks/
    pii_warning.py
  skills/
    payment-risk-review/  → Project-level skill
    claim-schema-check/   → Project-level skill
src/.claude/skills/
  pii-guard/SKILL.md
    → Nested skill: validates no raw emails/phones/policy numbers in logs
```

---

## Decision Tree: Where Does My Instruction Go?

```
Is it a gate (must block tool execution)?
├─ YES → Hook (PreToolUse or PostToolUse)
│  └─ Example: "Block edits to src/legacy without AUD ticket"
│
└─ NO: Is it a guided analysis pattern (activate on request)?
   ├─ YES → Skill
   │  └─ Example: "/payment-risk-review checks fee rounding"
   │
   └─ NO: Is it project context or decision history?
      └─ YES → CLAUDE.md
         └─ Example: "Why we settled on T+1 for India"
```

---

## Implementation Examples

### Example 1: "Prevent logging raw phone numbers"
- **Hook (PreToolUse)**: ✗ No—users should be able to write the code
- **Hook (PostToolUse)**: ✓ YES—warn after editing if "phone" keyword appears
- **Skill**: ✓ YES—guide analysis of claim handlers for PII risks
- **CLAUDE.md**: ✓ YES—"All phone numbers must be masked before logging per GDPR"

### Example 2: "Fees must use 1.10%, not 11%"
- **Hook**: ✗ No—cannot be enforced at tool time
- **Skill**: ✓ YES—`/payment-risk-review` checks fee calculations
- **Tests**: ✓ YES—`test_fees.py::test_fee_on_1000` catches the bug
- **CLAUDE.md**: ✓ YES—"Fee spec: 1.10% + Rs 2.50 flat per UniPay contract"

### Example 3: "Ticket mandatory for legacy changes"
- **Hook (PreToolUse)**: ✓ YES—`block_legacy_without_ticket.py` denies without AUD-*
- **CLAUDE.md**: ✓ YES—"All src/legacy changes are frozen; require AUD ticket"
- **Skill**: ✗ No—enforcement is non-negotiable

---

## Summary Table

| Scenario | CLAUDE.md | Skill | Hook |
|----------|-----------|-------|------|
| "Document why we chose T+1 settlement" | ✓ | ✗ | ✗ |
| "Check fees for decimal rounding" | ✗ | ✓ | ✗ |
| "Block legacy edits without ticket" | ✗ | ✗ | ✓ |
| "Warn on PII keywords in edits" | ✗ | ✗ | ✓ |
| "Audit every file change" | ✗ | ✗ | ✓ |
| "Guide PII review of claim code" | ✗ | ✓ | ✗ |
| "Team uses X linter config" | ✓ | ✗ | ✗ |
| "Enforce X linter before commit" | ✗ | ✗ | ✓ |
