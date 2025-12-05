
# Oyster Panel Interview Preparation Guide (Advanced)

## Oyster Context
Oyster is a global HR platform enabling distributed teams. Key considerations:
- **Global Compliance**: GDPR, tax regulations, and employment laws.
- **Distributed Collaboration**: Async communication across time zones.
- **Scalability & Reliability**: Serving thousands of companies worldwide.

---

## Part 1: High-Difficulty Interview Questions Aligned with Competencies

### Engineering Excellence
**Q1: How would you design a Rails architecture to ensure reliability and scalability for global payroll processing?**
- Use microservices for critical components.
- Implement database sharding and read replicas.
- Use background jobs for heavy tasks and caching for performance.

### Communication
**Q2: How do you handle a misunderstanding in a distributed team when working asynchronously?**
- Summarize decisions in shared docs.
- Use clear commit messages and PR descriptions.
- Schedule overlap hours for critical discussions.

### Customer Centricity
**Q3: How do you proactively identify unspoken customer needs in a Rails project?**
- Analyze usage patterns and logs.
- Conduct interviews and gather feedback.
- Implement features that reduce friction (e.g., automated compliance checks).

### Impact & Results
**Q4: How do you adapt when a major requirement changes mid-project?**
- Reassess priorities and communicate trade-offs.
- Use feature flags for incremental delivery.
- Maintain focus on business outcomes.

### Use of Data – Engineering
**Q5: How do you leverage metrics to improve a Rails app’s performance?**
- Collect metrics via Prometheus/Grafana.
- Analyze slow queries and optimize indexes.
- Use A/B testing to validate improvements.

### Code & Architecture Quality
**Q6: How do you enforce code quality in a large Rails codebase?**
- Use RuboCop and automated linters.
- Implement CI/CD pipelines with test coverage checks.
- Conduct thorough code reviews.

### Delivery
**Q7: How do you ensure continuous delivery without compromising stability?**
- Use blue-green deployments.
- Implement rollback strategies.
- Automate tests and health checks.

### How We Work
**Q8: How do you maintain productivity in an async environment?**
- Document decisions in shared tools.
- Use structured communication (threads, templates).
- Prioritize clarity over speed.

### Advanced Topics
**Q9: How do you design a multi-region failover strategy for a Rails app?**
- Use DNS-based load balancing.
- Implement read replicas and async replication.
- Automate failover scripts.

**Q10: How do you secure sensitive data in a distributed Rails system?**
- Encrypt fields using Lockbox.
- Use secure key management (AWS KMS).
- Implement strict access controls and audit logs.

---

## Part 2: Advanced Code Improvement Exercises

### Exercise 1: Optimize N+1 Queries in API
**Original Code:**
```ruby
class OrdersController < ApplicationController
  def index
    orders = Order.all
    render json: orders.map { |o| { id: o.id, customer: o.customer.name } }
  end
end
```
**Issues:**
- N+1 query when accessing `customer.name`.
- No pagination.

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
- Eliminates N+1 queries.
- Adds pagination for scalability.

### Exercise 2: Secure File Upload
**Original Code:**
```ruby
class DocumentsController < ApplicationController
  def upload
    file = params[:file]
    File.open("uploads/#{file.original_filename}", "wb") { |f| f.write(file.read) }
  end
end
```
**Issues:**
- Hardcoded path.
- No validation or security checks.

**Improved Version:**
```ruby
class DocumentsController < ApplicationController
  def upload
    file = params[:file]
    raise "Invalid file type" unless %w[pdf docx].include?(file.content_type.split('/').last)

    path = Rails.root.join("storage", SecureRandom.hex + "_" + file.original_filename)
    File.open(path, "wb") { |f| f.write(file.read) }
  end
end
```
**Why it matters:**
- Prevents arbitrary file uploads.
- Uses secure paths and randomization.

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
- No actual rate limiting logic.

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
- Protects API from abuse.
- Uses Redis for distributed rate limiting.

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
- No concurrency control.
- Risk of duplicate processing.

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
- Prevents race conditions.
- Ensures safe execution in distributed environments.

---

## Checklist
- Prepare examples for each competency.
- Use STAR method for behavioral questions.
- Review Rails performance, security, and scalability patterns.
