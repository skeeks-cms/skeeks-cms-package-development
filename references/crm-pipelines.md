# Deal pipelines, stages and the history of transitions

Verified in `skeeks/cms` (П2б, 2026-10-28, stand). Words: «сделка» =
`CmsLead`, «воронка» = `CmsLeadPipeline` (`crm-entity-names.md`).

## Model

- `CmsLeadPipeline` (`cms_lead_pipeline`): a site's pipeline — name, `code`
  (`inbound`, `outbound` for the defaults), `sort`, `is_default`,
  `success_rule` (`manual|order|payment`).
- `CmsLeadStage.cms_lead_pipeline_id`; a stage name is unique within its
  pipeline. `CmsLead.cms_lead_pipeline_id`, `CmsLead.stage_changed_at`.
- The four statuses stay authoritative: «Неразобранное» is `new`, the stages
  split `in_work`, «Успешно»/«Отказ» are `success`/`rejected`. Never add a
  "won/lost stage": partner cabinets, the API and reports read the statuses.
- **Implicit pipeline.** A site without pipelines has one: stages and deals
  without a pipeline. Once pipelines exist, rows without a pipeline belong to
  the default one (`CmsLeadPipeline::defaultFor()`, `isMain`). Use
  `$pipeline->leadCondition($alias)`, `$pipeline->stages`,
  `CmsLeadStage::forPipeline($pipeline)` and `$lead->getEffectivePipeline()`
  — never `cms_lead_pipeline_id = X` alone, never `CmsLeadStage::forCurrentSite()`
  for a board (it mixes pipelines).
- Readiness checks before the migration: `CmsLead::pipelineReady()`,
  `CmsLeadPipeline::ready()`, `CmsLeadStage::pipelineReady()`.

## Lifecycle rules (CmsLead)

- A new deal without a pipeline goes to the default one (forms, letters,
  API); a caller may set another. An unsorted deal has no stage.
- A stage must belong to the deal's pipeline (validation). Moving to another
  pipeline drops a foreign stage; a deal in work then takes the first stage.
- `stage_changed_at` is set on every change of status, pipeline or stage.

## History: `CmsLeadStageHistory`

One row per change of status, pipeline or stage, written by
`CmsLead::afterSave()` inside the save transaction — every path that saves a
deal is covered; `updateAll()`/`updateAttributes()` writers are not (old
`LeadImport`). Kinds: `create`, `take`, `stage`, `pipeline`, `success`,
`reject`, `reopen`, `migrated`. The stage name is copied (history outlives a
deleted stage); stage and pipeline ids have no foreign keys. The author is the
signed-in user, else the creator of a new deal, else empty (system).

`is_migrated` rows are "the state on the day of the move to pipelines": not a
transition. Reports never take a start time from them and show this date as
the reliability boundary. Never invent historical dates from old data.

## Services

- `crm\LeadPipelines::move($lead, $pipelineId, $stageId, $actorId)` — the only
  way to move a deal between pipelines (rights through `CrmAccess`).
- Stage changes go through `CmsLeadBoardService::forLead($lead)->move()`
  (kanban, the card stage bar); a terminal column only asks for the result form.
- `LeadPipelines::successSignals($lead)` — the pipeline criterion only
  *suggests* closing; a person closes. Packages add reasons through
  `LeadPipelines::EVENT_SUCCESS_SIGNALS` (`LeadSuccessSignalsEvent`, `handled`
  replaces the built-in check). The built-in check reads shop tables only when
  they exist: the first incoming payment of the client after the deal opened;
  a confirmed order through `shop_order.cms_lead_id` once it exists. The core
  does not depend on `cms-shop`.

## Board and report

- `new CmsLeadBoardService($pipeline, $filters)`: «Неразобранное» is always a
  column of the default pipeline, of another one only while it has deals;
  column `amount` sums open «Сумма» (`amount`) and, for success, `deal_amount`
  over a subquery (a search groups rows). Filters: `source_type`, `no_task`.
- `CmsLeadFunnelReportService`: a cohort of deals that *entered* the pipeline
  in the period (creation or the move); the furthest step reached by the
  history; conversion from «взято в работу»; sorting results and non-target
  reasons excluded; time on stages only from real transitions; first/repeat
  sales by closing date (repeat — the company, else the person, had an earlier
  successful deal; no client — unknown); one deal amount per deal, never bills
  or payments. Show these rules to people next to the numbers.

## Moving existing data

`php yii cms/lead-pipeline/init` (`crm\LeadPipelineMigration`): a report by
default, `--apply` to apply, repeatable (pipelines by code, only rows without
a pipeline or without history). Creates «Входящие» (default) and «Исходящие
продажи», cold deals (source name prefix `--coldNames`, `mailing`, `import`)
go to «Исходящие» on a copy of their stage, writes one `migrated` row per deal
without history. A new site does not need it.
