# Audit Log & Compliance (`nx_audit_log`) - Odoo 18

A tamper-evident audit trail for **any Odoo model, with no per-model code**.
It records field-level changes (direct, derived and database cascade),
the content of deleted records, business actions, data access (export,
print, import, bulk API reads), logins, and failed operations. It raises
alerts, seals evidence into an HMAC hash chain, applies retention policies
and supports governed (four-eyes) GDPR / PDPL redaction.

| | |
|---|---|
| Version | 18.0.1.0.0 |
| Depends | `base`, `web`, `mail`, `base_import` |
| License | OPL-1 |
| Author | Sayed Anwar |
| Demo data | `nx_audit_log_demo` (optional, removable) |
| User guide | `static/description/nx_audit_log_user_guide_en.pdf` (English), `nx_audit_log_user_guide_ar_msa.pdf` (Arabic) |

---

## 1. The problem

Odoo shows the current state of your data, not how it got there:

* Most fields are not tracked. An invoice total, a price or a bank account can change and the old value is lost.
* A deleted record is gone: nobody can say what it contained or who removed it.
* Exports, prints and API reads leave no trace, so data leaks are invisible.
* Chatter messages can be deleted and server logs are technical. Nothing can be shown to an auditor as proof.

## 2. What the module does

| Area | What you get |
|---|---|
| Changes | Create / update / delete / copy, one line per field with old → new value and origin: **Direct** (typed by the caller), **Derived** (recomputed by Odoo), **Database cascade** (done by PostgreSQL). One2many lines and many2many links included. |
| Deleted records | Full delete snapshot, plus child events for lines deleted with the record and links cleared by the database. |
| Business actions | Button methods (Confirm, Post, Archive...) as ACTION events linked to the field changes they caused. |
| Data access | Exports, printed reports, imports (as batches) and API reads above a threshold. Optional per-request read auditing. |
| Authentication | Login, logout, second factor, failed logins grouped by login + IP + time window. |
| Failures | Operations refused to the user (e.g. deleting a posted entry). |
| Alerts | Deleted record, failed operation, field changed (any / absolute / percent), mass deletion, brute force, business action. Deduplication and hourly cap. |
| Integrity | Evidence is read-only for everybody, sealed into HMAC-SHA256 hash chains (per company and year) and verified daily. |
| Retention | Online → archived (searchable) → tombstone → signed checkpoint, per policy. Finance evidence can be marked "never purge". |
| Privacy | Secret fields never stored, personal data flagged or kept as keyed digest, four-eyes redaction that keeps the chain valid. |
| Reading | Dashboard, filtered event menus, event form with tabs, record timeline (**Action ⚙ → Audit Trail** on every audited model), field-change pivot, batches. |

## 3. How it works

```
User click / API call / cron / import
        │
        ▼
ORM hooks (create, write, unlink, copy, export, load) + RPC, login, error hooks
        │   BEFORE state read in SQL at first touch
        ▼
Accumulator (one per DB transaction; savepoint rollbacks and retries remove entries)
        │
        ▼  at Cursor.commit()
Finalizer: AFTER state in SQL → diff → rule match → masking
        │
        ├─► audit.log + audit.log.line   (same transaction as the business change)
        └─► alert outbox                 (same transaction, delivered by cron)

Cron every 5 min: seal events into hash chains     Cron daily: verify every chain
```

A rolled-back change never becomes evidence; a committed change always does.
The source (Web UI, JSON-RPC, XML-RPC, controller, cron, import, automated /
server action, system) is decided by the server: a client cannot disable
auditing or fake its source. See `doc/ARCHITECTURE.md` for design decisions.

## 4. Installation

1. Copy `nx_audit_log` to your addons path and install **Audit Log & Compliance** from *Apps*.
2. Add the integrity key to `odoo.conf` and restart (strongly recommended):

   ```ini
   nx_audit_hmac_key = <64 random hex characters>   ; e.g. openssl rand -hex 32
   ```

   Without it a key is generated in the database and the settings show a warning.
   Keep the key with your backups: without it a restored database cannot be verified.
3. *Audit → Configuration → Audit Rules*: default rules are created for the apps installed at that time.
   After installing new apps click **Load default rules** in *Audit → Configuration → Settings*.
4. The broad rule *All business models (Create / Update / Delete)* ships **inactive**:
   review its excluded models, then activate it for near-total coverage.
5. Give roles (section 6) and set recipients on the alert rules.
6. Optional: install `nx_audit_log_demo` on a test database for realistic demo evidence
   (uninstalling it removes every demo record and demo event).

## 5. Configuration

### 5.1 Settings (*Audit → Configuration → Settings*, Audit Administrator)

Every change of these settings is itself recorded as a `CONFIG` event.

