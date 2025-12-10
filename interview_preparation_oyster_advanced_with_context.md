# Oyster Panel Interview Preparation Guide (Advanced) — with brief context

## Oyster Context
Oyster is a global HR platform enabling distributed teams. Key considerations:
- **Global Compliance**: GDPR, tax regulations, and employment laws. _Context: payroll and HR data is sensitive (PII/financial); apply data minimization, encryption-at-rest/in-transit, DPA/SCCs, and regional processing where required._
- **Distributed Collaboration**: Async communication across time zones. _Context: decisions captured in writing (ADRs/RFCs), clear SLAs for responses, and handover checklists to avoid blocking across regions._
- **Scalability & Reliability**: Serving thousands of companies worldwide. _Context: predictable latency across regions, graceful degradation during third-party outages (payroll/tax), and cost-aware autoscaling during peak cycles._

---

## Part 1: High-Difficulty Interview Questions Aligned with Competencies

### Engineering Excellence
**Q1: How would you design a Rails architecture to ensure reliability and scalability for global payroll processing?**
- Use microservices for critical components. _Context: isolate payroll calculation, compliance engines, and payments to limit blast radius and enable independent scaling._
- Implement database sharding and read replicas. _Context: shard by region/tenant to meet data residency and spread load; replicas serve read-heavy reporting without impacting writes._
- Use background jobs for heavy tasks and caching for performance. _Context: move long-running calculations to Sidekiq; cache stable reference data (tax tables) with explicit invalidation on updates._

### Communication
**Q2: How do you handle a misunderstanding in a distributed team when working asynchronously?**
- Summarize decisions in shared docs. _Context: post a short ADR with options, decision, and trade-offs so future teammates can onboard quickly._
- Use clear commit messages and PR descriptions. _Context: include problem statement, approach, and testing notes to reduce review cycles across time zones._
- Schedule overlap hours for critical discussions. _Context: time-box synchronous calls for contentious decisions; default to async for status updates._

### Customer Centricity
**Q3: How do you proactively identify unspoken customer needs in a Rails project?**
- Analyze usage patterns and logs. _Context: instrument funnels (onboarding → first payroll) to spot drop-offs and hidden friction._
- Conduct interviews and gather feedback. _Context: partner with CS/Support to classify top pain points and translate them into measurable hypotheses._
- Implement features that reduce friction (e.g., automated compliance checks). _Context: pre-validate country-specific requirements to shorten time-to-first-success._

### Impact & Results
**Q4: How do you adapt when a major requirement changes mid-project?**
- Reassess priorities and communicate trade-offs. _Context: share scope/impact matrix and update roadmap; keep risky changes behind flags._
- Use feature flags for incremental delivery. _Context: progressive rollout by tenant/region allows quick rollback without redeploys._
- Maintain focus on business outcomes. _Context: tie each scope change to metrics (activation, error rate, SLA) and stop low-impact work._

### Use of Data – Engineering
**Q5: How do you leverage metrics to improve a Rails app’s performance?**
- Collect metrics via Prometheus/Grafana. _Context: track RED/Golden Signals per endpoint and per region (p95 latency, error rate, throughput)._ 
- Analyze slow queries and optimize indexes. _Context: use EXPLAIN/ANALYZE; add composite/partial indexes tied to real filters (tenant_id, status)._ 
- Use A/B testing to validate improvements. _Context: verify real impact on time-to-value and success rates, not just raw latency._

### Code & Architecture Quality
**Q6: How do you enforce code quality in a large Rails codebase?**
- Use RuboCop and automated linters. _Context: standardize style and prevent common pitfalls before review._
- Implement CI/CD pipelines with test coverage checks. _Context: fast feedback; block merges on flaky or slow tests to maintain trust in the pipeline._
- Conduct thorough code reviews. _Context: focus on readability, coupling/cohesion, and explicit acceptance criteria; capture decisions in PRs._

### Delivery
**Q7: How do you ensure continuous delivery without compromising stability?**
- Use blue-green deployments. _Context: enable zero-downtime deploys and safe DB migrations (dual-write/read strategies)._ 
- Implement rollback strategies. _Context: feature flags and reversible migrations keep reversibility high._
- Automate tests and health checks. _Context: pre-merge checks and post-deploy synthetic probes by region._

### How We Work
**Q8: How do you maintain productivity in an async environment?**
- Document decisions in shared tools. _Context: centralize in a discoverable space; link issues, ADRs, and runbooks._
- Use structured communication (threads, templates). _Context: standardize request formats (context, proposal, timeline) to reduce back-and-forth._
- Prioritize clarity over speed. _Context: write to be understood later; accept slower initial responses for faster overall throughput._

### Advanced Topics
**Q9: How do you design a multi-region failover strategy for a Rails app?**
- Use DNS-based load balancing. _Context: route traffic across regions; support region stickiness for session/state constraints._
- Implement read replicas and async replication. _Context: prefer eventual consistency for reporting; for writes, plan for controlled failover with RPO/RTO targets._
- Automate failover scripts. _Context: playbooks and gamedays ensure people and tooling are ready for real incidents._

