# Oyster Panel Interview Pack — Stage 2 (Set B)

> Prepared for: **Senior Ruby on Rails Engineer** — Virtual Panel + Code Improvement Exercise
> Author: *Your Name (replace with Silva Rui)*
> Purpose: A fresh set of questions and hands-on exercises with complete answers and extra context, aligned to Oyster’s rubric.

---

## Oyster Context (refreshed)
- **Compliance-first**: Employment, payroll, and benefits vary per country → build **rule versioning**, **auditable changes**, **config-over-code** for localizations.
- **Async & global**: Decisions captured in **ADRs/RFCs**; handovers are critical around payroll cutoffs; minimize **blocking dependencies**.
- **Peak-aware scaling**: Monthly spikes around payroll approvals and disbursements; design **backpressure**, **idempotency**, **queue SLOs**.

> Tip: In every answer, mention **auditability**, **idempotency**, **observability**, and **safe migrations**.

---

## Part 1 — Questions, Model Answers & Extra Context (by competency)

### 1) Engineering Excellence
**Q1.** *How do you perform a zero-downtime schema change for a hot `payroll_runs` table (adding a non-null column with a default)?*

**Model Answer (short)**
1. **Expand**: `ALTER TABLE ... ADD COLUMN new_col TYPE NULL` (no default to avoid table-wide rewrite on Postgres).  
2. **Backfill in batches** using background jobs with `find_in_batches` / `in_batches.update_all` and throttle by load; wrap in **small transactions**.
3. **Dual-write**: Application writes both old+new fields behind a **feature flag**; new reads are guarded until coverage is ~100%.
4. **Set default** and **not null** only after backfill is complete (`ALTER TABLE ... SET DEFAULT`, then `... SET NOT NULL`).
5. **Contract tests** for serialization to ensure clients don’t break.

**Extra Context**: Emphasize avoiding long table locks, providing rollbacks, and observing backfill progress (metrics, logs). For extremely hot paths, prefer **computed defaults** at read-time until the backfill is done.

---

### 2) Communication
**Q2.** *Legal requests a last-minute copy change for multiple locales two hours before a payroll cutover. Product wants to stick to the freeze. How do you align async?*

**Model Answer (short)**
- Acknowledge both constraints; propose a **copy-only config release** that is **feature-flagged** and **scoped to locales** impacted; confirm no backend changes.  
- Create a brief **decision note** (Context → Options → Decision → Owner → Rollback) and circulate in the thread; schedule a **15-min overlap** only if a blocker remains; otherwise proceed async.

**Extra Context**: Show you can hold a freeze while unblocking low-risk updates via **config and flags**; capture rationale for later audits.

---

### 3) Customer Centricity
**Q3.** *How would you reduce accidental payroll submission errors by admins during peak time?*

**Model Answer (short)**
- Add **pre-submit checks** (unapproved timesheets, missing bank data, negative net pay) with **actionable links**.  
- Provide a **dry-run** mode that generates a **diff report** (previous vs. current payroll) before finalization.  
- Persist a **draft→ready→submitted** state machine with **server-side confirmations** (not only UI).  

**Extra Context**: Errors are expensive; preventing them with guardrails beats adding post-facto support workflows.

---

### 4) Impact & Results
**Q4.** *Mid-sprint, a provider deprecates an API we use for tax rates. The change is mandatory within 10 days. What do you optimize for?*

**Model Answer (short)**
- **Outcome-first**: uninterrupted tax accuracy. Spin a spike → **adapter layer** with the new API; keep old adapter in parallel.  
- **Shadow traffic** & **result diffing** to detect discrepancies; cut over via **flag** by tenant/region; commit to a **timeline** and **risk doc**.  
- Track **error budget** and **accuracy divergence** as key metrics.

**Extra Context**: This shows bias to action with safety nets; adapters contain blast radius and ease future swaps.

---

### 5) Use of Data — Engineering
**Q5.** *Design an experiment to decrease time-to-first-payroll after onboarding.*

