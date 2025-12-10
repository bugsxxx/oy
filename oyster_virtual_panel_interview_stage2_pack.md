# Oyster Panel Interview Pack — Stage 2 (Virtual Panel + Code Improvement)

> Prepared for: **Senior Ruby on Rails Engineer** interview at Oyster
> Author: *Your Name (replace with Silva Rui)*
> Goal: Provide **ready-to-use answers**, **code improvement exercises**, and **context** mapped to Oyster’s rubric.

---

## Oyster Context (Concise)
- **Global HR & EOR**: Payroll, onboarding, and benefits across many countries → **PII/financial data, country-specific workflows, data residency**.
- **Distributed-first**: Async by default; clear written decisions (ADRs), PRDs, and measurable outcomes.
- **Reliability & Scale**: Payroll cycles are periodic spikes; **latency, accuracy, and auditability** matter more than raw feature velocity.

> **Interview Tip**: Weave these constraints into your answers (compliance, audit trails, idempotency, multi-region behavior, async collaboration).

---

## Part 1 — Questions, Model Answers, and Extra Context (by competency)

### 1) Engineering Excellence
**Q1.** *How would you design a Rails service that calculates payroll reliably for thousands of customers across regions?*

**Model Answer (short):**
- Split critical domains (calculation, compliance rules, disbursements) into **bounded contexts** using engines/services. Persist **immutable ledgers** for money movements. Introduce an **event log** (e.g., `PayrollCalculated`, `PaymentDisbursed`) so downstream systems react idempotently. Use **Sidekiq** for long-running jobs; **Postgres** with **row-level locks** for consistency; **read replicas** for analytics.
- Protect calculations with **idempotency keys** and **exactly-once semantics** at the application boundary. Use **feature flags** and **shadow traffic** to validate new calculators before full cutover.

**Extra Context:** Reliability is created by *isolation + idempotency + observability*. Immutable money ledgers simplify audits; events provide traceability; flags enable safe iteration.

---

**Q2.** *What practices ensure performance and operability while keeping operational overhead low?*

**Model Answer (short):**
- **Golden signals** in Grafana (latency p95, error rate, saturation, RPS) per endpoint and per tenant/region. **SLOs** with alerting that page only on customer-impacting breaches.
- **Caching strategy**: reference data (tax tables) cached with explicit version keys; **`Cache-Version`** invalidation on rule updates.
- **DB hygiene**: composite/partial indexes from real query plans; background migrations; **online index creation**; **query budgets** in CI.

**Extra Context:** Performance budgets and SLOs focus teams on outcomes (customer-perceived latency) rather than incidental metrics.

---

### 2) Communication
**Q3.** *A teammate misread your ADR asynchronously and implemented the wrong approach. What do you do?*

**Model Answer (short):**
- Respond publicly and constructively: summarize **intent**, **constraints**, and **trade-offs**; add a **TL;DR** on the ADR; propose a **small corrective PR** or flag-guarded rollback.
- Create a **template** for future ADRs (Context → Options → Decision → Consequences). Offer a **15–20 min overlap call** strictly for alignment; document the outcome.

**Extra Context:** The goal is *shared understanding*, not blame. Writing for time zones means front-loading context and decisions.

---

### 3) Customer Centricity
**Q4.** *Give an example of uncovering an unspoken customer need and translating it into a feature.*

**Model Answer (short):**
- Observed drop-offs between *contract creation → first payroll*. Logs showed repeated validation failures for country-specific bank formats.
- Implemented **pre-flight validation** (client + server) and **guided forms** with localized hints and examples; added **inline API to validate IBAN/SWIFT** before submission.
- Result: time-to-first-payroll down 38%, support tickets down 27% (illustrative numbers — replace with yours).

**Extra Context:** Pair usage analytics with support insights. The best “features” remove steps, not add them.

---

### 4) Impact & Results
**Q5.** *A regulatory change arrives mid-sprint affecting tax calculations. How do you adapt?*

