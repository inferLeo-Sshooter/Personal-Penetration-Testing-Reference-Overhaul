# Access Control Testing Checklist

A practical checklist derived from 16.1 - Access control vulns and priv esc notes. Use during testing to make sure each access control bypass class has been covered.

## Table of Contents

- [1. Unprotected functionality / unpredictable URLs](#1-unprotected-functionality--unpredictable-urls)
- [2. Parameter-based access control](#2-parameter-based-access-control)
- [3. Platform misconfiguration](#3-platform-misconfiguration)
- [4. URL-matching discrepancies](#4-url-matching-discrepancies)
- [5. Horizontal privilege escalation](#5-horizontal-privilege-escalation)
- [6. Horizontal to vertical escalation](#6-horizontal-to-vertical-escalation)
- [7. Multi-step process flaws](#7-multi-step-process-flaws)
- [8. Referer-based access control](#8-referer-based-access-control)
- [9. General / cross-cutting](#9-general--cross-cutting)

---

## 1. Unprotected functionality / unpredictable URLs

- [ ] Try accessing admin/privileged paths directly (e.g. `/admin`, `/administrator-panel-xyz`) without going through the UI links
- [ ] Check `robots.txt`, sitemap.xml, JS bundles, and source comments for hidden paths
- [ ] Brute-force common admin/hidden paths with a wordlist (ffuf, dirsearch, gobuster)
- [ ] Inspect client-side JS for hardcoded sensitive URLs or role logic (e.g. `isAdmin` checks that still ship the link/endpoint to all users)

## 2. Parameter-based access control

- [ ] Look for role/permission values in hidden fields, cookies, or query strings (`admin=true`, `role=1`)
- [ ] Tamper with these values and resend the request to see if privilege changes

## 3. Platform misconfiguration

- [ ] Test alternate HTTP methods on restricted endpoints (if `POST` is blocked, try `GET`, `PUT`, etc.)
- [ ] Send `X-Original-URL` / `X-Rewrite-URL` headers pointing to restricted endpoints to see if backend honors them over the routed URL

## 4. URL-matching discrepancies

- [ ] Test case variations on restricted paths (`/ADMIN/deleteUser` vs `/admin/deleteUser`)
- [ ] Append junk extensions to restricted endpoints (`/admin/deleteUser.json`, `.anything`) — relevant for older Spring apps with `useSuffixPatternMatch`
- [ ] Test with/without trailing slash on restricted endpoints

## 5. Horizontal privilege escalation

- [ ] Change ID/reference parameters (`?id=123` → `?id=124`) to access another user's resource
- [ ] Search the app for leaked GUIDs/identifiers belonging to other users (in messages, reviews, public profiles)
- [ ] Check redirect responses for unauthorized requests — confirm they don't leak the target's data in the body even if redirecting to login

## 6. Horizontal to vertical escalation

- [ ] When horizontal IDOR works, try IDs likely to belong to admins/privileged accounts and check what access that grants

## 7. Multi-step process flaws

- [ ] Map the full multi-step workflow, then test whether later steps can be reached directly, skipping earlier authorization checks
- [ ] Try submitting final-step parameters without completing prior steps

## 8. Referer-based access control

- [ ] Identify any restricted sub-pages and test access while supplying a forged/spoofed `Referer` header pointing to an authorized page
- [ ] Try without any `Referer` header at all

## 9. General / cross-cutting

- [ ] Test access as each role (unauthenticated, low-priv, high-priv) against every sensitive endpoint
- [ ] Re-test access control after any app update/route change — discrepancies are often introduced incidentally
- [ ] Check both UI-visible and API-only endpoints (mobile app traffic, internal APIs) for the same flaws
