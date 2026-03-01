---
name: humanize-rlcr
description: Start RLCR (Ralph-Loop with Codex Review) iterative development loop. Implements plans with continuous Codex review until completion and code quality approval.
type: flow
---

# Humanize RLCR Loop

Starts an iterative development loop where:
1. AI implements your plan
2. AI writes a work summary  
3. Codex reviews the summary
4. Loop continues until COMPLETE, then code review phase begins
5. Code review checks quality with `[P0-9]` markers
6. Loop ends when no issues found

## Flow Diagram

```mermaid
flowchart TD
    BEGIN([BEGIN]) --> CHECK_ARGS{User provided<br/>plan file path?}
    CHECK_ARGS -->|No| ASK_PLAN[Ask user for plan file path<br/>or --skip-impl option]
    ASK_PLAN --> CHECK_ARGS
    CHECK_ARGS -->|Yes| SETUP[SETUP: Run setup script with args]
    SETUP --> CHECK_SETUP{Setup successful?}
    CHECK_SETUP -->|No| REPORT_ERROR[Report setup error]
    REPORT_ERROR --> END_FAIL([END])
    CHECK_SETUP -->|Yes| ROUND_0[ROUND 0: Init Goal Tracker]
    ROUND_0 --> ROUND_N[ROUND N: Implement & Write Summary]
    ROUND_N --> RUN_CODEX[CODEX_REVIEW: Run codex review script]
    RUN_CODEX --> CHECK_RESULT{Review result?}
    CHECK_RESULT -->|CONTINUE| SHOW_FEEDBACK[Show feedback & continue]
    SHOW_FEEDBACK --> ROUND_N
    CHECK_RESULT -->|COMPLETE| CODE_REVIEW[CODE_REVIEW: Run codex code review]
    CODE_REVIEW --> CHECK_ISSUES{Has [P0-9] issues?}
    CHECK_ISSUES -->|Yes| FIX_ISSUES[Fix issues & continue]
    FIX_ISSUES --> ROUND_N
    CHECK_ISSUES -->|No| FINALIZE[FINALIZE: Complete loop]
    FINALIZE --> END_SUCCESS([END])
```

## Flow Node Instructions

### SETUP: Run setup script with args

Execute the setup script:

```bash
/home/zyy/projects/humanize/scripts/setup-rlcr-loop.sh $ARGUMENTS
```

Capture the output. If exit code != 0, report error and end.

### ROUND 0: Init Goal Tracker

1. Read the plan file to understand Ultimate Goal and Acceptance Criteria
2. Create `.humanize/rlcr/<timestamp>/goal-tracker.md` with:
   - IMMUTABLE section: Ultimate Goal, Acceptance Criteria
   - MUTABLE section: Active Tasks, Completed Items, Deferred Items, Plan Evolution Log

### ROUND N: Implement & Write Summary

1. Work on implementation according to plan
2. Update goal-tracker.md with progress
3. Write work summary to `.humanize/rlcr/<timestamp>/summary-<round>.md`

### CODEX_REVIEW: Run codex review script

**This is the critical step - ACTUALLY RUN THE SCRIPT:**

The original plugin uses a template-based approach. Create a review prompt and pipe it to codex:

```bash
# Load proxy settings first
source ~/.zprofile

# Setup paths (adjust LOOP_DIR and CACHE_DIR based on actual paths)
LOOP_DIR=".humanize/rlcr/<timestamp>"
CACHE_DIR="~/.cache/humanize/<project-path>/<timestamp>"
ROUND="<N>"

mkdir -p "$LOOP_DIR" "$CACHE_DIR"

# Create the review prompt
REVIEW_PROMPT_FILE="$LOOP_DIR/round-${ROUND}-review-prompt.md"
cat > "$REVIEW_PROMPT_FILE" << 'EOF'
Please review this work summary against the goal tracker.

Output COMPLETE if all acceptance criteria are met.
Otherwise, provide specific feedback on what needs to be done.
EOF

# Output files
CODEX_STDOUT_FILE="$CACHE_DIR/round-${ROUND}-codex-stdout.md"
CODEX_LOG_FILE="$CACHE_DIR/round-${ROUND}-codex.log"
REVIEW_RESULT_FILE="$LOOP_DIR/round-${ROUND}-review-result.md"

# Run codex exec with output redirection
# Format: -m MODEL -c model_reasoning_effort=EFFORT --full-auto -C DIR
# IMPORTANT: Use timeout 5400 seconds (90 minutes) - DO NOT use short timeouts
cat "$REVIEW_PROMPT_FILE" | timeout 5400 codex exec \
  -m gpt-5.3-codex \
  -c model_reasoning_effort=xhigh \
  --full-auto \
  -C "$PWD" \
  - > "$CODEX_STDOUT_FILE" 2> "$CODEX_LOG_FILE"

# Copy stdout to review result for consistency
cp "$CODEX_STDOUT_FILE" "$REVIEW_RESULT_FILE"
```