**Q10: How do you secure sensitive data in a distributed Rails system?**
- Encrypt fields using Lockbox. _Context: protect PII (addresses, bank info) with per-attr encryption and key rotation._
- Use secure key management (AWS KMS). _Context: centralize encryption keys and audit access; avoid keys in env variables._
- Implement strict access controls and audit logs. _Context: least-privilege policies in app and DB; immutable audit trails for compliance._

---

## Part 2: Advanced Code Improvement Exercises

### Exercise 1: Optimize N+1 Queries in API
**Original Code:**
```ruby
class OrdersController < ApplicationController
  def index
    orders = Order.all
    render json: orders.map { |o|
      { id: o.id, customer: o.customer.name }
    }
  end
end
```
**Issues:**
- N+1 query when accessing `customer.name`. _Context: blows up latency at scale; noisy DB under high concurrency._
- No pagination. _Context: unbounded responses increase memory/time; poor mobile/network performance._

**Improved Version:**
```ruby
class OrdersController < ApplicationController
  def index
    orders = Order.includes(:customer).page(params[:page]).per(20)
    render json: orders.as_json(only: [:id], include: { customer: { only: [:name] } })
  end
end
```
**Why it matters:**
- Eliminates N+1 queries. _Context: predictable query count regardless of list size._
- Adds pagination for scalability. _Context: keeps endpoints fast and affordable for global clients._

---

### Exercise 2: Secure File Upload
**Original Code:**
```ruby
class DocumentsController < ApplicationController
  def upload
    file = params[:file]
    File.open("uploads/#{file.original_filename}", "wb") { |f|
      f.write(file.read) }
  end
end
```
**Issues:**
- Hardcoded path. _Context: path traversal risk and inconsistent environments._
- No validation or security checks. _Context: malware, over-sized files, or unexpected MIME types._

**Improved Version:**
```ruby
class DocumentsController < ApplicationController
  def upload
    file = params[:file]
    allowed = %w[pdf docx]
    ext = File.extname(file.original_filename).delete('.')
    raise ActionController::BadRequest, 'Invalid file type' unless allowed.include?(ext)

    path = Rails.root.join('storage', "#{SecureRandom.hex}_#{file.original_filename}")
    File.open(path, 'wb') { |f| f.write(file.read) }
    head :created
  end
end
```
**Why it matters:**
- Prevents arbitrary file uploads. _Context: reduces risk of executing or serving unsafe files._
- Uses secure paths and randomization. _Context: prevents collisions and information disclosure via filenames._

---

### Exercise 3: Implement Rate Limiting Middleware
**Original Code:**
```ruby
class ApplicationController < ActionController::Base
  before_action :check_rate_limit
  def check_rate_limit
    # TODO: implement
  end
end
```
**Issues:**
- No actual rate limiting logic. _Context: endpoints vulnerable to abuse and noisy neighbors._

**Improved Version:**
```ruby
class RateLimiter
  def initialize(app)
    @app = app
  end
  def call(env)
    ip = env['REMOTE_ADDR']
    key = "rate_limit:#{ip}"
    count = Redis.current.incr(key)
    Redis.current.expire(key, 60) if count == 1
    if count > 100
      [429, { 'Content-Type' => 'text/plain' }, ['Too Many Requests']]
    else
      @app.call(env)
    end
  end
end
```
**Why it matters:**
- Protects API from abuse. _Context: preserves capacity for legitimate users during spikes._
- Uses Redis for distributed rate limiting. _Context: works across multiple app servers/regions._

---

### Exercise 4: Concurrency-Safe Background Job
**Original Code:**
```ruby
class ReportJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    generate_report(user)
  end
end
```
**Issues:**
- No concurrency control. _Context: duplicate jobs may race and overwhelm downstream systems._
- Risk of duplicate processing. _Context: users may get multiple reports or inconsistent state._

**Improved Version:**
```ruby
class ReportJob < ApplicationJob
  def perform(user_id)
    lock_key = "lock:user:#{user_id}"
    if Redis.current.set(lock_key, 1, nx: true, ex: 60)
      user = User.find(user_id)
      generate_report(user)
    ensure
      Redis.current.del(lock_key)
    end
  end
  private
  def generate_report(user)
    # Generate report logic
  end
end
```
**Why it matters:**
- Prevents race conditions. _Context: guarantees one active job per user/report._
- Ensures safe execution in distributed environments. _Context: resilient to retries and worker restarts._

---

## Checklist
- Prepare examples for each competency. _Context: pick 3–5 STAR stories with metrics and trade-offs._
- Use STAR method for behavioral questions. _Context: keep each answer <3 minutes with measurable results._
- Review Rails performance, security, and scalability patterns. _Context: focus on N+1, locking/transactions, caching, idempotency, observability._