**Model Answer (short):**
- **Re-scope**: create a *compliance epic* behind a **feature flag**; split into *parser*, *rule versioning*, *regression suite*. Pause non-critical work.
- **Risk-first rollout**: shadow compute old vs. new; **diff results**; alert on divergence >0.1%. Roll out by region/tenant cohort with **fast rollback**.

**Extra Context:** Tie changes to KPIs (calculation accuracy, time-to-close payroll). Answer should show prioritization + safe delivery mechanics.

---

### 5) Use of Data — Engineering
**Q6.** *How do you choose what to instrument in a Rails app to guide decisions?*

**Model Answer (short):**
- Define **north-star metrics** (activation, time-to-first-success), **reliability SLOs**, and **operational KPIs** (queue latency, retry rate).
- Add **domain events** with payloads sufficient for funnels, but **no PII** unless hashed; aggregate in a warehouse. Build **funnel dashboards** and **error budgets** to trigger investment.

**Extra Context:** Data informs *what to fix next* and *how deep to go*. Error budgets legitimize pausing features to pay reliability debt.

---

### 6) Code & Architecture Quality
**Q7.** *What’s your approach to keeping a large Rails codebase clean and evolvable?*

**Model Answer (short):**
- **Architecture**: keep fat models thin with **service objects**, **value objects**, and **form objects**; apply **bounded contexts** via Rails engines.
- **Standards**: RuboCop (Style/Security), Brakeman, Bundler Audit; **pre-commit hooks**; consistent **API contracts** and **JSON:API** shapes.
- **Tests**: Pyramid favoring unit/component tests; **contract tests** for services; **smoke suites** post-deploy.

**Extra Context:** Architecture is a social contract: consistency improves velocity and review quality.

---

### 7) Delivery
**Q8.** *How do you practice continuous delivery without risking payroll outages?*

**Model Answer (short):**
- **Zero-downtime deploys** (blue/green or rolling), **reversible migrations**, and **feature flags** for risky paths.
- **Change windows** for high-risk flows; **pre-flight synthetic checks** per region; **automated rollback** on SLO breach.

**Extra Context:** Delivery is a discipline: small, reversible changes reduce blast radius.

---

### 8) How We Work (Distributed, Async)
**Q9.** *What habits keep you productive across time zones?*

**Model Answer (short):**
- Write **clear, skimmable updates** (Context → Decision → Next steps). Use **threaded discussions** with owners and due dates.
- **Handover notes** at EOD with blockers and links. Prefer ** Loom / short videos** for walkthroughs when text is heavy.

**Extra Context:** Optimize for *readability* and *discoverability*. Future-you is a consumer of today’s notes.

---

## Part 2 — Code & Improvement Exercises (with answers)

> Format: **Original → Issues → Improved Code → Why it works**

### Exercise 1: Kill an N+1 and Add Pagination & Serde Guards
**Original**
```ruby
class CompaniesController < ApplicationController
  def index
    render json: Company.all.map { |c| { id: c.id, name: c.name, country: c.address.country, people: c.employees.count } }
  end
end
```

**Issues**
- Multiple N+1s (`address`, `employees`).
- Unbounded result set (memory/latency risk).
- Manual serialization can leak fields later.

**Improved**
```ruby
class CompaniesController < ApplicationController
  def index
    companies = Company
      .includes(:address, :employees)
      .order(created_at: :desc)
      .page(params[:page]).per(25)

    render json: companies.as_json(only: %i[id name], include: { address: { only: %i[country] } })
  end
end
```

**Why it works**: `includes` eliminates N+1s; pagination bounds work; `as_json` whitelists output to avoid accidental PII exposure.

---

### Exercise 2: Idempotent Money Movement with DB Transactions
**Original**
```ruby
class PayoutsController < ApplicationController
  def create
    payout = Payout.create!(tenant_id: params[:tenant_id], amount_cents: params[:amount_cents])
    PaymentGateway.disburse!(payout)
    render json: { status: 'ok', id: payout.id }
  end
end
```