**Output file locations:**
- **Codex stdout**: `~/.cache/humanize/<project-path>/<timestamp>/round-<N>-codex-stdout.md`
- **Codex stderr/log**: `~/.cache/humanize/<project-path>/<timestamp>/round-<N>-codex.log`
- **Review prompt**: `.humanize/rlcr/<timestamp>/round-<N>-review-prompt.md`
- **Review result**: `.humanize/rlcr/<timestamp>/round-<N>-review-result.md`

Check the review result file for:
- `COMPLETE` → proceed to CODE_REVIEW
- Anything else → return feedback to user, continue to next round

### CODE_REVIEW: Run codex code review

**ACTUALLY RUN THE CODE REVIEW:**

```bash
# Load proxy settings first
source ~/.zprofile

# Setup paths
LOOP_DIR=".humanize/rlcr/<timestamp>"
CACHE_DIR="~/.cache/humanize/<project-path>/<timestamp>"
ROUND="<N>"
BASE_BRANCH="<base-branch>"  # Auto-detect or from --base-branch

mkdir -p "$LOOP_DIR" "$CACHE_DIR"

# Output files
CODEX_REVIEW_LOG_FILE="$CACHE_DIR/round-${ROUND}-codex-review.log"
REVIEW_PROMPT_FILE="$LOOP_DIR/round-${ROUND}-review-prompt.md"
REVIEW_RESULT_FILE="$LOOP_DIR/round-${ROUND}-review-result.md"

# Create audit prompt file (codex review doesn't accept prompts, but we create this for audit)
cat > "$REVIEW_PROMPT_FILE" << EOF
# Code Review Phase - Round ${ROUND}

This file documents the code review invocation for audit purposes.
Note: codex review does not accept prompt input; it performs automated code review based on git diff.

## Review Configuration
- Base Branch: ${BASE_BRANCH}
- Review Round: ${ROUND}
EOF

# Run codex review with output redirection
# IMPORTANT: Use timeout 5400 seconds (90 minutes) - DO NOT use short timeouts
# Note: codex review outputs to stderr, so we redirect both stdout and stderr to the log file
timeout 5400 codex review --base "$BASE_BRANCH" \
  -c model=gpt-5.3-codex \
  -c review_model=gpt-5.3-codex \
  -c model_reasoning_effort=xhigh \
  > "$CODEX_REVIEW_LOG_FILE" 2>&1

# Copy log to review result
cp "$CODEX_REVIEW_LOG_FILE" "$REVIEW_RESULT_FILE"
```

**Output file locations:**
- **Codex stdout/log**: `~/.cache/humanize/<project-path>/<timestamp>/round-<N>-codex-review.log`
- **Review prompt (audit)**: `.humanize/rlcr/<timestamp>/round-<N>-review-prompt.md`
- **Review result**: `.humanize/rlcr/<timestamp>/round-<N>-review-result.md`

Check the review result file for `[P0-9]` markers.

### FINALIZE: Complete loop

1. Create finalize documentation
2. Rename state file to complete
3. Report success

## Decision Points

At **CHECK_RESULT**: Parse Codex output
- If contains `COMPLETE` → go to CODE_REVIEW
- Else → go to SHOW_FEEDBACK

At **CHECK_ISSUES**: Parse review output
- If contains `[P0-9]` → go to FIX_ISSUES  
- Else → go to FINALIZE

## Command Options

| Option | Description | Default |
|--------|-------------|---------|
| `path/to/plan.md` | Plan file path | Required (unless --skip-impl) |
| `--plan-file <path>` | Explicit plan file path | - |
| `--max N` | Maximum iterations | 42 |
| `--codex-model MODEL:EFFORT` | Codex model | gpt-5.3-codex:xhigh |
| `--codex-timeout SECONDS` | Review timeout | 5400 |
| `--base-branch BRANCH` | Base for code review | auto-detect |
| `--skip-impl` | Skip to code review | false |

## Usage

```bash
# Start with plan file
/flow:humanize-rlcr path/to/plan.md

# Skip implementation, review-only mode  
/flow:humanize-rlcr --skip-impl

# Load skill without auto-execution
/skill:humanize-rlcr
```

## Canceling

```bash
/home/zyy/projects/humanize/scripts/cancel-rlcr-loop.sh
```
