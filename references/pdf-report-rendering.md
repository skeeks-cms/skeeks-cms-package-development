# HTML and PDF report rendering

Use this reference for reusable HTML reports and PDFs rendered through a browser
engine such as Chromium. Apply the same data and presentation contract to both
outputs.

## Treat template settings as authoritative

- Carry orientation, theme, cover, footer, page numbers, colors and margins from
  the form into the saved model, generated HTML/CSS and renderer options.
- Use portrait only when orientation is unset. Never override an explicit
  landscape selection.
- Enable background printing in the PDF renderer (`printBackground: true`).
- Reserve enough bottom page margin for the footer and a visible gap above it.
- Derive cover, body and footer widths from one content-width contract.

## Respect selected report columns

- Use the selected columns as the source of truth for the cover, task metadata
  and totals.
- When `fact_time` is not selected, omit worked time everywhere, including the
  title page and summary cards.
- Do not emit duplicate time metrics under different labels.
- Keep HTML preview and PDF output semantically identical.

## Seed defaults safely

- Create or update default templates with idempotent migrations.
- Resolve templates by a stable semantic key or documented attributes. Never
  depend on an incidental numeric row ID.
- Preserve user-customized templates when upgrading an existing database.
- Verify both a fresh database and an upgraded database.

## Render task results as structured content

- Render every result comment separately and in source order.
- Sanitize stored HTML, but preserve useful structure such as `p`, `br`, `ul`,
  `ol`, `li`, `a`, `blockquote`, `pre`, `code` and simple tables.
- Preserve explicit line breaks and list items instead of flattening them into
  one paragraph.
- Add clear spacing between the task title, metadata, each result and its
  attachments.

## Let long content paginate naturally

- Keep the normal font size. Never shrink a long comment merely to fit one page.
- Avoid fixed heights, whole-card scaling, clipping and overflow hiding.
- Do not apply `break-inside: avoid` to an unbounded task or comment container.
- Allow paragraphs, lists and other semantic fragments to flow onto the next
  page.

## Design for print fragmentation

Chromium may not reproduce a spanning parent's background, border or padding on
every page fragment.

- Put visual treatment on bounded fragments such as the task header, each
  result and each attachment group.
- Give every continuation-capable result its own background, padding and accent
  rule.
- Avoid depending on one outer card background across multiple pages.
- Verify the task following a split result; it must keep its background,
  padding, border and spacing.

## Render attachments with their result

- Associate attachments with the exact result comment that owns them.
- Render available images inline with constrained dimensions and preserved
  aspect ratio.
- Render non-image files with filename, type or size when available, and a
  usable link.
- For a missing image, show a neutral unavailable-file placeholder instead of a
  broken image icon. Keep the filename visible.
- Apply the same attachment rules in HTML and PDF outputs.

## Keep light and dark themes explicit

- Define semantic colors for page background, task surface, result surface,
  border, primary text, secondary text and accent.
- Keep page, task and result surfaces visibly distinct in both themes.
- Test printed backgrounds instead of relying on the browser preview alone.

## Verify the complete matrix

Test at least:

- portrait and landscape;
- dark and light themes;
- cover, footer and page-number toggles;
- worked time selected and unselected;
- short, multiple and very long results;
- paragraphs, lists, links and preformatted text;
- valid images, non-image files and missing attachments;
- content split across pages followed by more tasks;
- fresh and upgraded databases.

Inspect the generated PDF itself for page dimensions, backgrounds, footer
placement, numbering and continuation pages.

## Treat asset watch as a separate process

- Do not assume a Compose stack starts an asset watcher.
- Run the package's watch command separately only when editing compiled source
  assets.
- Do not make directly loaded server-rendered CSS depend on a watcher.
- Confirm whether the runtime consumes source files or built artifacts before
  debugging stale styles.
