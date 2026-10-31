# Date and time field

Every date or date-and-time field of Admin, UPA, future cabinets and site forms
uses the platform field
`skeeks\cms\widgets\formInputs\datetime\DateTimeInputWidget` (`skeeks/cms`).
Do not add new kartik `DateControl`, `DatePicker`, `DateTimePicker` or
`DateRangePicker` fields, jQuery UI pickers or another third-party picker; the
existing kartik fields are migrated gradually (decision of 2026-10-31).

## What the field is

- A button with the value ("Сегодня, 15:30", "Завтра", "пн, 3 нояб., 10:00";
  the year is added when it is not the current one) and × to clear.
- A window: the month grid (Monday first, today outlined, weekends muted,
  days of other months faded), quick choices on the right ("Сегодня",
  "Завтра", "В конце недели" = this or the next Friday, "Через неделю",
  "В конце месяца", each with its day) and, for date and time, the time:
  hours 00–23 and minutes by `minuteStep`.
- Date mode: a day click or a quick choice applies and closes. Date and time
  mode: a day goes into a draft (keeping the chosen time, or `defaultTime`,
  but not earlier than now for today and not earlier than `min`); a quick
  choice applies at once; "Готово", Enter on the selected day, Ctrl+Enter
  or a click outside apply; Esc cancels.
- Keyboard: arrows by day/week, Home/End — the week, PageUp/PageDown — the
  month (Shift — the year), arrows in the time grids, Tab stays inside the
  window, ArrowDown on the button opens it. The window is `role="dialog"`,
  days are a `role="grid"` with roving focus and full Russian labels.
- Below 576 px the window is a sheet at the bottom of the screen at full
  width with a backdrop; the quick choices become a scrolling row of chips.
- Colors come only from backend semantic variables (`--sx-form-control-*`,
  `--sx-calendar-*`, `--sx-color-*`) with fallbacks, so light and dark modes
  and palettes follow the shared theme and site pages still work.

## Configuration (close to DateControl)

A form migrates by changing the class; `type => DateControl::FORMAT_DATE`
works unchanged because the type values are the same strings.

| Option | Default | Meaning |
|---|---|---|
| `type` | `datetime` | `date`, `datetime`; `range` is reserved (throws `NotSupportedException`) |
| `displayFormat` | `human` | or a PHP pattern (`php:d.m.Y H:i`), tokens `d j m n Y y H G i s D M` (`D`, `M` — Russian short names) |
| `saveFormat` | `php:U` | `php:U`, `php:Y-m-d`, `php:Y-m-d H:i:s` or another pattern of the same tokens |
| `options` | | attributes of the button (`form-control` is removed) |
| `saveOptions` | | attributes of the hidden input |
| `minuteStep` | `5` | also read from `widgetOptions.pluginOptions.minuteStep` |
| `min`, `max` | null | unix time or a `strtotime()` string in the time zone (`today`, `now`) |
| `allowClear`, `placeholder` | true, by type | "Очистить" in the window and × on the button |
| `quickOptions` | five built-ins | keys `today tomorrow weekEnd nextWeek monthEnd`, `key => label`, `['label' => …, 'days' => N]`; `false` — none |
| `defaultTime` | `09:00` | time of a day chosen without one |
| `timeZone` (`displayTimezone`) | formatter time zone | wall time of the field |
| `disabled`, `readonly` | false | the button is disabled |

Values: `U` is converted between the time zone and unix time (Intl in the
browser, `DateTimeImmutable` on the server). String formats keep the wall
time of `timeZone`. In date mode the saved value is midnight of the day. The
server parses the current value (unix time, the save pattern, then
`strtotime()`) and writes it back in the save format; an unchanged value is
not rounded.

## Markup and JS contract

```
div.sx-dt[data-sx-datetime='{json}']#<id>-widget
  input[type=hidden].sx-dt__value#<id> name=…      — the posted value
  input[type=text][hidden].sx-dt__display#<id>-disp — the shown text (DateControl's display input)
  button.sx-dt__trigger#<id>-btn > svg + span.sx-dt__label
  button.sx-dt__clear
```

- The hidden input keeps the field id and name: ActiveForm validation,
  `WidgetField`, filters and old selectors such as `#model-attr` work.
- A change fires jQuery `change` on `.sx-dt__value` and then on
  `.sx-dt__display`. Old page scripts listening to
  `.field-… input[type='text']` (the DateControl display input) keep working.
- Setting `.sx-dt__value` from outside and firing `change` updates the
  button; emptying `.sx-dt__display` with `change` clears the value.
- Everything is delegated from `document` and the window is built on open
  and appended to `body` (or to the closest `.modal`, so Bootstrap focus
  enforcement keeps working), `position: fixed`. Fields loaded by PJAX, into
  drawers, iframes or modals need no initialization; `pjax:send` closes it.
- `sx.DateTimeInput.open(root)`, `close()`, `setValue(root, savedValue)`.
- `DateTimeInputAsset` (CSS + JS, depends only on jQuery) is registered by
  the widget, never globally.

## Migrating a kartik field

1. Replace the class; drop `displayFormat` to get the human text, keep
   `saveFormat` (the module default was already `php:U`).
2. Check page scripts that read or clear the display input or call the
   kartik plugin (`.kvDatepicker`, `.datetimepicker`); the mirror and the
   `change` contract cover reading and clearing, plugin calls must be
   rewritten.
3. A package that may run with an older `skeeks/cms` uses
   `class_exists(DateTimeInputWidget::class) ? DateTimeInputWidget::class : DateControl::class`
   until its `skeeks/cms` constraint requires the field.
4. Verify the create and update forms (an empty and a saved value), saving,
   both themes and 375 px.

Reference consumers: the task start (`AdminCmsTaskController`,
`plan_start_at`), the document date (`AdminCmsDocumentController`), the
proposal validity (`cms-proposal`) and the agent next run (`cms-agent`).
The remaining kartik places are listed in the platform plan
(ROADMAP, «Кабинеты»).
