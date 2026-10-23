# Read API of CRM clients for AI clients (MCP/REST)

Verified on skeeks.com, 24.10.2026 (`skeeks/cms-mcp`, `skeeks/cms-communication`,
`skeeks/cms-proposal`). Executable sources and `tests/crm-read.php`,
`tests/mcp-read.php` of these packages are the final truth.

## Ownership

- `cms-mcp` owns the transport, the client read model (`src/crm`,
  `services\CrmReadService`) and the extension points. It knows no mail,
  proposal or other package model.
- A package that owns data (letters, proposals, consents) owns its read tools,
  the description of its feed entries and its overview section, in its own
  `src/mcp` namespace. Those classes may implement cms-mcp interfaces: only
  cms-mcp loads them.
- Never register a package provider in a `cmsMcp` component block of the
  package config: a component config without `class` breaks every site that
  does not install cms-mcp. Use the params extension below.

## Extension: `Yii::$app->params['cmsMcp']`

```php
'params' => ['cmsMcp' => [
    'toolProviders' => ['<package>' => ReadToolProvider::class],              // McpToolProviderInterface
    'historyLogDescribers' => ['<log type>' => HistoryDescriber::class],      // crm\HistoryLogDescriberInterface
    'historySources' => ['<name>' => HistorySource::class],                   // crm\HistorySourceInterface
    'overviewContributors' => ['<name>' => OverviewContributor::class],       // crm\OverviewContributorInterface
    'personalLinkPatterns' => ['<name>' => '~/\~kp/~'],                       // regex of a client's personal link
]],
```

`McpComponent::extensions($name)` merges its own property with these entries;
string keys replace, `null` switches an entry off (a site can do it in its
params). `extensionObjects()` instantiates and checks the interface once.

## The client history is the card feed

- A client is a lead, a company or a client person (`crm\CrmRecord`), resolved
  only through `forManager()` of the OAuth employee — a worker never reaches
  another manager's lead, even by id.
- `cms_crm_history_get` = the card feed (`CmsLogQuery::activity()` of the
  record: exactly the card's entries) + records of the card blocks (tasks,
  deals, bills, payments) + package sources (opt-outs, consents). Do not build
  AI history from raw `cms_log`: insert/update entries are the change log
  (`cms_company_timeline_get`), not communication.
- A feed type of a package is described by that package with the same access
  rule as its feed renderer (`Skeeks::$logRenderers`): an entry whose record
  the employee may not see is returned with `access: false` and no content.
- Sources return their newest `fetchLimit` (= offset + limit) entries and a
  count; the history merges and sorts them. Keep `offset + limit` bounded
  (1000) and narrow by period instead of deep pages. Related feeds
  (`include_related`) may hold one letter twice: describers give a
  `dedup_key`.
- Entries: `date` (ISO 8601 in the formatter time zone, `crm\CrmDate`), `type`,
  `subtype`, `title`, `summary`, `direction`, `channel`, `actor {kind:
  employee|client|system}`, `ref {tool, id}` (the tool that opens the record),
  `url` of the admin card.

## Overview for going through the base

One call per client, or a page of 50 with filters, so an AI client does not
make dozens of calls: status, sphere (company categories), responsible, the
client's own sites (links except social networks), last contact (newest call
or message of the record and its leads), cooperation (deals, paid bills,
incoming payments, `state`, `ended_at`), open tasks; packages add sections
for a whole page at once. Filters on the last contact use separate indexed
subqueries joined by `GREATEST()`; one subquery with `OR` over the record and
its leads scans the log (minutes on 100k entries).

## Personal data

- Every text and contact of the read tools goes through `crm\CrmReadContext`:
  emails keep the domain (`i***@domain.ru`), phones two digits, free text
  loses addresses and numbers (`[телефон]`); full values only with
  `include_contacts=true`. Names stay (business part). Contacts of a card are
  returned as counts by default.
- Personal links of a client are replaced in every text whatever the flag
  says (`[личная ссылка клиента]`): opening a sending page counts as the
  client's opening, an opt-out link unsubscribes. Never return sending codes.
  A package with such a page adds its path to `personalLinkPatterns`.
- Mail text for AI: `MailText::withoutQuotes()`, then cut the signature,
  mobile footer and unrecognised quote headers (`--`, a line of dashes,
  `От:`/`From:` in the lower part); attachments by name only.

## Scopes and tests

Reuse existing scopes when the data belongs to them (`cms.activity.read`,
`cms.company.read`, `cms.communication.read`): a new scope makes every client
authorize again. Test read tools against a local site in a read-only
transaction through `McpComponent::executeTool()` with an unsaved token
carrying the resource scopes — the same path as REST and MCP; compare the
history with the card queries of real records and the lists with the
`forManager()` queries for a worker and an administrator.
