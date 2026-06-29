# Header Clipping Bug — Root Cause Analysis & Fix

**Date:** March 18, 2026
**Affected Pages:** NegotiationRoom, ConversationRoom, SummaryPage (all pages under `/chatbot/requisitions/:rfqId/vendors/:vendorId/deals/:dealId`)
**Severity:** High — Users could not see the back button or deal title header, making navigation impossible without a full page refresh.

---

## Symptom

When navigating to a deal's NegotiationRoom page, the top ~70px of the page was clipped/hidden above the visible viewport. The header containing the **back button**, **deal title**, and **status badge** was invisible. Only the "NEGOTIATING Round 1" status line and Chat/Summary tabs were visible at the top edge.

The issue persisted when navigating back to other pages (RequisitionListPage, RequisitionDealsPage), where their headers were also clipped. A full page refresh was the only workaround.

---

## Root Cause

The `DashBoardLayout` outer container (`<div class="flex h-screen pt-4 px-4 overflow-hidden">`) was being **programmatically scrolled** despite having `overflow: hidden`.

### Key Finding

```
dashLayout.scrollTop: 69    ← The smoking gun
dashLayout.overflow: hidden
main.rect.top: -53          ← Pushed 53px above viewport
```

The DashBoardLayout div had `scrollTop: 69`, which pushed the entire `<main>` element (and all page content) 69px upward. Since the layout div has `pt-4` (16px padding), the `<main>` element's visual top moved from `+16px` to `-53px` (16 - 69 = -53).

### Why `overflow: hidden` Didn't Prevent This

In CSS, `overflow: hidden` only:
- Hides the scrollbar
- Prevents the user from scrolling via mouse/trackpad

It does **NOT** prevent programmatic scrolling. The following can still scroll an `overflow: hidden` element:
- `element.scrollIntoView()` — called by browsers when focusing an element
- `element.focus()` — browsers auto-scroll to bring focused elements into view
- Direct `element.scrollTop = value` assignment

### Likely Trigger

When the NegotiationRoom rendered chat messages and the Composer/read-only notice bar, a browser focus event (likely on a form element, button, or the chat transcript area) triggered `scrollIntoView` on the nearest scrollable ancestor — which was the DashBoardLayout div. Even though it has `overflow: hidden`, the browser still set its `scrollTop` to bring the focused element into view.

---

## The Fix

**File:** `src/Layout/DashBoardLayout.tsx`

Added a `scroll` event listener on the DashBoardLayout container that immediately resets `scrollTop` to 0 whenever anything attempts to scroll it:

```tsx
const layoutRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  const layoutEl = layoutRef.current;
  if (!layoutEl) return;
  const preventScroll = () => {
    if (layoutEl.scrollTop !== 0) {
      layoutEl.scrollTop = 0;
    }
    if (layoutEl.scrollLeft !== 0) {
      layoutEl.scrollLeft = 0;
    }
  };
  layoutEl.addEventListener('scroll', preventScroll);
  return () => layoutEl.removeEventListener('scroll', preventScroll);
}, []);

// Applied ref to the outer div:
<div ref={layoutRef} className="flex h-screen pt-4 px-4 pb-0 ... overflow-hidden">
```

### Why This Works

