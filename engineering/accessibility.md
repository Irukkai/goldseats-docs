# Accessibility

**Target: WCAG 2.2 Level AA**, across every page of `goldseats-web`.

This isn't compliance theatre. GoldSeats exists to answer "which seat should I sit in",
and the users who most need a considered answer to that question — people with low vision,
people who need accessible seating, people who can't navigate a drag-and-drop seat picker —
are exactly the users a badly built seat map excludes. A seat map that only works with a
mouse and only communicates quality through colour has failed at the product's core job.

Also relevant: GoldSeats operates in Canada, where the AODA applies to Ontario
organisations and WCAG 2.0 AA is the legislated standard. AA on 2.2 clears that.

---

## Non-negotiables

Every one of these is a merge blocker, checked in review per
[`code-review.md`](code-review.md):

1. **Keyboard operable.** Every interaction reachable and completable with a keyboard
   alone. No exceptions, including the seat map and the 3D view.
2. **Visible focus.** A clearly visible focus indicator on every focusable element, with at
   least 3:1 contrast against the adjacent background. Never `outline: none` without a
   replacement.
3. **No information by colour alone.** Every colour-coded state also has text, an icon, or
   a shape. This is WCAG 1.4.1 and it's the one we're structurally most at risk of
   breaking.
4. **Contrast.** 4.5:1 for body text, 3:1 for large text and UI component boundaries.
5. **Accessible names.** Every control has a name that says what it does, exposed to
   assistive tech.
6. **Semantic HTML first.** A `<button>` before a `div` with a click handler. ARIA is a
   patch for when HTML can't express it, not a first resort.
7. **Reduced motion respected.** `prefers-reduced-motion` disables the animated seat-grid
   hero, the score pulse, and camera transitions in the 3D view.
8. **Zoom to 200%** without loss of content or function. Reflow at 320px width.

---

## The gold-on-dark palette

Our brand is `#FFD700` gold on `#0a0a0f` near-black, inherited from the landing page.
That combination is contrast-generous for text and contrast-*hostile* for the thing we
most want to highlight.

Measured against `ink-950` (`#0a0a0f`):

| Foreground | Ratio | Verdict |
| --- | --- | --- |
| `gold-500` `#FFD700` | 14.6:1 | ✅ Excellent for text and for large UI |
| `ink-200` `#c9c9d4` | 11.2:1 | ✅ Body text |
| Mid-grey `#6b6b7b` | 3.4:1 | ⚠️ Large text and borders only, never body copy |
| Dim grey `#3a3a47` | 1.6:1 | ❌ Decorative only |

So gold-on-black text is fine. The problem is elsewhere: **gold-vs-grey as the only signal
distinguishing a recommended seat from an ordinary one.** Gold and grey against the same
dark background are perfectly legible individually and can be indistinguishable to someone
with a colour vision deficiency, and `gold-500` against `ink-800` seat fill is only about
4:1 — passable as a boundary, not as a way to encode meaning.

Rules that follow:

- Never `gold-500` text on a light background. Our product surfaces are dark; the gold is
  for dark contexts only.
- Never rely on the gold halo alone to mean "recommended". See the seat map section.
- `ink-200` is the minimum for body text. `ink-400` and below are decorative.
- Disabled controls still need 3:1 against their background. "Sold out" must be readable,
  not just dimmed into the void.

---

## The 2D seat map

The hardest accessibility problem in the product ([M5](../product/milestones/M5.md)), and
the place most seat pickers on the internet give up. Ours doesn't.

### Structure

A seat map is a grid. Model it as one, so assistive tech announces position:

```tsx
<div role="grid" aria-label="Auditorium 3 seat map. 14 rows, 20 seats per row.">
  {rows.map((row) => (
    <div role="row" key={row.label} aria-label={`Row ${row.label}`}>
      {row.seats.map((seat) => (
        <button
          role="gridcell"
          key={seat.id}
          aria-label={seatLabel(seat)}
          aria-pressed={seat.id === selectedId}
          disabled={!seat.isAvailable}
          tabIndex={seat.id === focusedId ? 0 : -1}
        >
          <SeatGlyph seat={seat} />
        </button>
      ))}
    </div>
  ))}
</div>
```

