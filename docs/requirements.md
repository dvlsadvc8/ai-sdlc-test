# AI-SDLC Demo Website — Requirements

## 1. Project Overview

**Application name:** AI-SDLC Demo Website  
**Purpose:** Build a simple, publicly accessible webpage to validate the AI-SDLC framework's requirements analysis, architecture, planning, development, testing, and eventual Azure deployment workflows.

This is a deliberately small demonstration application, not a production business system.

## 2. Functional Requirements

### FR-01 — Page title
- Display **AI-SDLC Demo** as the main visible heading.
- Set the browser tab title to **AI-SDLC Demo**.

### FR-02 — Welcome message
- Display the message: **Welcome to the AI-SDLC demo website.**
- The message must be visible without user interaction when the page loads.

### FR-03 — Current date
- Display the current calendar date, based on the visitor's device-local date and time zone.
- Use a readable format such as **October 14, 2026**.
- Do not hardcode a date; refresh the displayed value on page load.

### FR-04 — Light/dark theme toggle
- Provide a clearly labeled control that switches the page between light and dark themes without reloading.
- The background, text, and controls must remain readable in both themes.
- Use the visitor's operating-system theme preference on first visit when available, defaulting to light otherwise.
- Remember a manually selected theme when the visitor reloads the page in the same browser.

### FR-05 — Responsive layout
- Support desktop and mobile screen sizes.
- The layout must remain readable and usable at viewport widths from **320 px** upward.
- Do not require horizontal scrolling for the primary page content.
- The theme toggle must remain accessible on narrow screens.

## 3. Nonfunctional Requirements

- **Accessibility:** Use semantic HTML, a single primary heading, keyboard-operable theme toggle, visible focus indicators, and sufficient contrast for both themes (target WCAG AA for normal text).
- **Performance:** Load quickly over an ordinary internet connection; do not add large libraries or assets without justification.
- **Security:** Do not request or store user accounts, credentials, or personal information. Do not embed API keys or secrets in client-side code.
- **Maintainability:** Use a straightforward project structure and readable, documented code where needed.
- **Hosting:** The completed application must be deployable to **Azure Static Web Apps**, preferably its Free plan.

## 4. Scope and Constraints

**In scope:** One static webpage, its styles and client-side behavior, automated verification, and the repository/configuration artifacts required by the AI-SDLC workflow.

**Out of scope:** User authentication, APIs, databases, Discord bots, paid services, analytics, contact forms, and complex navigation.

- Prefer plain HTML, CSS, and JavaScript unless the Architect identifies a concrete reason for a framework.
- Keep application source, tests, and architecture artifacts in the **application repository**, separate from the reusable AI-SDLC framework repository.
- Track project work through GitHub Issues and a GitHub Project, in accordance with the AI-SDLC framework's approval workflow.

## 5. Acceptance Criteria

1. The main heading reads **AI-SDLC Demo**, and the browser tab has the same title.
2. The welcome message appears on initial page load.
3. The date shown corresponds to the visitor's local calendar date when the page loads; it is not hardcoded.
4. Activating the theme toggle switches between light and dark appearances without navigation or reload.
5. The theme selection survives a page reload in the same browser.
6. Both themes maintain readable text and usable, keyboard-accessible controls.
7. At a viewport width of **320 px**, the heading, welcome message, date, and toggle are usable without horizontal page scrolling.
8. Automated tests or checks verify the date display and theme behavior; a responsive-layout check is also performed.
9. The completed page can be deployed to Azure Static Web Apps and accessed over HTTPS after deployment is approved.

## 6. AI-SDLC Milestone Boundary

This document is an **input to testing AI-SDLC Milestone 0.2 (Requirements and Project Planning)**.

For that test, AI-SDLC should analyze these requirements, create or select an authorized GitHub repository and Project, generate epics/stories and acceptance criteria, propose an architecture and first iteration, and request the required approvals.

**Do not implement, merge, provision Azure resources, or deploy this demo application during the Milestone 0.2 test.** Those activities belong to later milestones and require their respective approvals.
