# Claude Code Configuration - Multi-Project Architecture

A comprehensive demonstration of Claude Code's hook and skill system across multiple projects with compliance and security frameworks.

## Overview

This repository contains two interconnected projects showing regulated code management, automated enforcement, and domain-specific review patterns using Claude Code's configuration system.

**Key Features:**
- Multi-project architecture with different compliance requirements
- PreToolUse hooks for access control (legacy code protection)
- PostToolUse hooks for audit logging and PII warnings
- Project-level and nested skills for guided analysis
- Decimal precision validation for financial calculations
- PII prevention in claim processing

## Repository Structure

```
cca-claudecode-config/
├── README.md                                    [This file]
├── EXPLANATION.md                              [Architecture deep-dive]
├── .git/                                        [Version control]
│
├── project-one-payments-api/                   [Payments Processing]
│   ├── .claude/
│   │   ├── settings.json                       [Hook configuration]
│   │   └── hooks/
│   │       ├── block_legacy_without_ticket.py  [PreToolUse - enforce tickets]
│   │       └── audit_file_change.py            [PostToolUse - compliance log]
│   │
│   ├── src/
│   │   ├── fees.py                             [Fee calculations - FIXED]
│   │   ├── legacy/                             [FROZEN - requires AUD ticket]
│   │   │   └── settlement_core.py
│   │   └── settlement/
│   │       ├── rules.py                        [Settlement governance]
│   │       └── .claude/skills/
│   │           └── settlement-guard/
│   │               └── SKILL.md                [Nested skill]
│   │
│   └── tests/
│       └── test_fees.py                        [Fee tests - 6 passing]
│
└── project-two-claims-api/                     [Claims Processing]
    ├── .claude/
    │   ├── settings.json                       [PII warning hook config]
    │   ├── hooks/
    │   │   └── pii_warning.py                  [PostToolUse - PII alerts]
    │   └── skills/
    │       ├── payment-risk-review/
    │       │   └── SKILL.md                    [Project-level skill]
    │       └── claim-schema-check/
    │           └── SKILL.md                    [Project-level skill]
    │
    └── src/
        └── .claude/skills/
            └── pii-guard/
                └── SKILL.md                    [Nested skill - PII validation]
```

## Projects

### Project One: Payments API (`project-one-payments-api`)

**Purpose:** Process financial transactions with strict compliance requirements.

**Key Components:**

#### Code Changes
- **src/fees.py**: Fixed interchange fee calculation
  - ✓ Changed 0.11 (11%) → 0.011 (1.10%)
  - ✓ Spec: "1.10% of amount + Rs 2.50 flat, rounded to 2 decimals"

- **tests/test_fees.py**: Enhanced boundary value testing
  - ✓ 6 total tests (2 original + 4 new)
  - ✓ Tests: 1000, 250, 100, 99, 0, 123.45 amounts
  - ✓ All passing (verified with pytest)

- **src/legacy/settlement_core.py**: Modified with compliance ticket
  - Changed return value: "settled" → "done"
  - Requires AUD-999 ticket reference (enforced by hook)

#### Hooks

**PreToolUse: `block_legacy_without_ticket.py`**
- **When:** Before Edit/Write/MultiEdit tool execution
- **Rule:** Files in `src/legacy/*` require AUD-#### ticket reference
- **Action:** 
  - Scans for regex pattern: `AUD-\d+`
  - ALLOW if found
  - DENY if not found with error message
- **Example:** Attempted edit to `src/legacy/settlement_core.py` without ticket → BLOCKED

**PostToolUse: `audit_file_change.py`**
- **When:** After successful Edit/Write/MultiEdit
- **Output:** `.claude/audit/file_changes.jsonl`
- **Records:** timestamp, tool name, file path, action (modified/created)
- **Use Case:** Compliance audit trail for regulated code

#### Skills

**Nested Skill: `settlement-guard`**
- **Location:** `src/settlement/.claude/skills/settlement-guard/`
- **Scope:** Activates for files under `src/settlement/**`
- **Focus:** 
  - Settlement window validation (T+1 for India, T+2 elsewhere)
  - Idempotency checks
  - Approval thresholds
  - Duplicate payment prevention

