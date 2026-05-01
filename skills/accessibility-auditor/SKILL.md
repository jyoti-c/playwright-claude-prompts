---
name: accessibility-auditor
description: >
  Runs a full automated accessibility audit on any website using Playwright MCP and axe-core (WCAG 2.0/2.1 AA + best practices). Use this skill whenever the user asks to: check a website for accessibility issues, run an accessibility audit, find WCAG violations, audit a URL for a11y problems, generate accessibility bug reports, or test a site for screen-reader/keyboard/contrast issues. Also trigger when the user says "check accessibility", "audit this site", "find a11y bugs", or pastes a URL and mentions disabilities, compliance, or accessibility. Always use this skill — do not attempt manual inspection without axe-core injection.
---

# Accessibility Auditor Skill

Automated accessibility audit using Playwright MCP + axe-core. Produces structured bug reports per violation.

---

## Prerequisites

- **Playwright MCP** must be connected (browser_navigate, browser_run_code, browser_snapshot, browser_take_screenshot)
- **axe-core** is injected at runtime from CDN — no install needed
- Network must allow: `cdnjs.cloudflare.com`

---

## Workflow

### Step 1 — Get the URL

If the user hasn't provided a URL, ask for one. If they provided it, proceed immediately.

---

### Step 2 — Navigate & Screenshot

```
playwright:browser_navigate  →  url: "<the URL>"
playwright:browser_take_screenshot  →  fullPage: true, filename: "audit-screenshot.png"
playwright:browser_snapshot  →  (capture accessibility tree)
```

Take a full-page screenshot first so you have visual context for each bug.

---

### Step 3 — Inject axe-core and Run Audit

Use `browser_run_code` with this exact script:

```javascript
async (page) => {
  // Inject axe-core 4.9 from CDN
  await page.addScriptTag({
    url: 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.9.1/axe.min.js'
  });

  // Small delay to ensure injection is complete
  await page.waitForTimeout(500);

  const results = await page.evaluate(async () => {
    return await axe.run(document, {
      runOnly: {
        type: 'tag',
        values: ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'best-practice']
      },
      resultTypes: ['violations', 'incomplete']
    });
  });

  return JSON.stringify({
    url: window.location.href,
    violations: results.violations,
    incomplete: results.incomplete,
    timestamp: new Date().toISOString()
  }, null, 2);
}
```

Parse the returned JSON. The key fields are:
- `violations` — confirmed failures (must fix)
- `incomplete` — needs manual review (may be failures)

---

### Step 4 — Analyze & Classify Issues

For each violation, extract:
| Field | Source |
|---|---|
| Rule ID | `violation.id` |
| Impact | `violation.impact` (critical / serious / moderate / minor) |
| WCAG Tags | `violation.tags` (filter for `wcag*`) |
| Description | `violation.description` |
| Help URL | `violation.helpUrl` |
| Affected HTML | `node.html` for each node in `violation.nodes` |
| CSS Selector | `node.target` |
| Failure Reason | `node.failureSummary` |

**Impact → Severity mapping:**
| axe impact | Bug Severity |
|---|---|
| critical | P0 — Blocker |
| serious | P1 — High |
| moderate | P2 — Medium |
| minor | P3 — Low |

---

### Step 5 — Output: Bug Report

Produce the report in this format:

---

## Accessibility Audit Report

**URL:** `<url>`  
**Audit Date:** `<timestamp>`  
**Tool:** axe-core 4.9 (WCAG 2.0/2.1 A + AA + Best Practices)  
**Total Violations:** `<N>`  
**Needs Manual Review:** `<M>`

---

### Summary Table

| # | Rule ID | Impact | WCAG Criterion | Occurrences | Severity |
|---|---------|--------|----------------|-------------|----------|
| 1 | `rule-id` | critical | 1.1.1 | 3 | P0 — Blocker |
| … | … | … | … | … | … |

---

### Bug Reports

For each violation, produce one bug card:

---

#### BUG-001 — `<rule-id>`: `<short description>`

| Field | Value |
|---|---|
| **ID** | BUG-001 |
| **Severity** | P0 — Blocker (critical) |
| **WCAG** | 1.1.1 Non-text Content (Level A) |
| **Standard** | WCAG 2.1 |
| **Category** | Images / Forms / Color / Keyboard / Structure / … |
| **Rule** | `axe rule id` |
| **Reference** | `https://dequeuniversity.com/…` |

**Description**  
One sentence: what the rule checks and why it matters.

**Occurrences** (`N` affected elements)

For each affected node:
```
Element: <html snippet>
Selector: div > img:nth-child(2)
Failure: Missing alt attribute on informational image
```

**Steps to Reproduce**
1. Open `<url>` in a browser
2. Inspect `<selector>`
3. Observe: `<failure reason>`

**Expected Behavior**  
What the element should do/have to pass.

**Recommended Fix**  
Concrete code suggestion, e.g.:
```html
<!-- Before -->
<img src="hero.png">

<!-- After -->
<img src="hero.png" alt="Team collaborating around a whiteboard">
```

**Impact on Users**  
Who is affected: screen reader users / keyboard-only users / low-vision users / etc.

---

*(Repeat card for each violation)*

---

### Needs Manual Review

List `incomplete` items as a simple table with selectors and what to check manually.

---

### Recommendations Summary

After all bug cards, add a prioritized action list:

1. **Fix immediately (P0/P1):** … 
2. **Fix this sprint (P2):** …
3. **Backlog (P3):** …
4. **Manual checks needed:** …

---

## Tips & Edge Cases

- **SPAs / dynamic content**: After navigating, add a `waitForTimeout(2000)` before running axe to let JS render.
- **Auth-gated pages**: Ask the user to log in manually in the browser first, then trigger the audit.
- **iframes**: axe scans same-origin iframes automatically; cross-origin iframes are skipped (note this in the report).
- **Large pages**: axe may time out on very large DOMs. If it does, scope the audit: `axe.run('#main')` instead of `document`.
- **Color contrast**: axe checks color contrast automatically when `wcag2aa` is in scope. Ensure background/foreground aren't set by JS after load.
- **False positives in `incomplete`**: Many `incomplete` entries are safe to dismiss — describe what manual action is needed rather than marking them as failures.

---

## Output File

Save the full report as a markdown file:

```
/mnt/user-data/outputs/accessibility-report-<domain>-<date>.md
```

Then call `present_files` to share it with the user.

Also share the screenshot (`audit-screenshot.png`) via `present_files` so the user has visual context.
