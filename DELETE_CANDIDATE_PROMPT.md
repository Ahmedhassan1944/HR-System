# Fix Prompt — Permanent Candidate Deletion
> Files: `Database.gs` + `Script.html`  
> Goal: Delete a candidate permanently from every data source — Sheets + Drive — with a confirmation modal, then redirect to the candidates list.

---

## What gets deleted (complete wipe)

| Source | What is removed |
|---|---|
| `tbl_Candidates` | The candidate's own row |
| `tbl_Documents` | Every document row where `CandidateID` matches |
| `tbl_Events` | Every follow-up event row where `CandidateID` matches |
| `tbl_SystemLogs` | Every audit log row where `CandidateID` matches |
| Google Drive | The candidate's Drive folder moved to Trash (`setTrashed(true)`) |

Only **Admin** and **HR** roles can execute the delete.

---

## PART A — Backend: `Database.gs`

### ADD this entire new function anywhere after `api_updateCandidateDetails`:

```js
/**
 * Permanently deletes a candidate and ALL associated data:
 *   - Row in tbl_Candidates
 *   - All rows in tbl_Documents  where CandidateID matches
 *   - All rows in tbl_Events     where CandidateID matches
 *   - All rows in tbl_SystemLogs where CandidateID matches
 *   - Candidate's Google Drive folder (moved to Trash)
 * Restricted to Admin and HR roles only.
 */
function api_deleteCandidate(candidateId) {
  const auth = requireRole_(['Admin', 'HR']);
  if (!auth.authorized) return { success: false, error: auth.error };
  if (!candidateId) return { success: false, error: 'candidateId is required.' };

  try {
    const ss = getSpreadsheet_();

    // ── 1. Delete from tbl_Candidates ─────────────────────────────────────
    let driveFolderId = '';
    (function () {
      const sheet   = ss.getSheetByName(SHEET_CANDIDATES);
      const data    = sheet.getDataRange().getValues();
      const idCol   = data[0].indexOf('CandidateID');
      const fldCol  = data[0].indexOf('DriveFolderID');
      // Iterate bottom-up so row indices stay valid after deletion
      for (let i = data.length - 1; i >= 1; i--) {
        if (String(data[i][idCol]) === String(candidateId)) {
          driveFolderId = data[i][fldCol] || '';
          sheet.deleteRow(i + 1);
          break;
        }
      }
    })();

    // ── 2. Delete all rows in tbl_Documents ───────────────────────────────
    (function () {
      const sheet  = ss.getSheetByName(SHEET_DOCUMENTS);
      const data   = sheet.getDataRange().getValues();
      const idCol  = data[0].indexOf('CandidateID');
      for (let i = data.length - 1; i >= 1; i--) {
        if (String(data[i][idCol]) === String(candidateId)) sheet.deleteRow(i + 1);
      }
    })();

    // ── 3. Delete all rows in tbl_Events ──────────────────────────────────
    (function () {
      const sheet  = ss.getSheetByName(SHEET_EVENTS);
      const data   = sheet.getDataRange().getValues();
      const idCol  = data[0].indexOf('CandidateID');
      for (let i = data.length - 1; i >= 1; i--) {
        if (String(data[i][idCol]) === String(candidateId)) sheet.deleteRow(i + 1);
      }
    })();

    // ── 4. Delete all rows in tbl_SystemLogs ──────────────────────────────
    (function () {
      const sheet  = ss.getSheetByName(SHEET_LOGS);
      const data   = sheet.getDataRange().getValues();
      const idCol  = data[0].indexOf('CandidateID');
      for (let i = data.length - 1; i >= 1; i--) {
        if (String(data[i][idCol]) === String(candidateId)) sheet.deleteRow(i + 1);
      }
    })();

    // ── 5. Trash the Drive folder ──────────────────────────────────────────
    if (driveFolderId) {
      try {
        DriveApp.getFolderById(driveFolderId).setTrashed(true);
      } catch (driveErr) {
        // Non-fatal: folder may already be deleted or inaccessible
        Logger.log('Drive folder trash warning: ' + driveErr.message);
      }
    }

    // ── 6. Invalidate all related caches ──────────────────────────────────
    const cache = CacheService.getScriptCache();
    cache.removeAll([
      'dashboard_data',
      'all_candidates',
      'all_documents',
      'all_upcoming_events',
      'docs_'    + candidateId,
      'events_'  + candidateId
    ]);

    Logger.log('Candidate permanently deleted: ' + candidateId);
    return { success: true };

  } catch (e) {
    Logger.log('api_deleteCandidate error: ' + e.message);
    return { success: false, error: e.message };
  }
}
```

