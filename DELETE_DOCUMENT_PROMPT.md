# Fix Prompt — Delete Individual Document (Permanent)
> Files: `Database.gs` + `Script.html`
> Goal: Add a 🗑️ Delete button on every document row inside the candidate detail page.
> Clicking it shows a confirmation modal → deletes the row from `tbl_Documents` → trashes
> the file from Google Drive → refreshes the documents section in place (no page reload).

---

## What gets deleted per document

| Source | What is removed |
|---|---|
| `tbl_Documents` | The single row matching `DocumentID` |
| Google Drive | The actual file moved to Trash (`setTrashed(true)`) |
| Cache | `docs_{candidateId}`, `dashboard_data`, `all_documents` invalidated |
| Audit Log | Entry written: "Document Deleted: {DocType}" |

Only **Admin** and **HR** roles can delete documents.

---

## PART A — Backend: `Database.gs`

### ADD this entire new function (place it near `api_reviewDocument`):

```js
/**
 * Permanently deletes a single document:
 *   - Removes its row from tbl_Documents (matched by DocumentID)
 *   - Moves the actual Drive file to Trash (extracted from FileURL)
 *   - Invalidates related caches
 *   - Writes an audit log entry
 * Restricted to Admin and HR roles only.
 */
function api_deleteDocument(documentId, candidateId) {
  const auth = requireRole_(['Admin', 'HR']);
  if (!auth.authorized) return { success: false, error: auth.error };
  if (!documentId)  return { success: false, error: 'documentId is required.' };
  if (!candidateId) return { success: false, error: 'candidateId is required.' };

  try {
    const sheet   = getSheet_(SHEET_DOCUMENTS);
    const data    = sheet.getDataRange().getValues();
    const headers = data[0];
    const idCol   = headers.indexOf('DocumentID');
    const urlCol  = headers.indexOf('FileURL');
    const typeCol = headers.indexOf('DocType');

    let fileURL = '';
    let docType = '';
    let found   = false;

    // Iterate bottom-up to keep row indices valid after deletion
    for (let i = data.length - 1; i >= 1; i--) {
      if (String(data[i][idCol]) === String(documentId)) {
        fileURL = data[i][urlCol] || '';
        docType = data[i][typeCol] || 'Document';
        sheet.deleteRow(i + 1);
        found = true;
        break;
      }
    }

    if (!found) return { success: false, error: 'Document not found.' };

    // ── Trash the Drive file ───────────────────────────────────────────────
    // Drive file URLs come in two forms:
    //   https://drive.google.com/file/d/FILE_ID/view
    //   https://drive.google.com/open?id=FILE_ID
    if (fileURL && fileURL !== '#') {
      try {
        const match = fileURL.match(/\/d\/([a-zA-Z0-9_-]{10,})/) ||
                      fileURL.match(/[?&]id=([a-zA-Z0-9_-]{10,})/);
        if (match && match[1]) {
          DriveApp.getFileById(match[1]).setTrashed(true);
        }
      } catch (driveErr) {
        // Non-fatal — file may already be deleted or user lacks Drive access
        Logger.log('Drive file trash warning: ' + driveErr.message);
      }
    }

    // ── Audit log ─────────────────────────────────────────────────────────
    api_writeLog_(
      candidateId,
      Session.getActiveUser().getEmail(),
      'Document Deleted: ' + docType + ' (ID: ' + documentId + ')'
    );

    // ── Invalidate caches ─────────────────────────────────────────────────
    CacheService.getScriptCache().removeAll([
      'dashboard_data',
      'all_documents',
      'docs_' + candidateId
    ]);

    return { success: true };

  } catch (e) {
    Logger.log('api_deleteDocument error: ' + e.message);
    return { success: false, error: e.message };
  }
}
```

---

## PART B — Frontend: `Script.html`

### B1 — Add Delete button to every document row

**FIND** the document row rendering map (inside the async function that loads documents):

```js
          <div class="doc-row__actions">
            ${d.FileURL && d.FileURL !== '#' ? `<a href="${escHtml(d.FileURL)}" target="_blank" rel="noopener" class="btn btn--outline btn--sm" aria-label="View ${escHtml(d.DocType)}">View ↗</a>` : ''}
          </div>
        </div>`).join('');
```

**REPLACE WITH** (add the delete button after the View link):

```js
          <div class="doc-row__actions">
            ${d.FileURL && d.FileURL !== '#' ? `<a href="${escHtml(d.FileURL)}" target="_blank" rel="noopener" class="btn btn--outline btn--sm" aria-label="View ${escHtml(d.DocType)}">View ↗</a>` : ''}
            <button
              class="btn btn--danger btn--sm"
              aria-label="Delete ${escHtml(d.DocType)}"
              onclick="Views._confirmDeleteDocument('${escHtml(d.DocumentID)}','${escHtml(d.CandidateID)}','${escHtml(d.DocType)}','${escHtml(d.FileName)}')">
              🗑️
            </button>
          </div>
        </div>`).join('');
```

---

### B2 — Add the `_confirmDeleteDocument` function

**FIND** the `_confirmDeleteCandidate` function (or any nearby Views helper). **ADD the following block immediately before it**:

```js
    _confirmDeleteDocument(documentId, candidateId, docType, fileName) {
      Modal.open(`
        <div class="modal__header">
          <h2 class="modal__title">🗑️ Delete Document</h2>
          <button class="modal__close" onclick="Modal.close()">&times;</button>
        </div>
        <div class="modal__body">
          <p>Are you sure you want to permanently delete:</p>
          <p style="margin-top:var(--sp-2);font-weight:600">${escHtml(docType)} — ${escHtml(fileName)}</p>
          <p style="color:var(--c-danger);margin-top:var(--sp-2);font-size:.875rem">
            ⚠️ The file will be removed from Google Drive and cannot be recovered.
          </p>
        </div>
        <div class="modal__footer">
          <button class="btn btn--outline" onclick="Modal.close()">Cancel</button>
          <button class="btn btn--danger" id="btn-confirm-delete-doc">Yes, Delete</button>
        </div>
      `);
      document.getElementById('btn-confirm-delete-doc').onclick = async () => {
        try {
          Modal.close();
          App.setState({ loading: true });
          renderTopbar();
          const res = await GAS.call('api_deleteDocument', documentId, candidateId);
          if (res.success) {
            Toast.success(escHtml(docType) + ' deleted successfully.');
            // Reload documents section in place — no full page reload needed
            await Views._loadCandidateDocuments(candidateId);
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

**FIND** the mock block that contains the other `api_delete*` cases:

```js
      if (fnName === 'api_deleteCandidate') return { success: true };
```

**ADD immediately after:**

```js
      if (fnName === 'api_deleteDocument') return { success: true };
```

---

## Verification checklist

After pushing and redeploying as a new Web App version:

1. Open any candidate that has uploaded documents.
2. Each document row shows a small red 🗑️ button to the right of the **View ↗** link.
3. Click 🗑️ on a document → confirmation modal appears showing the document type and file name.
4. Click **Cancel** → modal closes, document row is unchanged.
5. Click **Yes, Delete** on a test document → modal closes → loading → success toast → the document row disappears from the list without reloading the page.
6. Open `tbl_Documents` in Google Sheets → confirm the row is gone.
7. Open Google Drive → Trash → confirm the file appears there.
8. Open the Audit Log for that candidate → confirm an entry: *"Document Deleted: {DocType}"*.
