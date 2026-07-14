# Fix Prompt — Remove Top-Right Topbar Content
> Apply to: `Script.html`  
> Goal: Make the top-right corner of the topbar completely empty.

---

## What to change

Inside the `renderTopbar` function (around line 351), find the `topbar__right` div and everything inside it — the notification bell button and the user profile block — and delete their contents entirely.

### FIND (inside `renderTopbar`):

```js
  <div class="topbar__right">
      <button class="notification-btn" id="notif-btn" aria-label="Notifications">
        <span class="notification-btn__dot" aria-hidden="true"></span>
      </button>
      <div class="topbar__user" tabindex="0" role="button" aria-label="User profile">
        <div class="topbar__avatar" aria-hidden="true">${fmt.initials(name)}</div>
        <div>
          <div class="topbar__username">${escHtml(name)}</div>
          <div class="topbar__role-badge">${escHtml(role)}</div>
        </div>
      </div>
    </div>
```

### REPLACE WITH (empty container — no children):

```js
  <div class="topbar__right"></div>
```

---

## Result

The top-right corner of the topbar will be completely empty.  
The brand logo/title on the left remains unchanged.  
No other functionality is affected.