| Setting | Default | Meaning |
|---|---|---|
| Text previews | 300 | Characters shown in the Old / New value columns. The full structured value is stored separately. |
| Large values (max stored text length) | 20000 | Longer text / HTML is stored as digest + beginning. |
| Evidence language | `en_US` | Language of record names and labels stored in evidence. |
| Excluded fields | `write_date, write_uid, create_date, create_uid, __last_update, message_ids, message_follower_ids, activity_ids, message_main_attachment_id, website_message_ids, parent_path, standard_price, access_token` | Never audited unless a rule selects them explicitly. |
| If auditing fails | Keep business running | *Keep business running and record an engine error* (availability) or *Block the transaction* (compliance). |
| Audit every export | On | Record all exports, even on models without a rule. |
| Bulk API read threshold | 500 | API reads returning at least this many records are logged as bulk reads. |
| Bulk operation threshold | 100 | One write/delete touching this many records is grouped into a batch. |
| Failed login window | 10 min | Failed logins are counted per login, IP and window. |
| Store unknown login names | Off | Off = unknown typed logins are stored hashed (users sometimes type a password there). |
| Sealing delay | 120 s | Delay before an event is sealed into its chain. |
| Integrity key | - | Shows whether the key comes from the configuration file or the database. |
| Maintenance | - | **Load default rules**, **Seal pending events**, **Enable value search indexes** (pg_trgm GIN). |

### 5.2 Audit rules (*Audit → Configuration → Audit Rules*)

| Field | Meaning |
|---|---|
| Scope | *One model* or *All eligible models* (+ Excluded Models). |
| Model, Sequence, Active | Audited model; lowest sequence wins when several rules match. |
| Operations | Create, Update, Delete, Business Actions, Export, Print / Report, Read (aggregated per request, expensive). |
| Action Methods | Comma-separated method names; empty = every button method. |
| Field Mode | All fields / Selected fields only / All fields except excluded. |
| Secret Fields | Change recorded, value never stored. |
| Derived Policy | Direct only / Direct + selected derived / Direct + all derived. |
| Domain | Only records matching before *or* after the change (stored fields only). |
| Companies, Sources, Only / Excluded Users and Groups | Who and where the rule applies. |
| Snapshot Policy | Changed fields, or full before/after snapshot (deletes always keep a snapshot). |
| Retention Policy | Online and total retention for the rule's events. |
| Chatter Mirror | Also post a short summary on the record chatter. |

A rule on a model also audits the lines it owns (one2many with cascade delete):
the *Journal entries* rule audits invoice lines, the *Sales orders* rule audits order lines.

**Default rules**

| Rule | Model | Notes | Retention |
|---|---|---|---|
| Journal entries & invoices | `account.move` | print on, full snapshot | Finance |
| Payments | `account.payment` | print on | Finance |
| Chart of accounts / Bank accounts | `account.account` / `res.partner.bank` | | Finance |
| Sales orders / Purchase orders | `sale.order` / `purchase.order` | print on | Standard |
| Transfers | `stock.picking` | | Standard |
| Stock quantities | `stock.quant` | selected fields only, direct only | Standard |
| Contacts | `res.partner` | direct only | Standard |
| Users & access | `res.users` | password secret | Security |
| Security groups | `res.groups` | users, implied groups, name | Security |
| Access rights / Record rules / Companies | `ir.model.access` / `ir.rule` / `res.company` | | Security |
| API keys / System parameters | `res.users.apikeys` / `ir.config_parameter` | key secret | Security |
| Attachments (metadata only) | `ir.attachment` | name, type, size, linked record | Standard |
| All business models | all eligible | **inactive**, direct only | - |

### 5.3 Alert rules (*Audit → Configuration → Alert Rules*)

Fields: Trigger, Severity, Model / Field / Action Method, Threshold type
(any / absolute / percent) and Threshold, Count and Window (minutes),
Deduplicate (minutes, default 30), Max alerts / hour (default 20),
Recipients / Recipient Group (empty = all Audit Managers).
Alerts are delivered every 5 minutes as a chatter notification and a *To Do*
activity; states: Pending → Notified → Acknowledged → Closed (or Suppressed).

| Default alert rule | Trigger | Severity |
|---|---|---|
| Journal entry deleted | Record deleted on `account.move` | High |
| Journal entry deletion blocked | Failed operation on `account.move` | High |
| Invoice total changed > 10% | `account.move.amount_total`, percent 10 | High |
| Bank account number changed | `res.partner.bank.acc_number` | Critical |
| User groups changed | `res.users.groups_id` | High |
| Group members changed | `res.groups.users` (inactive) | High |
| Mass deletion | 50 deletions in 10 minutes by one user | Critical |
| Brute force login | 10 failed logins | Critical |

### 5.4 Sensitive fields, redaction, retention

* **Sensitive Fields**: regex patterns on model / field names. *Secret* = never stored;
  *Personal* = stored and flagged (redactable) or stored as keyed digest only.
  Built-in secret names (password, token, api_key, secret, otp...) are always masked.
  Defaults: passwords and secrets, API key value, payment provider credentials, emails, phones,
  bank account numbers, identification numbers, birth dates, postal addresses.
