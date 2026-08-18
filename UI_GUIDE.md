# UI architecture

The interface was rebuilt from scratch on a token-driven design system.
This is the map

## Layers

```
ui/tokens.py        colour / type / spacing / radius / motion / breakpoints
ui/animation.py     tween engine on Tk's after-loop; easings; reduced motion
ui/surfaces.py      gradients, glass, shadows, avatars (Pillow-rendered)
ui/icons.py         62 vector icons drawn as paths, tinted per theme
ui/theme.py         appearance mode, cache invalidation, repaint notifications
ui/a11y.py          keyboard focus, accessible names, contrast auditing
ui/components/      buttons, cards, inputs, charts, table, toast, skeleton…
ui/sidebar.py       collapsible navigation
ui/shell.py         app shell: sidebar + top bar + router + Page base class

app/controllers.py  UI↔service seam; all async work marshals back via after()
app/analytics.py    every dashboard/report number, derived from real history

gui/auth.py         shared auth chrome (split layout, brand panel)
gui/login.py        sign-in
gui/register.py     sign-up
gui/home.py         main window; owns controllers, declares routes
gui/chatbot.py      floating AI assistant
gui/screens/        dashboard, translate, history, reports, profile, settings
```

Rules: screens never import from `modules/`, never hard-code a hex value
or pixel font size, and never construct a bare CTk widget where a
component exists.

## Working with Tk's constraints

Four constraints shaped most of the implementation. They are worth
knowing before editing anything visual.

**Widgets are opaque.** There is no transparency. A label placed over a
gradient paints a solid rectangle of its inherited background behind its
text. Anything sitting on a gradient is therefore drawn as a *canvas
item* via `ui.surfaces.GradientCanvas` — see `GradientButton`, `FAB`,
`GradientHeaderCard`, the sidebar mark, the chatbot header and the auth
brand panel.

**`fg_color="transparent"` inherits, it does not composite.** A
transparent child resolves to its parent's effective colour and paints
that. `GlassFrame` therefore sets a flat approximation of the frosted
colour (`flat_glass_color`) so its children blend into the panel while
the real blur still shows in the padding between them.

**An empty `CTkFrame` is 200×200.** Frames shrink-wrap packed children,
but an *empty* one keeps the library default and silently inflates its
section. Placeholder slots pass `width=0, height=0`. This is also why
badges and chips are built on `CTkLabel`, not `CTkFrame`.

**`_draw` is taken.** `CTkFrame` calls its own `_draw(no_color_updates=…)`
internally, so a subclass must not define a method with that name.

## Threading rules

Tk is single-threaded. Two rules, both learned the hard way:

**1. Never open a Tk dialog off the main thread.** `askopenfilename`
and friends must be called from the thread running the event loop.
Calling one from a worker deadlocks the interpreter's Tcl lock and the
entire window stops responding — every button appears dead. Controllers
therefore split the picker from the work: `pick_image()` /
`pick_pdf()` / `pick_export_path()` run on the UI thread and return a
path; `extract_from_image(path)` / `translate_pdf(path, …)` /
`export_to(path, …)` do the slow part inside `run_async`.

**2. Never marshal results back with `after()` from a worker.**
Scheduling from another thread can be dropped silently. `run_async`
hands the result over through a `queue.Queue` polled from the main
thread instead.

`tests/`-adjacent harnesses assert rule 1 by monkeypatching the file
dialogs to record the calling thread.

## Motion

Every transition runs through `ui.animation.Animator`, keyed per
property so a rapid hover in/out cancels cleanly. Tweens self-cancel
when their widget is destroyed. Setting
`ui.animation.prefers_reduced_motion = True` (Settings → Accessibility)
collapses every tween to its final value.

## Charts

Canvas-drawn, no plotting dependency: line, area, bar (vertical and
horizontal), donut, radial gauge, sparkline and an activity heat strip.
They animate from zero, re-render on resize, show hover readouts, and
follow the theme — a matplotlib bitmap would do none of that.

Canvas widgets cannot use CTk's `(light, dark)` colour pairs, so they
resolve colours through `tokens.pick()` at draw time and repaint on
theme change.

## Accessibility

- Contrast is enforced by tests, not by eye. `tests/test_design_system.py`
  checks every text/surface and status/soft-background pairing against
  WCAG AA **in both modes**. Several palette values are deliberately
  darker than the obvious choice because the obvious choice failed.
- Focus attaches to the inner Tk canvas of each CTk composite
  (`ui/a11y.py`), so Tab/Shift-Tab traversal and Enter/Space activation
  work. Focused controls draw a 2px ring.
- Icon-only controls register an accessible name, surfaced as a tooltip.
- Status is never colour-only: tones pair with an icon or a text label.

## Tests

```
python -m unittest discover -s tests
```

57 tests: analytics correctness (including empty accounts, NULL columns
and unparseable timestamps) and design-system guardrails (contrast,
scale monotonicity, breakpoints, icon coverage).

## Notes on data

Dashboard, reports and profile compute every figure from the signed-in
user's real translation history. An account with no history shows zeros
and empty states. The "Activity score" is a product metric with
published weights (volume, consistency, breadth, depth), shown with its
breakdown — not an objective rating.