**Model Answer (short)**
- Define metric: **TTFP** (median days from account creation → first payroll).  
- Hypothesis: Guided bank setup + inline validators will reduce TTFP by ≥20%.  
- **A/B** at tenant level, power calculation for sample size; log funnels: *onboard → bank_valid → contract_signed → payroll_created*.  
- Ship behind a **flag**, collect **p50/p95 TTFP**, **drop-offs**, and **ticket volume**; analyze with **non-parametric** tests if distributions are skewed.

**Extra Context**: Careful not to bias cohorts by region; consider seasonality around payroll cycles.

---

### 6) Code & Architecture Quality
**Q6.** *How would you implement a maintainable, country-aware rules engine for payroll taxes?*

**Model Answer (short)**
- **Strategy pattern** or data-driven rules: `RuleSet[country, version]` evaluated over a normalized `PayContext`.  
- **Versioned rules** stored as config (JSON/YAML) with validation; **effective_at** timestamps; roll-forward only.  
- Comprehensive **golden test fixtures** per country and **property-based tests** for edge values.  

**Extra Context**: Prefer configuration + evaluation engine to avoid a sprawling `if/else` jungle; ensure determinism for audits.

---

### 7) Delivery
**Q7.** *What’s your change-management plan during payroll week?*

**Model Answer (short)**
- **Freeze** critical paths; only allow **flagged**, **reversible**, **low-risk** changes; pre-approve **runbooks**.  
- Canary deploys; **synthetic probes** for create/read/write in each region; **auto-rollback** on SLO breach.  
- Staff **follow-the-sun on-call** with clear escalation; capture postmortems.

**Extra Context**: Demonstrate restraint and operational maturity; payroll week is different from normal weeks.

---

### 8) How We Work (Distributed)
**Q8.** *How do you run inclusive async design reviews across time zones?*

**Model Answer (short)**
- Use an **RFC template** with context, constraints, options, and explicit trade-offs.  
- Set a **response window** (e.g., 48–72h), tag stakeholders, and summarize decisions in an **ADR**; record a short Loom for complex diagrams.  
- Close with clear **Next Steps** and ownership.

**Extra Context**: Inclusion is time; async windows and crisp summaries raise quality and buy-in.

---

## Part 2 — Code & Improvement Exercises (Brand-new set)

> Format: **Original → Issues → Improved Code → Why it works**

### Exercise 1: Extract Business Logic from Controller + Proper Transactions
**Original**
```ruby
class ApprovalsController < ApplicationController
  def create
    payroll = PayrollRun.find(params[:id])
    payroll.update!(approved: true)
    PayoutService.disburse!(payroll)
    render json: { status: 'ok' }
  end
end
```

**Issues**
- Business logic in controller; no transaction boundaries; failure halfway leads to partial state.

**Improved**
```ruby
class ApprovePayroll
  def initialize(payroll_id)
    @payroll_id = payroll_id
  end

  def call
    PayrollRun.transaction do
      payroll = PayrollRun.lock.find(@payroll_id)
      raise 'Already approved' if payroll.approved?

      payroll.update!(approved: true)
      PayoutService.disburse!(payroll) # may raise
      Audit.log('payroll_approved', payroll_id: payroll.id)

      payroll
    end
  end
end
```
```ruby
class ApprovalsController < ApplicationController
  def create
    payroll = ApprovePayroll.new(params.require(:id)).call
    render json: { id: payroll.id, status: 'approved' }
  rescue => e
    render json: { error: e.message }, status: :unprocessable_entity
  end
end
```

**Why it works**: Clear domain service; atomicity with DB transaction; pessimistic lock prevents concurrent approvals.

---

### Exercise 2: Optimistic Locking for Concurrent Profile Edits
**Original**
```ruby
class Employee < ApplicationRecord
end
```
```ruby
# Controller blindly updates
employee.update!(params.permit(:name, :bank_account))
```

**Issues**
- Last write wins; admins may overwrite each other’s changes.

**Improved**
```ruby
class Employee < ApplicationRecord
  # add integer column `lock_version` in a migration
end
```
```ruby
# Controller
employee.assign_attributes(employee_params)
employee.save!
```

**Why it works**: Rails optimistic locking raises `ActiveRecord::StaleObjectError` on stale updates; the UI can prompt the user to refresh and merge.

---

