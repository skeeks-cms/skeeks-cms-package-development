# CRM entity names in the interface

Decision of the owner, 27.10.2026. Interface words and technical names are
different on purpose; never "fix" one by the other.

| Interface (ru) | Technical name | Former interface word |
|---|---|---|
| Сделка / сделки — a sale attempt | `CmsLead`, record type `lead`, `cms_lead_id`, `cms/admin-cms-lead` | Лид / лиды |
| Услуга / услуги — one-off work or regular service | `CmsDeal`, record type `deal`, `cms_deal_*`, `cms/admin-cms-deal` | Сделка / сделки |
| Заказ | `ShopOrder` (`cms-shop`) | — |

## Rules

- Do not rename classes, tables, columns, record types, routes, controllers,
  RBAC permission keys, MCP/REST tool names or API parameters to match the
  words. Change only text people see: menus, titles, tabs, labels, hints,
  buttons, empty states, notification kinds, feed texts, permission
  descriptions, validation and error messages, MCP descriptions.
- "сделка" in code next to `CmsLead` and in code next to `CmsDeal` mean
  different things. Decide every text by the class it belongs to; never
  replace a word across a package blindly.
- `CmsLead::deal_amount` is shown as «Сумма сделки»: correct after the rename.
- Grammar: «лид» was masculine, «сделка» is feminine. Rewrite the whole phrase
  («Сделка закреплена», «Новая сделка»). A message that glues a variable label
  to an adjective (`$label.' недоступен'`) must be made gender-neutral
  (`$label.': нет доступа…'`).
- Russian strings in `Yii::t('skeeks/cms' | 'skeeks/hosting' | 'skeeks/shop/app', '…')`
  have no entries in the message files (`cms/src/messages/ru` translates only
  English keys), so the Russian key itself is the displayed text: change the key.
- MCP/REST descriptions name both words so AI clients do not confuse them:
  `cms_deal_*` — услуги (обслуживание клиента); `lead` / `cms_lead_id` —
  сделки (продажи).
- Already stored texts (feed entries «добавил лид…», sent notifications, agent
  descriptions in the database) are history and are not rewritten.
- Source-reading contract tests (`cms/tests/cms-lead-contract.php` and others)
  assert exact strings; update them together with the text.

## Admin menu «Услуги»

`AdminComponent::getMenuFilesData()` merges every installed extension's
`config/admin/menu.php`, then `@common/config/admin/menu.php` and
`@app/config/admin/menu.php`, with `ArrayHelper::merge`: groups with the same
string key are joined and their numeric `items` are appended. `BackendMenu`
then sorts each level by `priority` (default 100).

- `skeeks/cms` declares the group key `project` («Услуги», priority 200) with
  the first item «Услуги клиентов» (`cms/admin-cms-deal`, priority 10).
- Packages and projects add their service resources to the same key
  `project` without a priority (`cms-hosting`: Сайты, VPS, DNS, SSL; a project:
  Продвижение, Товарные базы…). No package depends on another for this.
- «Сделки» (`cms/admin-cms-lead`) stays in the group `crm` «Клиенты и сделки».
- A collapsed group renders no children in the sidebar; open a page of the
  group to check its items.
