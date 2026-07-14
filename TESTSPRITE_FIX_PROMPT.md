# Fix Prompt — HR Mobilization System (Version 3.2.1)
> Prepared from TestSprite MCP Server report (2026-07-06).  
> Apply every section in order. Each section = one independent fix.

---

## Context for the AI

This project is a **Google Apps Script SPA** (no login page — Google OAuth happens automatically at the GAS layer). The frontend lives entirely in `Script.html`. All GAS API calls go through a `GAS.call(fnName, ...args)` helper that wraps `google.script.run`. When the app is opened in a plain browser (e.g. via Python HTTP server on port 5500 for local testing / TestSprite automation), `google.script.run` is `undefined` — the app should instantly fall back to built-in mock data **without hanging**.

**TestSprite ran 38 automated browser tests and ALL 38 timed out (>15 min each).** The root cause is two things working together:
1. The `GAS.call()` mock fallback is either not firing or is firing synchronously, blocking the JS event loop and preventing the UI from rendering before the test timeout hits.
2. Every test tries to navigate to `/login` first — a route that does not exist in this SPA — so the router falls through and nothing renders.

Three additional code issues (pre-existing) are bundled here because they are small and must also be fixed.

---

## FIX 1 — Make `GAS.call()` mock always resolve asynchronously (CRITICAL — fixes all 38 test timeouts)

**Root cause:** When `google` is undefined (local browser / TestSprite), the fallback mock must call the success handler on the **next tick** (`setTimeout(cb, 0)`), not synchronously and not by throwing. If it throws, the error is swallowed and nothing renders. If it resolves synchronously, some rendering pipelines that depend on the async cycle never complete.

**Find this pattern in `Script.html`** (the `GAS` object / `GAS.call` function — look for where `google.script.run` is referenced with a typeof check):

```js
// FIND — the existing mock branch inside GAS.call (exact whitespace may vary):
      if (typeof google === 'undefined') {
        // mock fallback
        const mockResult = GAS._mock(fnName, args);
        successHandler(mockResult);
        return;
      }
```

**Replace with:**

```js
      if (typeof google === 'undefined') {
        // Always resolve asynchronously so the call-site's rendering code
        // runs after the current stack unwinds — prevents UI hang in
        // local-browser / TestSprite environments.
        setTimeout(function () {
          try {
            const mockResult = GAS._mock(fnName, args);
            successHandler(mockResult);
          } catch (e) {
            if (failureHandler) failureHandler(e);
            else console.error('[GAS mock error]', fnName, e);
          }
        }, 0);
        return;
      }
```

> **Note:** If the existing code already uses `setTimeout` but with a delay > 0 (e.g. `setTimeout(fn, 300)`), change it to `setTimeout(fn, 0)`. The delay must be 0 — any artificial delay multiplied across dozens of API calls per view adds up to test timeouts.

---

## FIX 2 — Add a `/login` route handler so TestSprite's navigation doesn't hang (CRITICAL)

**Root cause:** Every TestSprite test opens `/login` first. The SPA router does not have a `login` route, so the `Router.navigate('login')` call either throws, silently no-ops, or leaves a blank screen. TestSprite then waits forever for elements that never appear.

**Find the router's route definitions / switch-case or `routes` map in `Script.html`.** It will look something like:

```js
// FIND — the routes map or switch that handles view names:
    routes: {
      'dashboard': Views.renderDashboard,
      'candidates': Views.renderCandidates,
      // ... other routes
    },
```

**Replace with** (add the `login` entry):

```js
    routes: {
      'login':      function() { Router.navigate('dashboard', {}, true); }, // GAS auth is automatic — redirect to dashboard
      'dashboard':  Views.renderDashboard,
      'candidates': Views.renderCandidates,
      // ... keep all other existing routes unchanged
    },
```

> If the router uses a `switch` statement instead of a map, add a `case 'login': Router.navigate('dashboard', {}, true); break;` before the `default` case.  
> The boolean `true` in `navigate` should be whatever your router uses to signal "replace history entry instead of push" — check how other redirects are done and match that pattern. If there is no such flag, just call `Router.navigate('dashboard')`.

---

## FIX 3 — Fetch real user role from backend instead of hardcoded placeholder

**Root cause:** Line ~20 of `Script.html` initialises `App.state` with a hardcoded role string instead of fetching the actual role from the backend. The RBAC UI (hide/show edit buttons etc.) therefore shows the wrong controls for every user except the one the developer hard-coded.

**Find in `Script.html`** (inside the `App.state` initialisation object):

```js
      userRole: 'HR_COORDINATOR',
```

**Replace with:**

```js
      userRole: null,  // populated by api_getCurrentUser on app init
```

**Then find the app initialisation function** (the function that runs on `DOMContentLoaded` or equivalent, where `api_getDashboardData` or the first GAS call is made). Add the following call **at the very start** of that init sequence, before rendering any view:

