# Data Tables Retention Cleanup



Scheduled, AI-free maintenance workflow for the FAQ assistant's configured n8n Data Tables. It runs daily at **03:00** (cron `0 3 * * *`) and deletes rows older than the configured retention period (30 days by default). The exported workflow does not set a timezone; set it to `Europe/Madrid` in the workflow settings if that is the intended local schedule.

---

## How it works

1. `Schedule 03:00` (cron `0 3 * * *`) or `Manual Run` starts the workflow. The timezone must be configured in workflow settings; otherwise the schedule uses the applicable n8n timezone rather than necessarily running at 03:00 Europe/Madrid.
2. **Run lock**: a row `table_name = "__run__"` in `maintenance_log` marks a running job. If a previous run is still within `lock_timeout_minutes`, the new run is skipped and a Slack warning is sent.
3. **Plan**: one item per configured table with its cutoff timestamp. Invalid entries (unset `table_id`, `retention_days < 1`) are reported, never executed.
4. **Per table, one after another** (`Loop Tables`):
   - read up to `max_delete_per_table + 1` candidate rows (Data Table *Get* with `ts_column < cutoff`);
   - **safety cap**: more candidates than the cap -> status `blocked`, nothing is deleted;
   - **dry run**: only reports what would be deleted;
   - otherwise *Delete rows* with the same filter;
   - the result is written to `maintenance_log` (Data Table write).
5. **Summary**: the lock is released after table processing; Slack sends a summary only if a table is `error` or `blocked` (or `notify_always` is true).
6. A global `Error Trigger` reports unexpected failures to Slack.

---

## Why numeric timestamp columns

Each table is cleaned by a **numeric epoch-millisecond column** that you control (not by the system `createdAt` / `updatedAt`). That keeps the filter simple and independent of how a given n8n version handles date comparisons, and it lets one column mean different things per table:

| Table | `ts_column` | Retention | Notes |
|---|---|---|---|
| `kb_qa_log` | `ts` | 30 days | main source of growth |
| `kb_unanswered` | `last_seen_ms` | 30 days | questions that keep coming back stay |
| `kb_alert_state` | `updated_ms` | 7 days | technical state, needs little history |
| `kb_documents` | `cleanup_ts` | 30 days | `0` for `indexed` rows, so they are **never** deleted; only `failed` rows get a timestamp |
| `maintenance_log` | `ts` | 90 days | the log cleans itself |

Every delete uses two conditions: `column < cutoff` **and** `column > 0`. The second one is what protects rows that deliberately carry `0`.

---

## Data Table `maintenance_log`

| Column | Type |
|---|---|
| `run_id`, `table_name`, `status`, `message`, `cutoff_iso` | string |
| `candidates`, `deleted`, `dry_run`, `ts` | number |

`status` values: `ok`, `dry_run`, `deleted`, `blocked`, `error` (plus `running` / `finished` on the `__run__` lock row).

---

## Config reference

| Key | Default | Meaning |
|---|---|---|
| `dry_run` | **`true`** | report only. Set to `false` after the first review |
| `retention_days_default` | 30 | used when a table has no `retention_days` |
| `max_delete_per_table` | 5000 | safety cap per table and run |
| `lock_timeout_minutes` | 120 | after this time a stuck lock is ignored |
| `notify_always` | `false` | also send a Slack summary when everything is fine |
| `slack_channel_id` | `C0000000000` | alert channel |
| `tables[]` | see JSON | `name`, `table_id`, `ts_column`, `retention_days` |

To find a `table_id`, open the Data Table in n8n and copy the ID from the URL. Entries with `REPLACE_WITH_TABLE_ID` are reported as errors and skipped. To add a table, add one object to `tables`; make sure it has a numeric timestamp column.

---

## Setup

1. Create the `maintenance_log` table and the FAQ assistant's four Data Tables: `kb_qa_log`, `kb_unanswered`, `kb_alert_state`, and `kb_documents`. This workflow only cleans tables listed in its `Config`.
2. Import `data_tables_retention_cleanup.json`.
3. Attach the **Slack** credential to the Slack nodes.
4. Open `Get Lock`, `Acquire Lock`, `Write Log` and `Release Lock` and select the `maintenance_log` table from the list.
5. Fill in `Config`: the real `table_id` for each table, Slack channel id; also the channel id inside `Format Error`.
6. In workflow settings, set the timezone to `Europe/Madrid` if the job should run at 03:00 Madrid time. Set *Error Workflow* to this workflow to enable its global error alert.
7. **First run: keep `dry_run = true`**, start it with `Manual Run`, and check `maintenance_log` and the Slack summary.
8. If the numbers look right, set `dry_run` to `false` and activate the workflow.

### First deployment with a big backlog

If a table already holds more old rows than `max_delete_per_table`, the run reports `blocked` and deletes nothing. That is the safety net working. Raise the cap once (for example to the real number plus a margin), run it manually, then lower it again.

---

## Known limitations

- The lock is a Data Table read followed by an upsert, not an atomic distributed lock; simultaneous starts can race. A crash in the middle of a run leaves the lock as `running` until `lock_timeout_minutes` passes; a later run can then proceed.
- Counting is done by reading candidate rows (up to cap + 1), which is fine for a daily job on tables that are cleaned regularly.
- The logged `deleted` value is the candidate count read before deletion, not a count verified from the delete operation's response. Avoid concurrent writers that can change matching rows during cleanup if exact counts matter.
- Inside the loop, Code nodes read the current table with `$('Loop Tables').first()`. Test it once with two or more tables in dry-run mode after import.
- MongoDB collections (for example the FAQ assistant's agent chat memory) are not touched; use a TTL index there.
- The workflow JSON leaves `maintenance_log` table selections unset. Open `Get Lock`, `Acquire Lock`, `Write Log`, and `Release Lock` after import and select the actual table.