### Project Two: Claims API (`project-two-claims-api`)

**Purpose:** Process insurance claims with PII protection requirements.

**Key Components:**

#### Hooks

**PostToolUse: `pii_warning.py`**
- **When:** After successful Edit/Write/MultiEdit
- **Scans:** File edits for "email" or "phone" keywords (case-insensitive)
- **Output:** `.claude/warnings/pii_warnings.log`
- **Action:** 
  1. Log JSON entry: timestamp, file_path, PII types, message
  2. Print warning to stderr: `⚠️  PII Warning: ...`
- **Effect:** ALERTS (non-blocking) — raises developer awareness
- **Example Entry:**
  ```json
  {
    "timestamp": "2026-09-12T06:30:00.000Z",
    "file_path": "src/handlers/email_processor.py",
    "pii_types_detected": ["email"],
    "message": "Warning: Edit contains email references. Ensure no raw PII is logged."
  }
  ```

#### Skills

**Project-Level Skills:**
1. **payment-risk-review** — Invoke via `/payment-risk-review`
   - Focus: Fee calculations, decimal rounding, idempotency
   - Files: `src/fees.py`, `src/settlement/rules.py`, `tests/`

2. **claim-schema-check** — Invoke via `/claim-schema-check`
   - Focus: Claim data schema validation

**Nested Skill: `pii-guard`**
- **Location:** `src/.claude/skills/pii-guard/`
- **Scope:** Activates for files under `src/**`
- **Focus:**
  - No raw email addresses in logs
  - No raw phone numbers in logs
  - No raw policy numbers in logs
  - Proper masking/hashing before logging
  - Audit trail compliance
- **Pattern Checks:**
  - `logger.*email` or `print.*email`
  - `logger.*phone` or `print.*phone`
  - `logger.*policy` or `print.*policy`

## Hook System

### Hook Lifecycle

**PreToolUse (Before execution):**
- Fires before Edit/Write/MultiEdit tools run
- Can BLOCK tool execution
- Use case: Access control, validation gates
- Example: `block_legacy_without_ticket.py` denies risky edits

**PostToolUse (After execution):**
- Fires after Edit/Write/MultiEdit tools succeed
- Cannot block (tool already executed)
- Use case: Logging, alerting, audit trails
- Examples: 
  - `audit_file_change.py` logs to compliance file
  - `pii_warning.py` alerts on PII keywords

### Configuration

**Project One:** `.claude/settings.json`
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python ${CLAUDE_PROJECT_DIR}/.claude/hooks/block_legacy_without_ticket.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python ${CLAUDE_PROJECT_DIR}/.claude/hooks/audit_file_change.py"
          }
        ]
      }
    ]
  }
}
```

**Project Two:** `.claude/settings.json`
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python ${CLAUDE_PROJECT_DIR}/.claude/hooks/pii_warning.py"
          }
        ]
      }
    ]
  }
}
```

## Skill System

### Three-Tier Architecture

1. **Global Skills** (future)
   - Apply workspace-wide to all projects
   - Example: "code-security-review"

2. **Project-Level Skills**
   - Defined at project root: `.claude/skills/`
   - Example: `/payment-risk-review`, `/claim-schema-check`

3. **Nested Skills** (directory-scoped)
   - Defined in subdirectories: `src/**/.claude/skills/`
   - Auto-activate when editing files in that scope
   - Example: `settlement-guard`, `pii-guard`

### Usage

**Project-Level Skill:**
```bash
# In project-two-claims-api
/payment-risk-review
```

**Nested Skill:**
- Auto-activated when editing files under the skill's scope
- Example: Editing `src/settlement/rules.py` → `settlement-guard` auto-suggested

## Configuration Architecture

See **EXPLANATION.md** for comprehensive guide on:
- When to use CLAUDE.md vs Skills vs Hooks
- Decision tree for configuration choices
- Real-world examples for each category