---

## PART B — Frontend: `Script.html`

### B1 — Add the Delete button to the candidate detail header

**FIND** (inside `candidateDetail`, the `page-header__actions` div — last button in the group):

```js
          <button class="btn btn--primary" onclick="Views._openUploadModal('${escHtml(c.CandidateID)}','${escHtml(c.DriveFolderID || '')}')">
            📤 Upload Document
          </button>
        </div>
      </div>
```

**REPLACE WITH:**

```js
          <button class="btn btn--primary" onclick="Views._openUploadModal('${escHtml(c.CandidateID)}','${escHtml(c.DriveFolderID || '')}')">
            📤 Upload Document
          </button>
          <button class="btn btn--danger btn--sm" onclick="Views._confirmDeleteCandidate('${escHtml(c.CandidateID)}','${escHtml(c.FullName)}')">
            🗑️ Delete
          </button>
        </div>
      </div>
```

---

### B2 — Add the `_confirmDeleteCandidate` function

**FIND** any existing Views helper function — for example find this line:

```js
    _deleteEvent(eventId) {
```

**ADD the following block immediately BEFORE that line** (or at the end of the `Views` object before the closing `};`, whichever is easier):

```js
    _confirmDeleteCandidate(candidateId, candidateName) {
      Modal.open(`
        <div class="modal__header">
          <h2 class="modal__title">🗑️ Delete Candidate</h2>
          <button class="modal__close" onclick="Modal.close()">&times;</button>
        </div>
        <div class="modal__body">
          <p>Are you sure you want to <strong>permanently delete</strong> <strong>${escHtml(candidateName)}</strong>?</p>
          <p style="color:var(--c-danger);margin-top:var(--sp-2);font-size:.9rem">
            ⚠️ This will erase the candidate, all their documents, all follow-up events,
            and the audit log from the database and Google Drive — with no way to recover them.
          </p>
        </div>
        <div class="modal__footer">
          <button class="btn btn--outline" onclick="Modal.close()">Cancel</button>
          <button class="btn btn--danger" id="btn-confirm-delete-candidate">Yes, Delete Permanently</button>
        </div>
      `);
      document.getElementById('btn-confirm-delete-candidate').onclick = async () => {
        try {
          Modal.close();
          App.setState({ loading: true });
          renderTopbar();
          const res = await GAS.call('api_deleteCandidate', candidateId);
          if (res.success) {
            // Remove from local state so the candidates list is immediately clean
            App.setState({
              candidates: App.state.candidates.filter(c => c.CandidateID !== candidateId),
              selectedCandidate: null
            });
            Toast.success('Candidate deleted permanently.');
            Router.navigate('candidates');
          } else {
            Toast.error(res.error || 'Delete failed. Please try again.');
          }
        } catch (e) {
          Toast.error(e.message);
        } finally {
          App.setState({ loading: false });
          renderTopbar();
        }
      };
    },
```

---

### B3 — Add mock entry for local / TestSprite testing

**FIND** the mock entries block inside `GAS._mock` (or wherever the other `api_*` mock cases live):

```js
      if (fnName === 'api_markEventCompleted') return { success: true };
```

**ADD immediately after that line:**

```js
      if (fnName === 'api_deleteCandidate') return { success: true };
```

> If this mock line already exists, skip this step — do not duplicate it.

---

## CSS note (only if `btn--danger` is not already defined)

If the red danger button style is missing, add this to `Styles.html` (or the `<style>` block in `Script.html`):

```css
.btn--danger {
  background: var(--c-danger, #dc2626);
  color: #fff;
  border-color: transparent;
}
.btn--danger:hover {
  background: #b91c1c;
}
```

---

## Verification checklist

After pushing and redeploying as a new Web App version:

1. Open any candidate detail page — confirm a red **🗑️ Delete** button appears in the top-right action bar.
2. Click Delete — a confirmation modal appears with a warning message and two buttons (Cancel / Yes, Delete Permanently).
3. Click Cancel — modal closes, candidate is unchanged.
4. Click Delete on a test candidate — modal closes → loading spinner → success toast → redirected to `/candidates` list → deleted candidate is gone from the list.
5. Open Google Sheets — confirm the candidate's row is gone from `tbl_Candidates`, `tbl_Documents`, `tbl_Events`, and `tbl_SystemLogs`.
6. Open Google Drive → Trash — confirm the candidate's folder appears there.