- The `scroll` event fires whenever `scrollTop` changes — even programmatically
- The listener immediately resets `scrollTop` to 0, counteracting any auto-scroll
- Since this div should never scroll (it's the viewport-locked layout container with `overflow: hidden`), resetting to 0 is always correct
- The fix is invisible to users — no visual flicker since the reset happens in the same frame

### Additional Fix: Scroll Position Reset on Navigation

Also added a scroll-to-top reset on route changes to prevent stale scroll positions when navigating between pages:

```tsx
const mainRef = useRef<HTMLElement>(null);
const { pathname } = useLocation();

useEffect(() => {
  if (mainRef.current) {
    mainRef.current.scrollTop = 0;
  }
  document.documentElement.scrollTop = 0;
  document.body.scrollTop = 0;
  window.scrollTo(0, 0);
}, [pathname]);
```

---

## Debugging Methodology

### The Problem With Initial Approaches

Several CSS-level fixes were attempted first, all unsuccessful:
1. **`h-full` instead of `calc(100vh - 2rem)`** — Didn't help because the issue wasn't the element's height
2. **`min-h-0` on flex children** — Didn't help; the element was correctly sized
3. **`absolute inset-0` positioning** — Didn't help; the parent itself was displaced
4. **`100dvh` (dynamic viewport height)** — Didn't help; it wasn't a browser toolbar issue
5. **`position: sticky` on header** — Didn't help; sticky is relative to the scroll container, which was the displaced `<main>`

These all failed because the problem wasn't CSS sizing — it was an **unexpected scroll state** on a parent element.

### The Diagnostic Overlay Technique

To identify the actual root cause, a temporary JavaScript debug overlay was injected into `index.html`:

```html
<!-- TEMPORARY DEBUG OVERLAY -->
<script>
  setInterval(function() {
    // Only activate on deal pages
    if (!window.location.pathname.includes('/deals/')) return;

    var main = document.querySelector('main');
    if (!main) return;

    // Create or find the overlay element
    var dbg = document.getElementById('_dbg_overlay');
    if (!dbg) {
      dbg = document.createElement('div');
      dbg.id = '_dbg_overlay';
      dbg.style.cssText = 'position:fixed;bottom:60px;right:10px;' +
        'background:red;color:white;padding:8px 12px;font-size:11px;' +
        'z-index:99999;border-radius:6px;font-family:monospace;' +
        'line-height:1.6;pointer-events:none;white-space:pre';
      document.body.appendChild(dbg);
    }

    // Gather measurements from the DOM
    var mainRect = main.getBoundingClientRect();
    var dashLayout = main.parentElement;
    var rootEl = document.getElementById('root');
    var bodyRect = document.body.getBoundingClientRect();
    var negRoom = main.firstElementChild;
    var header = negRoom ? negRoom.firstElementChild : null;
    var headerRect = header ? header.getBoundingClientRect() : {};

    // Display all measurements
    dbg.textContent =
      'body.rect.top: ' + (bodyRect.top | 0) +
      '\nroot.rect.top: ' + (rootEl ? (rootEl.getBoundingClientRect().top | 0) : 'N/A') +
      '\ndashLayout.rect.top: ' + (dashLayout ? (dashLayout.getBoundingClientRect().top | 0) : 'N/A') +
      '\nmain.rect.top: ' + (mainRect.top | 0) +
      '\nheader.rect.top: ' + ((headerRect.top | 0) || 'N/A') +
      '\nheader.height: ' + ((headerRect.height | 0) || 'N/A') +
      '\nbody.scrollTop: ' + document.body.scrollTop +
      '\nhtml.scrollTop: ' + document.documentElement.scrollTop +
      '\nroot.scrollTop: ' + (rootEl ? rootEl.scrollTop : 'N/A') +
      '\ndashLayout.scrollTop: ' + (dashLayout ? dashLayout.scrollTop : 'N/A') +
      '\ndashLayout.overflow: ' + (dashLayout ? getComputedStyle(dashLayout).overflow : 'N/A') +
      '\nbody.overflow: ' + getComputedStyle(document.body).overflowY +
      '\nhtml.overflow: ' + getComputedStyle(document.documentElement).overflowY;
  }, 1000);
</script>
```

### How to Use This Technique for Future Debugging

1. **Add the script** to `index.html` before the closing `</body>` tag
2. **Adjust the URL filter** (`window.location.pathname.includes(...)`) to target the affected page
3. **Navigate to the page** — a red overlay box appears in the bottom-right corner
4. **Read the measurements** to understand:
   - `rect.top` — Where each element is positioned relative to the viewport (negative = above screen)
   - `scrollTop` — How far each element is scrolled (non-zero on `overflow:hidden` = bug)
   - `clientHeight` vs `scrollHeight` — Whether content overflows
   - `overflow` — Computed overflow style
5. **Screenshot the overlay** for analysis
6. **Remove the script** after debugging is complete

### Diagnostic Process (Step by Step)

| Step | What We Measured | Finding |
|------|-----------------|---------|
| 1 | `main.scrollTop`, `body.scrollTop`, `html.scrollTop` | All 0 — no standard scroll containers were scrolled |
| 2 | `main.clientH` vs `child.clientH` | Both 844px — NegotiationRoom fits inside `<main>` exactly |
| 3 | `child.offsetTop` | 16px — appeared to be offset, but was relative to `offsetParent` (misleading) |
| 4 | `main.rect.top` | **-53px** — `<main>` was above the viewport! |
| 5 | `dashLayout.scrollTop` | **69px** — The layout container was scrolled despite `overflow: hidden` |
| 6 | Applied scroll prevention listener | `dashLayout.scrollTop` dropped to 0, `main.rect.top` became 16px (correct) |

---

## Files Changed

| File | Change |
|------|--------|
| `src/Layout/DashBoardLayout.tsx` | Added `layoutRef` + scroll prevention listener + route change scroll reset |
| `src/components/chatbot/common/ConfirmDialog.tsx` | Changed `body.style.overflow = "unset"` → `""` (proper cleanup) |
| `src/components/BidAnalysis/SuccessModal.tsx` | Changed `body.style.overflow = 'unset'` → `''` (proper cleanup) |
| `src/components/chatbot/sidebar/AiReasoningModal.tsx` | Changed `body.style.overflow = "unset"` → `""` (proper cleanup) |

---

## Prevention

To prevent similar issues in the future:

1. **Never use `document.body.style.overflow = "unset"`** in modal cleanup — use `""` (empty string) to remove the inline style and let CSS take over
2. **Be cautious with `scrollIntoView()`** on elements inside `overflow: hidden` containers — it can scroll the container despite the CSS rule
3. **Use the diagnostic overlay technique** whenever layout displacement issues occur — it's faster than guessing at CSS fixes
4. **The DashBoardLayout scroll prevention listener** is a permanent guard — it will catch any future instances of programmatic scrolling on the layout container