### The accessible name is the whole feature

Everything a sighted user gets from position, colour, and the score badge has to be in the
name:

```ts
function seatLabel(seat: SeatView): string {
  const parts = [`Row ${seat.rowLabel}, seat ${seat.seatNumber}`];

  if (!seat.isAvailable) parts.push('taken');
  else if (seat.recommendationRank === 1) parts.push(`recommended, best seat, score ${seat.score} of 100`);
  else if (seat.recommendationRank) parts.push(`recommended, option ${seat.recommendationRank}, score ${seat.score} of 100`);
  else if (seat.score != null) parts.push(`available, score ${seat.score} of 100`);
  else parts.push('available');

  if (seat.seatType === 'wheelchair') parts.push('wheelchair space');
  if (seat.seatType === 'companion') parts.push('companion seat');
  if (seat.seatType === 'recliner') parts.push('recliner');
  if (seat.isAisle) parts.push('aisle');

  return parts.join(', ');
}
```

"Row H, seat 12, recommended, best seat, score 98 of 100, aisle" is a complete answer
delivered through one channel. That's the bar.

### Keyboard model

One tab stop for the whole grid (roving `tabIndex`), then arrows to move within it. Tabbing
through 280 seats would be unusable, which is why the naive implementation is also the
wrong one.

| Key | Action |
| --- | --- |
| `Tab` | Enter/leave the grid. Focus lands on the top recommendation, not seat A1. |
| `←` `→` | Previous/next seat in the row, skipping aisle gaps |
| `↑` `↓` | Same seat position in the adjacent row |
| `Home` / `End` | First/last seat in the row |
| `PageUp` / `PageDown` | First/last row |
| `Enter` / `Space` | Select the focused seat |
| `Esc` | Clear selection |
| `G` | Jump to the next golden (recommended) seat |

That last one isn't in any spec. It's there because "take me to the good seats" is the
single most common intent, and making it one keystroke for keyboard users is strictly
better than what mouse users get.

### Announcing changes

```tsx
<div aria-live="polite" className="sr-only">
  {announcement}
</div>
```

