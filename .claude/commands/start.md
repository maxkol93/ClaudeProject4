# /start — Project Bootstrap

Bootstrap TelePub from SPARC documentation. Run at project start or after onboarding a new developer.

## What this does

1. Reads all docs in `docs/` to understand the full project
2. Checks current state (what's built vs what's planned)
3. Recommends the next action based on PRD feature roadmap

## Usage

```
/start
/start --focus=payments     # bootstrap with focus on payment subsystem
/start --feature=US-01      # bootstrap context for a specific user story
```

## Steps

### 1. Read project context
Read these files in order:
- `CLAUDE.md` — project overview and architecture
- `docs/PRD.md` — product requirements and user stories
- `docs/Architecture.md` — technical architecture and data model
- `docs/Pseudocode.md` — core algorithms
- `docs/Refinement.md` — edge cases and test strategy

### 2. Check current state
```bash
git log --oneline -20
git status
find . -name "*.py" | grep -v __pycache__ | head -30
ls bot/ api/ worker/ 2>/dev/null || echo "No source directories yet"
```

### 3. Identify what's built
Map existing code to PRD features:
- US-01 (Author Registration): bot/handlers/registration.py
- US-02 (Subscription): bot/handlers/subscription.py + worker/tasks/access.py
- US-03 (Access Management): worker/tasks/lifecycle.py
- US-04 (Payouts): worker/tasks/payouts.py
- US-05 (Analytics): api/routers/analytics.py

### 4. Recommend next step

Output a structured report:

```
## TelePub — Project Status

**Phase:** [MVP / v1.0 / v2.0]

### Built ✅
- [list features that have code]

### In Progress 🔄
- [features with partial code]

### Next: [recommended feature]
- Story: US-XX
- Why: [reason based on dependencies]
- Start with: /feature [feature-name]
```

## First-time setup

If no source code exists yet:

```bash
# Create directory structure
mkdir -p bot/{handlers,middlewares,keyboards,states} \
         api/{routers,dependencies,schemas} \
         worker/{tasks,beat} \
         shared/{models,database,config} \
         tests/{unit,integration,e2e} \
         alembic/versions \
         nginx \
         monitoring/{prometheus,grafana}

# Create __init__.py files
find bot api worker shared -type d -exec touch {}/__init__.py \;

echo "Directory structure created. Run /feature registration to start US-01."
```

## Context hints

After /start, you have full project context. Typical next commands:
- `/feature registration` — implement US-01 (Author Registration)
- `/plan payments` — plan the ЮKassa integration
- `/test unit` — generate unit tests for existing code
