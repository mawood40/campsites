# UI / UX Documentation

## Scope

This document describes **member-visible and admin-visible** experiences for the Campsite Reviews Membership Platform. It implements PRD §4 (User Experience Summary) and informs SPA routing and component work. Visual design may use Radix Themes or a minimal custom layout per `.cursor/rules/tech-stack-node.mdc` — **consistency** (spacing, typography, focus states) matters more than a specific palette in v1.

## Global patterns

- **Unauthenticated entry:** Users land on **Login** with a link to **Sign Up**. Protected routes redirect here with a `next` query where helpful.
- **Authenticated shell:** Header with app title, current user display name, **Log out**, and role badge for admins. Main content area for routes below.
- **Feedback:** Inline field errors for forms; toast or banner for async success/failure; never rely on color alone for success/error (use icon + text).
- **Accessibility:** Keyboard navigable forms, visible focus, labels tied to inputs, sufficient contrast (WCAG 2.1 AA target for v1).

## Screens and flows

### Login

- Fields: email, password.
- Actions: submit, link to Sign Up.
- Errors: generic message for failed login per PRD (“safe” — do not reveal whether email exists).

### Sign Up

- Fields: email, password (with rules shown from config: min length, complexity), display name (or pseudonym per `auth.json`).
- Optional: “verification pending” state if email verification is enabled.
- Success: redirect to SPA home (campsite list) or “check your email” interstitial per config.

### Campsite list (home)

- Grid or list of **cards**: primary image, name, location, price snippet, short general info teaser, **review count**, **percent positive** (e.g. “75% positive · 12 reviews”).
- Primary action: **+ Add campsite** (FAB or toolbar).
- Card action: open **detail**; secondary **Reviews** affordance may go to detail’s review panel or dedicated reviews route.

### Campsite detail

- Shows full fields: name, location, pricing, general info, large primary image, aggregates.
- Actions: **Write review** (opens form or inline panel), **View all reviews** (scroll or navigate to reviews subview).
- **Back** to list preserves scroll position when feasible.

### Add campsite

- Form: name, location, pricing, general information (textarea), **image** file input with preview.
- Validation before submit (required fields, image type/size client-side mirror of server rules).
- Success: redirect to list with new campsite visible, or to detail of created campsite.

### Review submission

- Controls: thumbs up / thumbs down (mutually exclusive), comments (textarea, allow empty vs min length per policy in `app.json`).
- Submit sends single payload; on success, update displayed aggregates and disable duplicate review UI for this user/campsite (per one-review rule).

### Review inspection

- List or timeline: each row shows polarity, comments, author display name, timestamp; order **newest first** unless user changes sort (sort optional; default matches PRD).

### Admin — subscriber management

- Route gated to **admin** role (server-enforced; UI hides entry for subscribers).
- Table: email, display name, status, created date; action **Remove** or **Deactivate**.
- **Confirmation modal:** explicit copy (“This deactivates the account …”), require confirm checkbox or typed phrase; call API with `confirm: true`.
- Post-success: refresh list; show non-blocking success message.

## Responsive behavior

- **Mobile-first:** single column list; touch-friendly tap targets (min ~44px).
- Tables on small screens: card layout or horizontal scroll for admin list.

## Empty and loading states

- **Empty list:** Illustration or simple message + CTA to add first campsite.
- **Loading:** Skeleton cards or spinner on list; avoid layout shift when images load.

## Related documents

- [PRD.md](../PRD.md) §4, §6
- [API.md](API.md) for data shapes shown in UI
- [FRD_AuthenticationAndAuthorization.md](Features/FRD_AuthenticationAndAuthorization.md)
