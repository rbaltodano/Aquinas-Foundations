# Aquinas Design System: "The Scriptorium"

## Design Principles
- **Aesthetic: "Modern Antiquarian" / "Digital Scriptorium"** The app should feel like a tactile, high-quality theological manuscript brought into the digital age. It uses "Academic Minimalism"—low-contrast, warm earthy tones, and typography-driven hierarchies instead of modern, bubbly iOS paradigms.
- **Layout Architecture: The Infinite Canvas**
  Unlike traditional vertical chat feeds, the UI is a cartographic, node-based tree. Content is organized into discrete cards connected by thin, 1px horizontal and vertical structural lines, mimicking a scholar organizing index cards on a desk.
- **Component Restraint:**
  Avoid floating elements. Cards should be flat against the background, defined by thin borders and gentle corner radii rather than heavy drop shadows. The interface must step back and let the textual logic shine.

## Colors

`AquinasTheme.Colors` in `DesignSystem/SharedTypography.swift` is the implementation source of
truth. Important light/dark pairs:

- **Canvas:** `#F3EEE2` / `#120F0C`
- **Canvas Secondary:** `#FFFAF0` / `#24201C`
- **Primary Brown:** `#4A321C` / `#FFFAF0`
- **Heading Text:** `#614C40` / `#FFFAF0`
- **Paragraph Text:** primary brown at 75% opacity / warm cream at 75%
- **Placeholder Text:** primary brown at 50% / warm cream at 50%
- **Light Green:** `#86803E` / `#B7AE78`
- **Accent Red:** `#AF4949` in both modes
- **Borders:** dark brown at low opacity in light mode; warm cream at low opacity in dark mode

## Typography
We use a dual-font system. Libre Baskerville (Serif) is for academic/structural hierarchy, and Figtree (Sans-Serif) is for readable UI data and body copy.

- **Primary Font (Serif):** Libre Baskerville
  - **Title:** 24pt Regular; larger title tokens are 34pt, 36pt, and 40pt
  - **Heading:** 20pt Regular
  - **Quote:** 16pt Italic

- **Secondary Font (Sans-Serif):** Figtree
  - **UI Heading:** 18pt Bold
  - **UI Subheading / Body:** 14pt Bold / 14pt Regular
  - **Body Large:** 16pt Regular
  - **Inline Insight:** 16pt Bold in light green
  - **Labels / Chips:** 12pt Bold

## Spacing & Layout
- **Base Unit:** 8px
- **Border Radius:** 24px
- **Screen Padding:** 16px

## Components
- **Buttons:** Style: Outline or flat fill. No gradients.
  - Shape: Rounded rectangle with 24px corner radius.
  - Colors: Use `#AF4949` (Accent) for primary actions, `#4A321C` (Primary) for secondary.
- **Cards:** Use semantic canvas/component-background tokens, a quiet 1px semantic border, and
  restrained elevation.
- **Inputs (Text Fields / Chat Box):** Style: Minimalist, with the semantic side-menu/input border.
  - Padding: 24px internal padding.
  - Font: Figtree, 16pt.
- **Model Status:** Replaces the old Thinking toggle. Inactive copy is `Idle`; active copy is
  `Thinking` in light green with shimmer. When multiple jobs exist, prefix it with progress such
  as `1/3`. Only `Thinking` shimmers. The control is present in Branch and conversation Insight
  Tree modes, but disappears while an Insight is hovered so the contextual actions have priority.
- **Model Tasks popup:** A 345px-wide card with 28px internal padding, 24px corner radius, an 8px
  outer vertical gap, and a 4px task-list gap. The heading and completed checkmarks use heading
  text color. Current tasks use the context wheel spinner and a `stop.circle.fill`; upcoming tasks
  use a dotted circle and removable `xmark`. Task-state icons crossfade. The idle card stays full
  width rather than hugging its copy.
- **Popup motion:** Model Tasks and Insight hover cards enter with a spring-driven scale-up plus
  upward translation from below and leave with the inverse transition. Switching hovered Insights
  replaces the whole card so outgoing and incoming content animate independently. Opening Model
  Tasks also produces a light haptic tap.
- **Queued state:** Queued definitions and the `Question queued` response state use the shared
  breathing animation. Cancelling the associated queued/current task must stop the animation.
- **Thinking response state:** The initial `Thinking` placeholder is bold and centered. Once the
  model supplies its user-facing approach summary, that text uses response line height, leading
  alignment, regular weight, and the same shimmer treatment.
- **Context gauge:** The control dock retains the wheel as a static usage gauge. It has no visible
  `Context` label and does not spin while the model is loading; tapping it opens the context card.
- **Insight Tree update affordance:** When the tree control reads `Updated`, its entry and exit
  text animations include a 5% scale-up-and-down pulse.

## Guardrails (Do's and Don'ts)
- NEVER use standard Apple iOS colors like `.blue` or `.red`. Use `AquinasTheme.Colors`; do not
  repeat raw hex values in feature code.
- DO NOT use generic `Text()` views without applying the semantic fonts from `SharedTypography.swift`.
- Keep layouts structured and hierarchical; this is a tool for logic, not a playful social media app.
- Keep model activity and context capacity visually distinct: shimmer communicates active model
  work; the context wheel communicates usage only.