```js
    // Fetch real user role first — RBAC controls depend on this
    GAS.call('api_getCurrentUser')
      .then(function(result) {
        if (result && result.success && result.data) {
          App.state.userRole = result.data.role;   // e.g. 'Admin', 'HR', 'Coordinator', 'Viewer'
          App.state.userEmail = result.data.email;
          App.state.userName  = result.data.name;
        }
      })
      .catch(function() {
        // Non-fatal: leave userRole as null; write-action guards on the
        // backend will still protect data even if UI doesn't hide buttons.
        console.warn('[Auth] Could not fetch user role — RBAC UI may be inaccurate.');
      });
```

**Also add a mock entry in `GAS._mock`** so local/TestSprite runs return a sensible default:

```js
// FIND in GAS._mock (or wherever the mock data map is defined):
// Add this case alongside the other api_* cases:

      case 'api_getCurrentUser':
        return {
          success: true,
          data: { email: 'hr@example.com', name: 'HR User', role: 'HR' }
        };
```

---

## FIX 4 — Fix 17-value seed rows in `Tests.js` (missing `Batch_Number` column)

**Root cause:** `tbl_Candidates` has **18 columns** (CandidateID, FullName, Position, Department, Email, Phone, Nationality, OfferSalary, AssignedCoordinatorEmail, CurrentStatus, CreatedAt, UpdatedAt, DriveFolderID, Notes, LocalPath, Batch_Number, … check the exact column order in `Database.js` `CANDIDATE_HEADERS` constant). The two seed rows in `Tests.js` each provide only 17 values, so the last column is always `undefined` in tests — any test that reads or filters by the missing column silently gets wrong data.

**Find in `Tests.js`** — there are two seed rows. They look like:

```js
// FIND — first seed row (17 values):
    ['C001', 'Ahmed Hassan', 'Engineer', 'IT', 'ahmed@example.com', '0501234567',
     'Saudi', 8000, 'coord@example.com', 'Documents Complete',
     '2024-01-01', '2024-06-01', 'folder123', 'Test notes', '/local/path'],
```

**Replace with** (add the empty `Batch_Number` as the 18th value):

```js
// REPLACE — first seed row (18 values):
    ['C001', 'Ahmed Hassan', 'Engineer', 'IT', 'ahmed@example.com', '0501234567',
     'Saudi', 8000, 'coord@example.com', 'Documents Complete',
     '2024-01-01', '2024-06-01', 'folder123', 'Test notes', '/local/path', ''],
```

**Find in `Tests.js`** — the second seed row (also 17 values):

```js
// FIND — second seed row (17 values):
    ['C002', 'Sara Ali', 'HR Specialist', 'HR', 'sara@example.com', '0507654321',
     'Egyptian', 6000, 'coord@example.com', 'New Candidate',
     '2024-02-01', '2024-06-15', 'folder456', '', ''],
```

**Replace with:**

```js
// REPLACE — second seed row (18 values):
    ['C002', 'Sara Ali', 'HR Specialist', 'HR', 'sara@example.com', '0507654321',
     'Egyptian', 6000, 'coord@example.com', 'New Candidate',
     '2024-02-01', '2024-06-15', 'folder456', '', '', ''],
```

> **Verify the column order:** Before applying, confirm the 18th column name by looking at the `CANDIDATE_HEADERS` constant in `Database.js`. If the column is not `Batch_Number`, use whatever the actual 18th header is. The value should be an empty string `''` for test seed rows.

---

## Verification checklist after applying all fixes

After pushing and redeploying:

1. **Local browser test:** Open `index.html` in a plain browser (no GAS environment). The dashboard should render in < 2 seconds with mock data. No spinning loaders or blank screens.
2. **Navigation to `/login`:** Manually append `#login` (or `/login`, depending on router hash mode) to the URL — it should immediately redirect to the dashboard view.
3. **Role display:** Open the browser console and check `App.state.userRole` — should be `'HR'` (from mock) instead of `'HR_COORDINATOR'`.
4. **Unit tests:** Run the internal test suite (open `Tests.js` in the Apps Script editor and run `runAllTests()`). All 27 tests should pass with no column-mismatch errors.
5. **Re-run TestSprite:** With fixes 1 and 2 in place, TestSprite should be able to navigate past the login redirect and reach a rendered dashboard within the timeout window. Tests that were previously killed at TC001 should now at least reach the dashboard assertion step.

---

## Summary of files changed

| File | Change |
|---|---|
| `Script.html` | FIX 1: `GAS.call` mock uses `setTimeout(fn, 0)` |
| `Script.html` | FIX 2: Router handles `'login'` → redirect to `'dashboard'` |
| `Script.html` | FIX 3: `userRole` initialised as `null`, fetched via `api_getCurrentUser` on init; mock case added |
| `Tests.js` | FIX 4: Both seed rows extended to 18 values (add trailing `''`) |