### Exercise 3: Multi-tenant Authorization & Row-level Scoping
**Original**
```ruby
class EmployeesController < ApplicationController
  def index
    render json: Employee.where(active: true)
  end
end
```

**Issues**
- Missing tenant scoping; risks data leakage across companies.

**Improved**
```ruby
class ApplicationController < ActionController::API
  before_action :require_tenant!
  def current_tenant
    @current_tenant ||= Tenant.find_by!(slug: request.headers['X-Tenant'])
  end
end
```
```ruby
class EmployeesController < ApplicationController
  def index
    employees = policy_scope(Employee).where(active: true)
    render json: employees
  end
end
```
```ruby
class EmployeePolicy < ApplicationPolicy
  class Scope < Scope
    def resolve
      scope.where(tenant_id: user.tenant_id)
    end
  end

  def index?
    user.tenant_id.present?
  end
end
```

**Why it works**: Enforces **least privilege** via policy scopes; all queries filtered by `tenant_id`, preventing cross-tenant exposure.

---

### Exercise 4: Circuit Breaker for Unreliable Tax Provider
**Original**
```ruby
class TaxClient
  def rates(country)
    Faraday.get("https://tax.example.com/rates/#{country}").body
  end
end
```

**Issues**
- No timeouts/retries; can cascade failures; no fallback.

**Improved**
```ruby
class TaxClient
  def initialize
    @conn = Faraday.new(request: { timeout: 3, open_timeout: 1 })
  end

  def rates(country)
    Circuitbox['tax_api'].run do
      response = @conn.get("https://tax.example.com/rates/#{country}")
      JSON.parse(response.body)
    end
  rescue Circuitbox::Error
    Rails.cache.read(["tax_rates", country]) || { 'status' => 'stale' }
  end
end
```

**Why it works**: **Circuit breaker** prevents thundering herds; returns cached/stale data with clear status to degrade gracefully.

---

### Exercise 5: Safe Background Migration (add defaulted column)
**Original**
```ruby
class AddStatusToPayslips < ActiveRecord::Migration[7.1]
  def change
    add_column :payslips, :status, :string, null: false, default: 'issued'
  end
end
```

**Issues**
- On large tables this can lock/rewrite entire table.

**Improved (two-step)**
```ruby
class AddStatusColumn < ActiveRecord::Migration[7.1]
  def up
    add_column :payslips, :status, :string
    add_index :payslips, :status
  end
  def down
    remove_column :payslips, :status
  end
end
```
```ruby
# Backfill job
Payslip.in_batches.update_all(status: 'issued')
```
```ruby
class EnforceStatusConstraints < ActiveRecord::Migration[7.1]
  def up
    change_column_default :payslips, :status, 'issued'
    execute "UPDATE payslips SET status = 'issued' WHERE status IS NULL"
    change_column_null :payslips, :status, false
  end
end
```

**Why it works**: Avoids table rewrite; constraints come **after** data is ready; backfill is throttled.

---

### Exercise 6: API Versioning & Deprecation
**Original**
```ruby
namespace :api do
  resources :payrolls, only: :show
end
```

**Issues**
- No versioning; breaking changes will hurt clients.

**Improved**
```ruby
namespace :api do
  namespace :v1 do
    resources :payrolls, only: :show
  end
  namespace :v2 do
    resources :payrolls, only: :show
  end
end
```
```ruby
class Api::BaseController < ActionController::API
  after_action :set_deprecation_headers
  private
  def set_deprecation_headers
    response.set_header('Deprecation', 'true') if request.path.start_with?('/api/v1')
    response.set_header('Sunset', (Date.today + 180).httpdate)
    response.set_header('Link', '<https://docs.example.com/api>; rel="deprecation"')
  end
end
```

**Why it works**: Explicit versions + HTTP deprecation headers allow **safe evolution** and give clients a migration path.

---

### Exercise 7: GDPR DSAR Export (streaming + signed URL)
**Original**
```ruby
class DsarController < ApplicationController
  def export
    user = User.find(params[:id])
    send_data user.to_json
  end
end
```

**Issues**
- Synchronous, blocking; no auth on re-download; may leak PII.

