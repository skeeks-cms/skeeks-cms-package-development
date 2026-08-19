# Entity search in queries

Use this reference for the free-text search users type into backend grids and
into the AJAX selects that pick companies, clients, contractors and similar
entities.

## One search contract per entity

`ActiveQuery::search($word)` is the single entry point. Grid filters, AJAX
selects (`AjaxSelectModel` and its `searchQuery` callbacks) and API lookups all
route through it, so anything a grid can find a select must find too.

Do not reimplement the field list inside a controller filter. A filter that
builds its own `LIKE` list silently diverges from the select on the same entity.
Call `$query->search($e->field->value)` and extend the query class instead.

`search()` returns the query untouched for an empty or whitespace-only string.
It never adds `LIKE '%%'`.

## Split the query into words

`CmsActiveQuery::searchWords()` is the shared tokenizer:

- a phone-looking string stays one token, because phone numbers are written
  with spaces;
- everything else is split on whitespace, capped at five words.

Each word becomes its own `OR` group over the searchable columns, and the groups
are combined with `AND`. `Ivanov Ivan` then matches a person whose first and
last name live in different columns, which a single `LIKE` cannot do.

`CmsActiveQuery::searchIdCondition()` adds an exact primary-key match for short
all-digit queries, so pasting a record ID works.

## Normalize phone numbers on both sides

`PhoneValidator` stores numbers in libphonenumber `INTERNATIONAL` format
(`+7 495 005-79-26`). Users type `84950057926`, `+7(495)0057926` or just the
last digits, so a raw `LIKE` on the stored column finds nothing.

`skeeks\cms\helpers\PhoneHelper` normalizes both sides:

- `isSearchablePhone()` — the string contains only digits and separators, and
  at least four digits;
- `searchDigits()` — digits only, with a leading country code or trunk `8`
  removed, so every way of writing one number yields the same significant part;
- `sqlDigits()` — nested `REPLACE` that strips separators from the column;
- `likeCondition()` — a contains-match for search, `null` when the string is
  not a phone;
- `equalCondition()` — a same-number match for lookups such as login by phone
  and matching an incoming telephony call; it refuses partial numbers.

Add the phone condition to the existing `OR` list. Never replace the plain
`LIKE` on the stored value: a user who pastes the exact stored format must keep
matching.

## Keep the join set proportional to the query

`search()` joins related tables, and every extra `LEFT JOIN` multiplies the row
product the `WHERE` list is evaluated against. Join a relation only when the
current query can actually hit it — for example, join the employees' phones and
emails of a company only when a word looks like a phone or contains `@`.

Aliased nested `joinWith('users.cmsUserPhones as usersPhones')` joins the
intermediate relation a second time without its alias. Join the leaf table
explicitly with `leftJoin()` against the alias already in the query.

Keep `groupBy` on the primary key whenever a to-many relation is joined.