**Issues**
- No idempotency → duplicates on retries.
- Side-effect before durable commit; no audit trail; no concurrency protection.

**Improved**
```ruby
class PayoutsController < ApplicationController
  def create
    key = params.require(:idempotency_key)
    tenant_id = params.require(:tenant_id)
    amount = params.require(:amount_cents)

    Payout.transaction do
      token = IdempotencyKey.lock.find_or_create_by!(key: key, scope: "tenant:#{tenant_id}")
      raise ActionController::BadRequest, 'Replayed request' if token.performed?

      payout = Payout.create!(tenant_id: tenant_id, amount_cents: amount, status: 'pending')
      PaymentGateway.disburse!(payout) # external side-effect
      payout.update!(status: 'settled')

      token.update!(performed: true, reference_id: payout.id)
      render json: { id: payout.id, status: payout.status }
    end
  rescue ActiveRecord::RecordNotUnique
    head :conflict
  end
end
```

**Why it works**: Idempotency + transaction boundaries ensure **at-most-once semantics** per key; auditability via persisted status.

---

### Exercise 3: Concurrency-Safe Background Job (unique per tenant)
**Original**
```ruby
class ClosePayrollJob < ApplicationJob
  queue_as :default
  def perform(tenant_id)
    PayrollCloser.new(tenant_id).call
  end
end
```

**Issues**
- Duplicate enqueues can close the same payroll twice during spikes.

**Improved**
```ruby
class ClosePayrollJob < ApplicationJob
  queue_as :critical

  def perform(tenant_id)
    key = "lock:close_payroll:tenant:#{tenant_id}"
    # NX = set if not exists; EX = TTL seconds
    acquired = Redis.current.set(key, 1, nx: true, ex: 600)
    return unless acquired

    begin
      PayrollCloser.new(tenant_id).call
    ensure
      Redis.current.del(key)
    end
  end
end
```

**Why it works**: Distributed lock guarantees a **single active closer** per tenant; TTL prevents deadlocks after crashes; `critical` queue prioritizes time-sensitive work.

---

### Exercise 4: Safe File Uploads with Active Storage and Content Validation
**Original**
```ruby
class DocumentsController < ApplicationController
  def create
    @doc = Document.create!(file: params[:file])
    render json: { id: @doc.id }
  end
end
```

**Issues**
- No content-type/size validation; predictable filenames; potential malware; no authorization.

**Improved**
```ruby
class Document < ApplicationRecord
  has_one_attached :file
  validate :acceptable_file

  private
  def acceptable_file
    return unless file.attached?
    unless file.byte_size <= 10.megabytes
      errors.add(:file, 'is too big')
    end
    allowed = %w[application/pdf application/vnd.openxmlformats-officedocument.wordprocessingml.document]
    unless allowed.include?(file.content_type)
      errors.add(:file, 'type is not allowed')
    end
  end
end
```
```ruby
class DocumentsController < ApplicationController
  before_action :authenticate_user!

  def create
    doc = current_user.documents.create!(file: params.require(:file))
    render json: { id: doc.id }, status: :created
  end
end
```

**Why it works**: Validations enforce **type/size**; scoping to `current_user` avoids data leaks. Active Storage handles safe storage + randomized keys.

---

### Exercise 5: Caching with Explicit Invalidation
**Original**
```ruby
class TaxRulesController < ApplicationController
  def show
    render json: TaxRule.for_country(params[:country])
  end
end
```

**Issues**
- Expensive rule assembly on every request; no invalidation when rules change.

**Improved**
```ruby
class TaxRulesController < ApplicationController
  def show
    country = params[:country]
    version = TaxRule.version_for(country) # increments on rule update
    cache_key = ["tax_rules", country, version]

    payload = Rails.cache.fetch(cache_key, expires_in: 12.hours) do
      TaxRuleAssembler.build(country)
    end

    render json: payload
  end
end
```

**Why it works**: Versioned cache keys make invalidation **deterministic**; avoids stale rules without global cache clears.