Announce: recommendations arriving ("3 seat options found. Best: Row H, seats 12 and 13,
score 98."), selection ("Row H, seats 12 and 13 selected."), and failures ("No 4 seats
available together in this auditorium. Try a different showtime.").

`polite`, not `assertive`. Seat selection is never urgent enough to interrupt.

### Never colour alone

Each seat state carries at least two signals:

| State | Colour | Shape / glyph | Text |
| --- | --- | --- | --- |
| Recommended (rank 1) | `gold-500` fill + halo | Filled rounded square with a star | Score in the accessible name, badge visible on hover/focus |
| Recommended (2–3) | `gold-400` outline | Outlined with a star | Rank and score in the name |
| Available | `ink-200` outline | Outlined rounded square | "available" |
| Taken | `ink-800` fill | Filled with a diagonal strike | "taken" + `disabled` |
| Wheelchair space | `ink-200` | Wheelchair glyph | "wheelchair space" |
| Selected | `gold-500` + ring | Ring | `aria-pressed="true"` |

A visible legend maps every glyph to its meaning, and it's on the page, not in a tooltip.

### Touch targets

Minimum 24×24 CSS pixels (WCAG 2.5.8 AA), and we target 44×44 on touch devices. On a
phone, a 20-seat row doesn't fit at 44px — so the mobile seat map pans and zooms rather
than shrinking seats below the target size. Pinch-zoom is never disabled.

---

## Accessible seating is a product feature, not a legend entry

`seats.seat_type` already distinguishes `wheelchair` and `companion`. Make it useful:

- `accessible_seating_required` in the recommendation request filters to wheelchair spaces
  and their companion seats, as a contiguous pair.
- The scoring engine still scores them honestly. An accessible space with a poor view gets
  a low score — telling someone their only option is great when it isn't is worse than
  useless.
- Wheelchair spaces are visually and textually labelled at all times, not only when the
  filter is on.
- When no accessible seating is available for a showtime, say that plainly and suggest
  another showtime. Don't return an empty map with no explanation.

---

## The 3D seat view

[M6](../product/milestones/M6.md) renders a WebGL scene. WebGL is opaque to assistive
technology, so the scene is an *enhancement* and never the only path:

- Every seat is selectable and fully described in the 2D map without the 3D view existing.
  The 2D map is the accessible baseline and always available.
- The `<canvas>` has an `aria-label` describing what it shows: "3D preview of the view from
  Row H, seat 12. The screen fills 36 degrees of your horizontal field of view."
- Alongside the canvas, a text panel gives the same information in words: distance to
  screen, horizontal viewing angle, vertical angle, and how that compares to the
  THX-recommended 36 degrees. For many users that text is *more* useful than the render.
- Camera controls are keyboard operable: arrows to look around, `Esc` to reset.
- `prefers-reduced-motion` disables automatic camera movement and transitions.
- A visible "Skip 3D preview" control, and the 3D view never auto-focuses.
- If WebGL is unavailable or the device fails the capability check, we show the text panel
  and a static image. Never a broken canvas.

---

## The chatbot

[M7](../product/milestones/M7.md) — a conversational interface is often *more* accessible
than a seat grid, and it's easy to accidentally make it less:

- The message log is `role="log"` with `aria-live="polite"`. New messages are announced
  once, when complete — not token by token as they stream, which is unusable.
- Streaming text goes into a visually rendered region that is `aria-hidden` while
  streaming; the finished message is what gets announced.
- Recommended seats in a reply are real links/buttons into the seat map, not just prose.
- The input is a labelled `<textarea>`; `Enter` sends, `Shift+Enter` newlines, and that's
  documented in visible hint text.
- Loading state is announced ("Finding seats…"), not just a spinner.
- Tool-call traces are developer UI and are hidden from assistive tech entirely.

---

## Forms

- Every input has a visible `<label>`. Placeholder text is never a label — it vanishes on
  focus and fails contrast.
- Errors are associated with `aria-describedby` and `aria-invalid`, appear in text next to
  the field, and are summarised at the top of a long form.
- The home page filters (title, release date, language) are a `<form>` with a real submit,
  so it works without JavaScript and announces result changes via a live region.
- The language selector is a native `<select>`. A custom combobox here would be strictly
  worse for everyone.
- Required fields marked in text, not only with an asterisk.

---

## Content

- One `<h1>` per page, headings in order, no levels skipped for styling.
- Link text describes the destination. "Showtimes for Dune: Part Three", not "click here".
- `lang` on `<html>`, and `lang` on any element in a different language — relevant for us,
  since the catalog is bilingual and a French film title inside an English page needs
  `lang="fr"` or a screen reader mispronounces it.
- Images have `alt`. Posters: `alt="Poster for Dune: Part Three"`. Purely decorative
  images: `alt=""`.
- "Skip to main content" as the first focusable element on every page.
- Times shown with both a readable form and a `<time datetime>` machine value.

---

## Testing

Automated tooling catches roughly a third of real problems, so both layers are required —
see [`testing-strategy.md`](testing-strategy.md).

**Automated, in CI:**

- `eslint-plugin-jsx-a11y`, errors not warnings
- `@axe-core/playwright` on the home page, film detail, seat map, and chat. Any violation
  fails the build.
- Lighthouse accessibility score ≥ 95 on every page.

**Manual, before each milestone closes:**

- Unplug the mouse. Complete the whole flow: browse → filter → film → showtime →
  recommendation → booking handoff.
- VoiceOver on macOS and Safari, over the seat map specifically.
- Zoom to 200% and to 400%, check reflow at 320px.
- A colour-blindness simulator (deuteranopia and protanopia) over the seat map. If you
  can't tell recommended from available, the second signal is missing.
- `prefers-reduced-motion: reduce` enabled at the OS level.

---

## Statement of non-compliance

If we ship something that doesn't meet AA, it gets written down here with a date and an
issue — not quietly shipped. Currently: nothing outstanding. The 3D view's WebGL canvas is
inherently non-semantic, which is why the text panel and 2D map are specified as equal
paths rather than fallbacks.
