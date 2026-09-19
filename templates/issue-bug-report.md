<!--
Install as .github/ISSUE_TEMPLATE/bug_report.yml (as a GitHub issue form) or
.github/ISSUE_TEMPLATE/bug_report.md in goldseats-web and goldseats-api.

A YAML issue-form version is at the bottom of this file.

NOT FOR SECURITY VULNERABILITIES. All three repos are public, so an issue is a
disclosure. Use the Security tab → Report a vulnerability, or email
security@goldseats.app. See SECURITY.md.
-->

---
name: Bug report
about: Something isn't working the way it should
labels: bug, needs-triage
---

## What happened

<!-- One or two sentences. -->

## What should have happened

## Steps to reproduce

1.
2.
3.

<!-- If you can reproduce it with curl, that's the most useful thing you can give us:

curl -s 'https://api.goldseats.app/v1/films/now-playing?language=fr' | jq .
-->

## Request ID

<!-- Every GoldSeats error response includes a `request_id`, and the web app shows it in
     error UI. Paste it here — it ties your report directly to our logs and usually
     saves an hour. -->

```
request_id:
```

## Environment

- **Where:** production / staging / local
- **URL:**
- **Browser and version:** <!-- for web issues -->
- **Device and OS:**
- **Screen width:** <!-- 375 / 768 / 1440 — layout bugs are usually width-specific -->
- **Signed in?** yes / no
- **API version:** <!-- curl -s https://api.goldseats.app/health/ready | jq .version -->

## Screenshots or recording

<!-- For UI bugs, near-essential. Include the browser console if there's an error. -->

## Frequency

- [ ] Every time
- [ ] Sometimes — roughly how often:
- [ ] Happened once
- [ ] Started recently — when:

## Impact

- [ ] **Blocking** — can't use GoldSeats at all
- [ ] **Major** — a core flow is broken (browse, film detail, recommendations, booking handoff)
- [ ] **Minor** — annoying, there's a workaround
- [ ] **Cosmetic**

---

## If this is about a seat recommendation

<!-- Delete if not. This section makes seat bugs actually diagnosable. -->

- **Theatre and auditorium:**
- **Showtime (with timezone):**
- **Party size:**
- **Preferences set:** aisle preferred / avoid front rows / accessible seating
- **Seats we recommended:**
- **What's wrong with them:**
- **Did the page say availability was simulated?** yes / no

<!-- Worth knowing: seat availability is currently SIMULATED, not live. If the bug is
     "these seats were actually taken", that's expected and disclosed — see
     legal/data-sources-policy.md. Still tell us if the disclosure wasn't visible. -->

## If this is about a seat map or layout being wrong

- **Theatre and auditorium:**
- **What's wrong:** row labels / seat numbers / aisle positions / total seat count / seat types
- **What it should be, and how you know:** <!-- a link to the theatre's seating chart is ideal -->

<!-- Our layouts are hand-entered from published seating charts and can be out of date or
     wrong. This is a genuinely valuable bug report and we want it. -->

## If this is about the 3D view

- **Device and GPU:**
- **Quality tier shown:** high / medium / low / none
- **Frame rate, if you can tell:**
- **Did it crash or reload the tab?**

## If this is about the chatbot

- **What you typed:**
- **What it replied:**
- **Did it name a seat that doesn't exist?** <!-- If yes, this is high priority. It should be
     structurally impossible — see architecture/adr/0004-self-hosted-llm.md -->
- **Conversation ID, if visible:**

## If this is an accessibility issue

- **Assistive technology and version:** <!-- VoiceOver, NVDA, keyboard only, zoom level -->
- **What you couldn't do:**
- **Where you got stuck:**

<!-- Accessibility bugs are treated as functional bugs, not enhancements. See
     engineering/accessibility.md. -->

---

## Anything else

<!-- Guesses about the cause are welcome. Related issues, recent changes you noticed. -->

---
---

<!-- ============================================================
     GitHub issue form version — .github/ISSUE_TEMPLATE/bug_report.yml
     ============================================================ -->

```yaml
name: Bug report
description: Something isn't working the way it should
labels: [bug, needs-triage]
body:
  - type: markdown
    attributes:
      value: |
        **Not for security vulnerabilities.** This repo is public, so an issue is a
        disclosure. Use the Security tab → Report a vulnerability, or email
        security@goldseats.app.

  - type: textarea
    id: what-happened
    attributes:
      label: What happened
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: What should have happened
    validations:
      required: true

  - type: textarea
    id: steps
    attributes:
      label: Steps to reproduce
      placeholder: |
        1.
        2.
        3.
    validations:
      required: true

  - type: input
    id: request-id
    attributes:
      label: Request ID
      description: From the error message or the API response. This ties your report to our logs.

  - type: dropdown
    id: environment
    attributes:
      label: Where
      options: [production, staging, local]
    validations:
      required: true

  - type: input
    id: url
    attributes:
      label: URL

  - type: input
    id: browser
    attributes:
      label: Browser / device / OS

  - type: dropdown
    id: frequency
    attributes:
      label: How often
      options: [Every time, Sometimes, Once, Started recently]
    validations:
      required: true

  - type: dropdown
    id: impact
    attributes:
      label: Impact
      options:
        - Blocking — can't use GoldSeats
        - Major — a core flow is broken
        - Minor — annoying, has a workaround
        - Cosmetic
    validations:
      required: true

  - type: dropdown
    id: area
    attributes:
      label: Area
      options:
        - Home page / filters
        - Film detail / timeline
        - Seat map / recommendations
        - Booking handoff
        - 3D seat view
        - Chatbot
        - Feed / follows / notifications
        - Auth
        - Seat layout is wrong for a real auditorium
        - Accessibility
        - Other
    validations:
      required: true

  - type: textarea
    id: area-detail
    attributes:
      label: Area-specific details
      description: |
        Seat recommendation → theatre, auditorium, showtime, party size, preferences, seats we suggested.
        Wrong layout → what's wrong and a link to the real seating chart.
        3D view → device, GPU, quality tier.
        Chatbot → what you typed, what it replied, whether it named a seat that doesn't exist.
        Accessibility → assistive technology and where you got stuck.

  - type: textarea
    id: screenshots
    attributes:
      label: Screenshots, recording, or console output

  - type: textarea
    id: anything-else
    attributes:
      label: Anything else
```