**Improved**
```ruby
class DsarExportJob < ApplicationJob
  queue_as :low
  def perform(user_id)
    user = User.find(user_id)
    io = StringIO.new(JSON.dump(DsarExport.build(user)))
    blob = ActiveStorage::Blob.create_and_upload!(io: io, filename: "dsar_#{user.id}.json", content_type: 'application/json')
    user.update!(dsar_blob_id: blob.id)
  end
end
```
```ruby
class DsarController < ApplicationController
  before_action :authenticate_user!
  def request_export
    DsarExportJob.perform_later(current_user.id)
    head :accepted
  end

  def download
    blob = ActiveStorage::Blob.find(current_user.dsar_blob_id)
    url = blob.service_url(disposition: 'attachment', expires_in: 15.minutes)
    render json: { url: url }
  end
end
```

**Why it works**: Async export avoids timeouts; **short-lived signed URL** controls access; PII never sent inline in responses or logs.

---

### Exercise 8: Sliding-Window Rate Limiter per Tenant
**Original**
```ruby
before_action :rate_limit

def rate_limit
  # TODO
end
```

**Issues**
- No protection vs abuse; per-IP is insufficient for multi-tenant APIs.

**Improved**
```ruby
class SlidingWindowLimiter
  WINDOW = 60 # seconds
  LIMIT  = 300

  def self.allowed?(tenant)
    now = Time.now.to_i
    bucket = "rl:tenant:#{tenant.id}:#{now / WINDOW}"
    count = Redis.current.incr(bucket)
    Redis.current.expire(bucket, WINDOW + 5) if count == 1
    count <= LIMIT
  end
end
```
```ruby
class ApplicationController < ActionController::API
  before_action :enforce_rate_limit
  def enforce_rate_limit
    tenant = current_tenant
    return if SlidingWindowLimiter.allowed?(tenant)
    render json: { error: 'Too Many Requests' }, status: 429
  end
end
```

**Why it works**: Simple sliding window per-tenant; distributed across nodes via Redis; prevents **noisy neighbors**.

---

## Part 3 — System Design Prompt (new)
**Prompt**: *Design a global document generation & e-signing pipeline (contracts, offer letters) with data residency and audit trails.*

**Compact Answer Outline**
1. **Ingress & Auth**: Tenant-scoped API; JWT with scoped permissions; request IDs for tracing.
2. **Template Service**: Versioned templates per locale/country; RBAC on who can publish; stored in-region.
3. **Renderers**: Stateless render workers produce PDFs from HTML; **deterministic assets**; watermarking for drafts.
4. **Storage & Residency**: PII stored in-region buckets; cross-region only store **hashes/metadata**; signed URLs for access.
5. **E-sign**: Integrate provider(s) via adapter; **webhook verifications**, **idempotent upserts**; append-only audit log of signer events.
6. **Observability & SLOs**: p95 render latency, error rate, queue depth; dead-letter for failed renders; runbooks.
7. **Rollouts**: Feature flags per tenant; canary with backpressure; fallback to cached last-known-good template if rendering fails.

**Why it fits**: Handles residency, auditability, and spikes; adapter pattern isolates external providers; deterministic outputs aid compliance.

---

## Part 4 — Delivery & Collaboration Aids (Set B)
- **Review checklist**: migrations safe? idempotency? country rules versioned? logs redacted? metric added? rollback plan?
- **Questions to ask the panel**
  1. How do you handle **schema changes** near payroll cutoffs?  
  2. What are your **error budgets** and on-call expectations during peak cycles?  
  3. How are **country rule changes** deployed and verified (shadow runs, diff tooling)?
- **STAR story ideas** (fill with your numbers): provider outage resilience, onboarding funnel uplift, GDPR export pipeline.

---

## Night-before Checklist (Set B)
- [ ] Rehearse zero-downtime migration steps out loud.
- [ ] Practice explaining circuit breakers vs timeouts vs retries.
- [ ] Prepare one concrete data story (TTFP or activation) with a chart if asked.
- [ ] Have a 1-minute self-intro that ties experience to **residency + reliability + async**.

---

**You’ve got this.** Keep answers crisp, reference flags/idempotency/observability, and finish with the *metric that matters*.