* **Redaction Requests**: Draft → *Submit for approval* → *Approve* by another administrator
  (the requester cannot approve) → *Execute redaction*. Plain values are removed, digests and
  hashes stay valid, and a `REDACTION` event records the operation.
* **Retention Policies**: Online (days), Total retention (days, 0 = forever), Prohibit Purge.
  Defaults: *Finance* (730 days online, kept forever, purge prohibited), *Security* (365 / 1825),
  *Standard* (365 / 1095).

## 6. Roles

| Group | Access |
|---|---|
| Audit User | Dashboard, all Audit Logs menus, record timelines |
| Audit Manager | + Alerts, Reports, snapshots, archived events, integrity checks, hash chains |
| Audit Administrator | + Configuration (settings, rules, alert rules, sensitive fields, retention, redaction) |
| Audit: see shared / global records | Events on records without company (implied by Audit User) |
| Audit: cross-company auditor | Events of every company |
| Audit: compliance override for restricted fields | Lines of fields restricted with `groups=` |

Nobody, including administrators, has create / write / delete access on evidence.

## 7. Hands-on example: catch a changed invoice

On a test database with Accounting installed:

1. *Audit → Configuration → Audit Rules*: check *Journal entries & invoices* exists (else **Load default rules**).
   Open alert rule *Invoice total changed > 10%* and add yourself as recipient.
2. *Invoicing → Customers → Invoices → New*: 10 × a product at 1,000. **Confirm**.
   **Reset to Draft**, change the price to 1,250, **Confirm** again.
3. *Audit → Audit Logs → Updates*, filter *Journal Entry*: open the event. The *Changes* tab shows the price
   (direct) and totals (derived), old → new, with user, IP and source *Web UI*.
4. *Audit → Alerts*: "Total changed on ... 11,500.00 → 14,375.00". Click **Evidence**, then **Acknowledge**.
5. On the invoice, **Action ⚙ → Audit Trail** shows the whole story in a timeline.
6. Try to delete the posted invoice: Odoo refuses, *Failed Operations* shows the attempt and the
   alert *Journal entry deletion blocked* is raised.
7. *Settings → Seal pending events*, then *Reports → Integrity Checks → New → Verify now*: **Passed**.

## 8. For developers

```python
from odoo.addons.nx_audit_log.services.api import audit_action, audit_bypass, record_action

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @audit_action('Approve discount')          # semantic ACTION + linked field changes
    def action_approve_discount(self):
        ...

    def _sync_from_erp(self):
        with audit_bypass('ERP mirror sync - audited on the ERP side'):  # written to the server log
            ...

record_action(records, 'Recalculate commissions', method='_recalc')   # imperative variant
```

* Override `_audit_dynamic_secret_fields(self, row)` to mask values per row.
* `flush_audit(env)` finalizes pending evidence without committing (tests, long scripts).
* Direct SQL writes bypass the ORM: run `tools/sql_coverage_scan.py` after each upgrade (see `doc/COVERAGE.md`).

Run the tests:

    odoo-bin -d test_db -i nx_audit_log --test-enable --test-tags nx_audit --stop-after-init

## 9. Operations

| Scheduled action | Interval |
|---|---|
| Audit: seal events (hash chain) | 5 minutes |
| Audit: dispatch alerts | 5 minutes (also triggered on demand) |
| Audit: retention (archive / purge / checkpoints) | daily |
| Audit: verify integrity | daily |

* After the first large purges: `VACUUM (ANALYZE) audit_log, audit_log_line, audit_log_archive;`
* Large databases: **Enable value search indexes** in the settings (needs `pg_trgm`).
* For non-repudiation export the `audit.chain` heads / checkpoints regularly to external immutable storage.
* Uninstalling drops the audit tables; the `uninstall_hook` first exports every table to
  `<data_dir>/nx_audit_log_exports/<db>/<timestamp>/*.jsonl.gz`.

## 10. Known limitations

* Raw SQL writes are not captured (e.g. reconciliation flags on `account.move.line`); see `doc/COVERAGE.md`. Password changes are covered by a dedicated hook.
* Data loaded during module install / upgrade is not audited.
* Session expiry is not an event; only explicit logout is recorded.
* Deleting a company or user referenced by sealed evidence is reported by verification: archive them instead.
* A PostgreSQL superuser can rewrite rows and hashes together; export chain heads externally.
* Changes made inside `audit_bypass()` go to the server log, not to evidence.

## 11. Documentation

* `static/description/nx_audit_log_user_guide_en.pdf` - user guide (English)
* `static/description/nx_audit_log_user_guide_ar_msa.pdf` - user guide (Arabic, Modern Standard)
* `doc/ARCHITECTURE.md` - design decisions and limitations
* `doc/COVERAGE.md` - direct-SQL inventory