**Quick Reference:**
| Layer | Purpose | When to Use |
|-------|---------|------------|
| **CLAUDE.md** | Documentation | Project context, decision history, team norms |
| **Skills** | Review frameworks | Domain-specific analysis (request-activated) |
| **Hooks** | Enforcement | Automated validation & logging (event-triggered) |

## Testing

### Run Tests

```bash
cd project-one-payments-api
pip install pytest
pytest tests/test_fees.py -v
```

**Expected Output:**
```
tests/test_fees.py::test_fee_on_1000 PASSED               [ 16%]
tests/test_fees.py::test_fee_on_250 PASSED                [ 33%]
tests/test_fees.py::test_fee_rounding_down PASSED         [ 50%]
tests/test_fees.py::test_fee_rounding_edge_case PASSED    [ 66%]
tests/test_fees.py::test_fee_zero PASSED                  [ 83%]
tests/test_fees.py::test_fee_decimal_amount PASSED        [100%]

6 passed in 0.03s
```

### Test Coverage

**Boundary Value Tests:**
- ✓ Standard amount (1000)
- ✓ Medium amount (250)
- ✓ Round amounts (100)
- ✓ Edge case rounding (99 → 3.59)
- ✓ Zero amount edge case
- ✓ Decimal amounts (123.45 → 3.86)

## Compliance & Security

### Financial Compliance (Project One)
- ✓ Decimal precision validation (rounded to 2 decimals)
- ✓ Audit trail for all code changes
- ✓ Legacy code frozen (requires compliance tickets)
- ✓ Boundary value testing for rounding correctness

### PII Protection (Project Two)
- ✓ Email/phone keyword warnings on edits
- ✓ Audit log for all PII mentions
- ✓ Nested skill guides policy number handling
- ✓ Masking/hashing best practices in skill

## Directory Permissions

### Project One (Legacy Protection)
- ✓ `src/legacy/` — FROZEN (requires AUD-#### ticket)
- ✓ `src/settlement/` — Guided by `settlement-guard` skill
- ✓ `src/fees.py` — Critical for payment calculations
- ✓ `tests/` — Boundary cases validated

### Project Two (PII Protection)
- ✓ `src/` — Monitored for PII keywords via hook
- ✓ All edits generate PII warning log entries
- ✓ `src/` nested skill guides PII handling

## Contributing

### When Adding Code

1. **Project One (Payments):**
   - Run `/payment-risk-review` before editing fees
   - Legacy code requires AUD ticket reference
   - All changes logged to audit file

2. **Project Two (Claims):**
   - Run `/pii-guard` before editing claim processors
   - PII keywords trigger warnings
   - Ensure no raw emails/phones in logs

### Making Changes

```bash
# Edit a file (hooks activate automatically)
# PreToolUse: Validates against enforcement rules
# PostToolUse: Logs the change for audit trail

git add .
git commit -m "Your message"
git push origin master
```

## Verification Commands

```bash
# Check hook configuration
cat project-one-payments-api/.claude/settings.json
cat project-two-claims-api/.claude/settings.json

# View nested skills
find . -path "*/.claude/skills/*/SKILL.md"

# Run payment tests
cd project-one-payments-api
pytest tests/test_fees.py -v

# Check audit logs (after edits)
cat project-one-payments-api/.claude/audit/file_changes.jsonl
cat project-two-claims-api/.claude/warnings/pii_warnings.log
```

## Resources

- **EXPLANATION.md** — Architecture deep-dive and decision framework
- **Hook Scripts:** `.claude/hooks/*.py`
- **Skill Guides:** `.claude/skills/*/SKILL.md` and `src/**/.claude/skills/*/SKILL.md`
- **Tests:** `project-one-payments-api/tests/`

## License

This is a demonstration project for Claude Code configuration patterns.

## Support

For questions about:
- **Hook system** — See `.claude/hooks/` and `settings.json`
- **Skills architecture** — See `EXPLANATION.md`
- **Fee calculations** — See `project-one-payments-api/src/fees.py` and tests
- **PII handling** — See `project-two-claims-api/src/.claude/skills/pii-guard/SKILL.md`