---

### Exercise 6: Observability — Trace a Hot Path
**Original**
```ruby
class WebhooksController < ApplicationController
  def create
    Processor.new(params[:event]).call
    head :ok
  end
end
```

**Issues**
- No logs or metrics; debugging production incidents is guesswork.

**Improved**
```ruby
class WebhooksController < ApplicationController
  def create
    event = params.require(:event)
    trace_id = SecureRandom.uuid

    Rails.logger.info(event: 'webhook_received', type: event[:type], trace_id: trace_id)
    result = Processor.new(event, trace_id: trace_id).call

    StatsD.increment('webhooks.received', tags: ["type:#{event[:type]}"])
    StatsD.distribution('webhooks.processing_ms', result.duration_ms, tags: ["type:#{event[:type]}"])

    head :ok
  end
end
```

**Why it works**: Structured logs + metrics + trace id make incidents diagnosable and correlate across services.

---

## Part 3 — System Design Prompt (with a compact example answer)
**Prompt**: *Design a multi-region payroll calculation and payout service for EMEA + APAC with data residency constraints.*

**Compact Answer Outline**
1. **Data Residency**: Partition tenants by region; store PII in-region (`eu-west`, `ap-southeast`). Only **derived, anonymized aggregates** cross regions.
2. **Traffic Control**: Geo-DNS → region ingress → sticky routing by tenant. Graceful failover with read-only mode for non-critical features.
3. **Storage**: Postgres primary per region, read replicas for analytics. **Event log** (append-only) replicated cross-region with PII redaction.
4. **Compute**: Stateless Rails API; Sidekiq workers per region; **cron windows** tuned to payroll cycles. Idempotent jobs & retry with backoff.
5. **Security**: Field-level encryption (Lockbox) for sensitive columns; KMS for keys; audit trails and tamper-evident logs.
6. **Deployability**: Blue/green per region; config flags to gate risky flows; synthetic checks and SLO-driven rollouts.

**Why this fits Oyster**: Meets residency/compliance, limits blast radius, keeps auditability, and supports asynchronous, cohort-based releases.

---

## Part 4 — Delivery & Collaboration Aids
- **PR template**: Problem → Proposed change → Risks → Test plan → Rollout plan → Metrics → Owner.
- **STAR stories**: Prepare 4–5 with metrics (Reliability win, Performance win, Security incident, Cross-timezone delivery, Customer insight → feature).
- **Panel dynamics**: Keep answers **< 3 minutes**, end with a metric or a trade-off and a question back to panel (e.g., *“Do you prioritize error budgets or feature throughput during payroll week?”*).

---

## Appendix — 3 Ready-to-Use STAR Stories (skeletons)

1. **Reliability**
   - *S*: Payroll disbursement failures spiked during traffic burst.
   - *T*: Reduce failure rate and MTTR.
   - *A*: Added idempotency keys, queue backpressure, circuit breaker; on-call runbooks.
   - *R*: Failure rate −72%, MTTR 40→12 min, no customer data loss.

2. **Performance**
   - *S*: p95 latency for `/payrolls` at 1400ms due to N+1 and JSON serialization.
   - *T*: Hit p95 < 500ms.
   - *A*: Eager loads, pagination, precomputed views, Oj serializer.
   - *R*: p95 420ms, infra costs −18% from reduced DB CPU.

3. **Security/Compliance**
   - *S*: Audit finding: PII in logs and S3 object keys.
   - *T*: Eliminate PII exposure and meet DPIA recommendations.
   - *A*: Structured logging with redaction, random object keys, per-field encryption.
   - *R*: Cleared audit; zero PII leak incidents over 12 months.

---

## Checklist (night-before)
- [ ] 5 STAR stories with numbers you can defend.
- [ ] Run through the 6 exercises out loud (what, why, trade-offs).
- [ ] 2–3 crisp questions for the panel about reliability posture and async practices.
- [ ] Environment recap: feature flags, idempotency, residency, observability.

---

**Good luck!**
