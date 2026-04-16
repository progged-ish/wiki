---
title: Merging Flask Servers Without Version Control
created: 2026-04-15
updated: 2026-04-15
type: concept
tags: [flask, python, debugging, operations, engineering-lessons]
sources: []
---

# Merging Flask Servers Without Version Control

A hard-earned lesson from the NWS Dashboard + VILE dashboard merge on 2026-04-15,
when combining two large Flask apps into one `app.py` resulted in a NameError crash
that was difficult to diagnose without git history to fall back on.

## The Situation

The project had two Flask servers that needed to be unified:
- The NWS/VILE threat dashboard (port 5000, `app.py`)
- The WX Event Tracker (port 5001, `wx_events/dashboard/app.py`)

The goal was one `app.py` serving all routes. The server was large: ~146KB,
~4500 lines of Python with large inline HTML template strings (THREATS_HTML,
WIKI_HTML, SKEWT_HTML, DASHBOARD_HTML).

There was no version control on the project. No git. No backup taken before merging.

## What Went Wrong

During the merge, a previous agent inserted a duplicate `if __name__ == "__main__":` block
at approximately line 2168 — in the middle of the file, before all the large HTML
template variables were defined:

```python
# ... routes above this line ...

# ---------------------------------------------------------------------------
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)   # <-- ROGUE BLOCK


# ── Threats Template ──────────────────────────────────────────────────────────
THREATS_HTML = r'''...'''    # line 2176

WIKI_HTML = r'''...'''       # line 2793

SKEWT_HTML = r'''...'''      # line 3046

DASHBOARD_HTML = r'''...'''  # line 3582

# The real startup block, at end of file:
if __name__ == "__main__":
    # ... watchdog setup, cron, proper startup logic ...
    app.run(host="0.0.0.0", port=5000, debug=False)
```

When `python app.py` was run, Python executes sequentially. It hit the rogue
`app.run()` at line 2170 BEFORE reaching the HTML variable definitions.
Flask started serving immediately. Any route that referenced `DASHBOARD_HTML`
raised `NameError: name 'DASHBOARD_HTML' is not defined`.

## Why It Was Hard to Diagnose

1. **Import-time vs runtime**: `python -c "import app"` always succeeded — Python
   imports don't call `app.run()`, so all variables get defined fine. The bug only
   manifested when running the server (which triggers the early `app.run()`).

2. **Large file**: With 4500 lines, scanning for duplicate `if __name__` blocks is
   non-trivial without grep.

3. **No version control**: There was no `git diff` to show what had changed. Each
   diagnostic session started blind.

4. **Agent loop**: Without being able to quickly reproduce and fix, multiple agent
   sessions spun up making the same mistaken diagnosis.

5. **Silent swallowing**: Initial error messages were ambiguous — the NameError
   appeared in logs but the root cause (early `app.run()`) was not obvious from
   the traceback alone.

## The Fix

Search for all `app.run` calls in the file:

```bash
grep -n "app.run" app.py
```

Found two: line 2170 (rogue) and line ~4490 (real). Removed the rogue block:

```python
# REMOVED this block — was at line ~2168:
# if __name__ == "__main__":
#     app.run(host="0.0.0.0", port=5000, debug=False)
```

Verified with a route test before restarting:

```python
python -c "
import app
with app.app.test_client() as c:
    resp = c.get('/')
    print(f'/ -> HTTP {resp.status_code}')
"
```

Got HTTP 200. Started server with nohup. All routes 200.

## Lessons

### Before Any Merge — Do This

1. **Make a backup**: `cp app.py app.py.backup-$(date +%Y%m%d-%H%M%S)` before touching anything.
   Even without git, a timestamped backup takes 2 seconds and saves hours.

2. **Initialize git immediately if not present**: `git init && git add . && git commit -m "pre-merge snapshot"`.
   Even a local git repo (no remote) gives you `git diff` and `git checkout` fallbacks.

3. **Audit structure before merging**: Run `grep -n "if __name__" app.py` and `grep -n "app.run" app.py`
   to catalog all entry points. There should be exactly one of each in a clean Flask file.

### The Structure Rule for Large Flask Apps

In a Flask app with large inline HTML templates, follow this layout strictly:

```
1. imports
2. Flask app = Flask(__name__)
3. configuration
4. helper functions / data loaders
5. route handlers (all @app.route)
6. <<< ALL template variables (DASHBOARD_HTML, etc.) must be HERE at module level >>>
7. if __name__ == "__main__":
       startup logic
       app.run(...)
```

Never insert `app.run()` before all module-level variables are defined.
Flask's request handling references these variables at call time, not import time,
so the NameError only surfaces when a request is served — not during startup.

### Diagnosing NameError in Flask Without Version Control

If you see `NameError: name 'X' is not defined` in a Flask route:

```bash
# 1. Check if variable is defined at module level at all
grep -n "^X = \|^X=\|^X =" app.py

# 2. Check for duplicate app.run / if __name__ blocks
grep -n "app\.run\|if __name__" app.py

# 3. Verify import works (does NOT prove runtime works):
python -c "import app; print(hasattr(app, 'X'))"

# 4. Verify route works with test client (DOES prove runtime):
python -c "
import app
with app.app.test_client() as c:
    r = c.get('/')
    print(r.status_code)
"

# 5. Check line order — if the variable is defined AFTER app.run, move it before
python -c "
import ast, sys
with open('app.py') as f: src = f.read()
tree = ast.parse(src)
for node in ast.walk(tree):
    if isinstance(node, ast.Assign):
        for t in node.targets:
            if isinstance(t, ast.Name) and 'HTML' in t.id:
                print(f'  {t.id} defined at line {node.lineno}')
"
```

### When Agent Sessions Loop

If you see an agent spending multiple sessions re-diagnosing the same issue,
it usually means:
- No reproducible test (add `app.test_client()` validation)
- The real state isn't being checked (processes from previous sessions still running)
- The bug is order-dependent (works in isolation, fails in context)

Always kill old processes and verify port state before re-running:

```bash
pkill -f "python.*app.py"
sleep 2
ss -tlnp | grep 5000
```

## Related Pages

- [[nam-data-pipeline]] — the pipeline this dashboard serves
- [[vile-orchestrator]] — the VILE framework whose threats are displayed

## Action Items

- [ ] Initialize git in `/home/progged-ish/nws_dashboard/` to prevent future no-version-control situations
- [ ] Add a `Makefile` or `start.sh` that always checks for duplicate `app.run()` calls before starting
