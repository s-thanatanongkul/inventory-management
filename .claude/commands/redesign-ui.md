---
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface with a vertical sidebar navigation, consistent spacing, and a polished professional look
---

Redesign this Vue 3 application's UI into a modern SaaS-style interface. Follow every step below in order.

---

## Step 1 — Audit the current layout

Read `client/src/App.vue` and all files in `client/src/views/` and `client/src/components/`. Understand:
- The current navigation structure (top nav, links, active states)
- Global CSS classes defined in `App.vue` (card, badge, table, stat-card, etc.)
- Any composables used for filters, auth, or i18n (`client/src/composables/`)
- The router setup in `client/src/main.js`

---

## Step 2 — Design the sidebar layout

**Delegate to vue-expert** to rewrite `client/src/App.vue` with a two-column layout:

```
┌──────────────────────────────────────────────┐
│  Sidebar (240px fixed)  │  Main content area  │
│                         │                     │
│  Logo / brand           │  <router-view />    │
│  Nav links (vertical)   │                     │
│  [bottom] user profile  │                     │
└──────────────────────────────────────────────┘
```

### Layout requirements

- **Outer shell**: `display: flex; height: 100vh; overflow: hidden`
- **Sidebar**: `width: 240px; flex-shrink: 0; height: 100vh; position: sticky; top: 0; overflow-y: auto`
- **Content area**: `flex: 1; overflow-y: auto; padding: 2rem`
- The FilterBar (if present) moves inside the content area, not the sidebar

### Sidebar sections

1. **Brand block** (top of sidebar, ~64px tall):
   - App name in bold, subtitle below it in muted color
   - Bottom border separating it from nav links

2. **Navigation links** (vertical stack):
   - Each link: full-width, `padding: 0.625rem 1rem`, `border-radius: 8px`, `display: flex; align-items: center; gap: 0.75rem`
   - Inactive: text `#64748b`, transparent background
   - Hover: `background: #f1f5f9`, text `#0f172a`
   - Active: `background: #eff6ff`, text `#2563eb`, left accent bar (`border-left: 3px solid #2563eb; padding-left: calc(1rem - 3px)`)
   - No bottom underline indicator (that was the top-nav pattern)

3. **User / profile block** (bottom of sidebar, pushed down with `margin-top: auto`):
   - Render any existing ProfileMenu and LanguageSwitcher components here

### Sidebar color
Use `background: #ffffff; border-right: 1px solid #e2e8f0` for a clean white sidebar. If you want a dark variant, use `background: #0f172a` with white/slate text and `border-right: none`.

---

## Step 3 — Update global styles in App.vue

Keep all existing utility classes (`.card`, `.badge`, `.stat-card`, `.table-container`, etc.) unchanged so views don't break.

Add or update:

```css
/* Shell */
.app {
  display: flex;
  height: 100vh;
  overflow: hidden;
  background: #f8fafc;
}

/* Sidebar */
.sidebar {
  width: 240px;
  flex-shrink: 0;
  height: 100vh;
  position: sticky;
  top: 0;
  display: flex;
  flex-direction: column;
  background: #ffffff;
  border-right: 1px solid #e2e8f0;
  overflow-y: auto;
}

.sidebar-brand {
  padding: 1.25rem 1rem;
  border-bottom: 1px solid #e2e8f0;
}

.sidebar-brand h1 {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.sidebar-brand .subtitle {
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.125rem;
  border-left: none;
  padding-left: 0;
}

.sidebar-nav {
  flex: 1;
  padding: 0.75rem 0.625rem;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.sidebar-nav a {
  display: flex;
  align-items: center;
  padding: 0.625rem 1rem;
  border-radius: 8px;
  color: #64748b;
  text-decoration: none;
  font-size: 0.9rem;
  font-weight: 500;
  transition: background 0.15s ease, color 0.15s ease;
  border-left: 3px solid transparent;
}

.sidebar-nav a:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.sidebar-nav a.active {
  background: #eff6ff;
  color: #2563eb;
  border-left-color: #2563eb;
}

.sidebar-bottom {
  padding: 0.75rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

/* Main content */
.main-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  min-width: 0;
}

.main-content {
  flex: 1;
  padding: 1.75rem 2rem;
  max-width: 1400px;
  width: 100%;
}
```

Remove the old `.top-nav`, `.nav-container`, `.nav-tabs` styles since the top bar is gone.

---

## Step 4 — Polish spacing and typography

**Delegate to vue-expert** to apply these refinements across all view files:

### Page headers
Standardize every view's page header:
```vue
<div class="page-header">
  <h2>Page Title</h2>
  <p>Brief description of this page.</p>
</div>
```
Ensure `page-header` has `margin-bottom: 1.5rem`.

### Stat cards
Ensure the stats grid uses:
```css
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem 1.5rem;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
}

.stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.stat-value {
  font-size: 2rem;
  font-weight: 700;
  letter-spacing: -0.025em;
  color: #0f172a;
}
```

### Cards
```css
.card {
  background: white;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
  padding: 1.25rem 1.5rem;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 1rem;
  margin-bottom: 1rem;
  border-bottom: 1px solid #f1f5f9;
}

.card-title {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.01em;
}
```

---

## Step 5 — Verify nothing broke

After all edits are complete:

1. Check that every route still has a corresponding nav link in the sidebar
2. Check that active route detection still works (`:class="{ active: $route.path === '...' }"`)
3. Check that the FilterBar (if it exists as a component) renders in the right place — inside `.main-wrapper`, above `<router-view />`
4. Check that no view imports `.top-nav` or `.nav-container` styles directly

Report a short summary of what was changed and flag anything that may need manual testing.
