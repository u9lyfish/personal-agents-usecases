# Personal Agents Use Cases

> **中文简介**：本仓库收集 X（Twitter）上真实用户使用“个人 AI 智能体”（Grok Bot、Meta Muse、OpenAI Dots、Claude Duet、Manus Cue、Today AI、蚂蚁灵光、千问Go 等）的具体用例，包括成功和失败的经历。每条记录都是一条帖子，字段固定，同时有中英文摘要，以 JSONL 格式按日期存放在 `data/` 下。每个工作日早上（北京时间）更新一次，周一的那次覆盖周末。增量消费方式：读取 `index.json`，加载新增的 `data/*.jsonl`，按 `id` 去重。

A machine-readable, append-only dataset of **real, first-hand (or clearly reported) use cases of personal AI agents**, collected from public posts on X. It's meant for ingestion into a database.

## Inclusion rules

A post is included only if it describes a **concrete, real use** of a personal agent. The use can be first-hand ("my agent did X") or clearly reported (e.g. a named tester case), and failures and complaints count too.

Excluded:
- referral / invite-code spam (e.g. Muse code `4X23B6`, "we each get 1 billion tokens")
- memecoin shilling
- launch or funding news
- repo listicles
- generic hype or opinion
- questions with no use
- automated bot summaries (e.g. `@grok` replies)

Near-duplicate reposts by the same author are collapsed to a single record. Every record is unique on `id`.

Sponsored posts (`#ad`, `#sponsored`, `#…partner` and similar) are kept but flagged with `is_sponsored: true`.

## File layout

```
README.md            this file (schema + conventions)
products.json        tracked products (canonical names, vendors, aliases, search terms)
index.json           manifest of data files (for incremental consumption)
state.json           collector state (last run, search window end)
data/YYYY-MM-DD.jsonl  records collected in the run on that date (Asia/Shanghai date), one JSON object per line, UTF-8
```

The data files are **append-only**:
- Each run writes a new `data/YYYY-MM-DD.jsonl` named after the run date in Asia/Shanghai.
- Existing files are never rewritten, except to fix an error, which is noted in the commit message.
- A post already published in an earlier file is not emitted again.
- Within a file, records are sorted by `created_at` ascending, then `post_id`.

## Record schema

Every field is always present. Unknown values are `null`.

| field | type | description |
|---|---|---|
| `id` | string | Global unique key: `"x:<post_id>"`. **Dedupe on this.** |
| `source` | string | Always `"x"` for now. |
| `post_id` | string | X post ID, kept as a string to avoid precision loss. |
| `post_url` | string | `https://x.com/<author_username>/status/<post_id>` |
| `author_username` | string | Handle without `@`. |
| `author_name` | string | Display name at collection time. |
| `created_at` | string | Post time in ISO 8601 UTC, e.g. `2026-10-21T08:15:00Z`. |
| `lang` | string\|null | X language code (`en`, `zh`, `ja`, …). |
| `text` | string | Full original post text, including long posts (note_tweet). Not translated or trimmed. |
| `product` | string | Canonical product name; must match a `name` in `products.json`. |
| `vendor` | string | Vendor of the product, copied from `products.json`. |
| `use_case_category` | string | One value from the enum below. |
| `use_case_summary_en` | string | One-sentence English summary of what the agent did or failed to do. |
| `use_case_summary_zh` | string | The same summary in Simplified Chinese. |
| `sentiment` | string | `positive` \| `neutral` \| `negative` (the author's view of this use). |
| `is_complaint` | boolean | `true` if the post mainly reports a failure, limitation or frustration. |
| `is_sponsored` | boolean | `true` if the post is disclosed as an ad or partnership (`#ad`, `#sponsored`, `#…partner`). |
| `metrics` | object | `{likes, reposts, replies, quotes, bookmarks, impressions}`, integers (or `null`) as of `collected_at`. |
| `collected_at` | string | ISO 8601 UTC time the record was collected. |

### `use_case_category` enum

| value | meaning |
|---|---|
| `productivity_work` | inbox, calendar, docs, task management, briefings for work |
| `research_info` | finding, monitoring or summarizing information |
| `business_ops` | running a business: customers, invoices, ops, marketing ops |
| `coding_dev` | writing, deploying or debugging software |
| `trading_investing` | markets, crypto, portfolio monitoring |
| `shopping_booking` | buying things, reservations, appointments, returns |
| `travel` | flights, hotels, trip logistics, travel emergencies |
| `personal_life` | household, family, personal admin, finances, subscriptions |
| `creative_media` | writing, video, images, music, content publishing |
| `communication` | calls, messages, email on the user's behalf |
| `learning_education` | courses, studying, credentials |
| `health_wellness` | doctors, health data, fitness |
| `agent_building_meta` | setting up or orchestrating agents, multi-agent setups, agent self-maintenance |
| `other` | anything else, including generic product-level failures |

Each record has exactly one (primary) category.

## products.json

The file is an array of:

```json
{"name": "Meta Muse", "vendor": "Meta", "aliases": ["Muse"], "search_terms": ["\"Muse agent\""], "first_seen": "YYYY-MM-DD", "notes": "..."}
```

- `name` is the canonical name used in records.
- `first_seen` is the date the product was added to this dataset.
- New products are appended when discovered.
- Existing names are never renamed. Use `aliases` for alternative names.
- Some products are tracked even though no records exist for them yet.

## Incremental consumption

1. Fetch `index.json`:
   ```json
   {"updated_at": "<ISO UTC>", "files": [{"date": "YYYY-MM-DD", "path": "data/YYYY-MM-DD.jsonl", "count": N}], "total": N}
   ```
2. Load any `files[].path` you have not ingested yet (track by `path` or `date`). Each line is one record.
3. Upsert on `id`. If an `id` ever appears twice, keep the record with the latest `collected_at`.
4. Optionally reload `products.json` to pick up new products and aliases.

`state.json` is collector bookkeeping:

```json
{"last_run_at": "<ISO UTC>", "last_search_end_time_utc": "<ISO UTC>"}
```

The next run searches from `last_search_end_time_utc` (minus a small overlap) to now, and dedupes against already published ids.

## Update cadence

- Updated on **weekday mornings, Asia/Shanghai time**.
- The Monday run covers the whole weekend.
- Commit messages follow `data: YYYY-MM-DD (+N use cases)`.
- A run that finds nothing new still updates `state.json` and `index.json` (`updated_at`), and adds no data file.

## Caveats

- Classification and summaries are curated by an AI agent, so expect some judgment calls.
- Metrics are a snapshot taken at collection time and are not refreshed.
- Posts are public X content. They are quoted verbatim for research purposes, and copyright stays with their authors.
