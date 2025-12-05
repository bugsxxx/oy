
# Comprehensive Senior Ruby on Rails Developer Interview Preparation Guide

## Part 1: Mock Interview Questions + Answers

### Basic Questions
**Q1: How do you ensure scalability in a Rails application?**
- Use database indexing and optimize queries.
- Implement caching (Rails cache, Redis).
- Use background jobs for heavy tasks (Sidekiq).
- Horizontal scaling with load balancers and read replicas.
- Monitor performance with tools like New Relic or Skylight.

**Q2: How do you handle misunderstandings in a remote team?**
- Confirm understanding by summarizing decisions in writing.
- Use async tools like Slack threads or Notion docs for clarity.
- Encourage video calls for complex topics when needed.

**Q3: Tell me about a time you discovered an unspoken customer need.**
- Example: While implementing a payroll feature, noticed customers struggled with tax compliance. Proposed an automated tax calculation module, reducing errors and saving time.

**Q4: How do you measure success in your projects?**
- Define KPIs upfront (e.g., response time, error rate).
- Use data-driven metrics from logs and monitoring tools.
- Validate with customer feedback and adoption rates.

### Expanded Questions
**Q5: How do you handle performance bottlenecks in ActiveRecord?**
- Use includes to avoid N+1 queries.
- Optimize queries with select and pluck.
- Add proper indexes.
- Use database-level optimizations.

**Q6: How do you explain technical concepts to non-technical stakeholders?**
- Use analogies and simple language.
- Focus on business impact rather than technical jargon.
- Provide visual aids (diagrams, charts) when possible.

**Q7: How do you balance technical debt with delivering customer value?**
- Prioritize fixes that impact user experience.
- Schedule refactoring alongside feature development.
- Communicate trade-offs clearly to stakeholders.

### Advanced Questions
**Q8: How would you design a multi-tenant Rails application ensuring data isolation and scalability?**
- Use PostgreSQL schemas or separate databases per tenant.
- Implement Row-Level Security for strict isolation.
- Use connection pooling and caching strategies.
- Ensure indexing and query optimization for large datasets.

**Q9: How do you handle concurrency in Rails when multiple processes update the same record?**
- Use optimistic locking with ActiveRecord’s lock_version.
- For critical sections, use database transactions and SELECT FOR UPDATE.
- Consider Redis-based distributed locks for cross-process safety.

**Q10: How do you ensure global compliance (GDPR) in a Rails app handling sensitive data?**
- Encrypt sensitive fields using attr_encrypted or Lockbox.
- Implement data retention policies and anonymization.
- Ensure audit trails and consent management.

### Hardest Questions
**Q11: How do you ensure zero-downtime deployments in Rails?**
- Use rolling deployments with Puma phased restart.
- Preload assets and run migrations safely using strong_migrations.
- Use feature flags for gradual rollouts.
- Implement health checks and blue-green deployments for failover.

**Q12: How do you use A/B testing in Rails to validate feature impact?**
- Use gems like Split or build a custom experiment framework.
- Collect metrics in Redis or PostgreSQL for fast reads.
- Analyze conversion rates and ensure statistical significance before rollout.

**Q13: How do you implement background job retries with exponential backoff in Sidekiq?**
- Configure Sidekiq retry options with custom logic.
- Use middleware to implement exponential delay between retries.
- Log failures and alert via monitoring tools (e.g., Sentry, Honeybadger).

---

## Part 2: Code Improvement Exercises with Explanations

### Exercise 1: Basic Refactor
```ruby
class OrdersController < ApplicationController
  def index
    @orders = Order.all
    @total_price = 0
    @orders.each do |order|
      @total_price += order.price
    end
    render json: { orders: @orders, total_price: @total_price }
  end
end
```
**Improved Version:**
```ruby
class OrdersController < ApplicationController
  def index
    orders = Order.includes(:customer).page(params[:page]).per(20)
    total_price = orders.sum(:price)

    render json: {
      orders: orders.as_json(only: [:id, :price], include: { customer: { only: [:name] } }),
      total_price: total_price
    }
  end
end
```
**Why these changes matter:**
- Pagination prevents loading all records into memory.
- `sum(:price)` uses SQL aggregation instead of Ruby loops.
- `includes(:customer)` avoids N+1 queries.
- `as_json` ensures clean serialization.

### Exercise 2: Service Object Refactor
```ruby
class ReportGenerator
  def generate(user_id)
    user = User.find(user_id)
    report = "Name: " + user.name + "\nEmail: " + user.email
    File.open("report.txt", "w") { |f| f.write(report) }
  end
end
```
**Improved Version:**
```ruby
class ReportGenerator
  def generate(user_id)
    user = User.find(user_id)
    report = "Name: #{user.name}\nEmail: #{user.email}"
    File.write(Rails.root.join("tmp", "report.txt"), report)
  rescue ActiveRecord::RecordNotFound
    Rails.logger.error("User not found: #{user_id}")
  end
end
```
**Why these changes matter:**
- String interpolation is cleaner.
- Rails.root.join avoids hardcoded paths.
- Error handling prevents crashes.

### Exercise 3: Advanced Refactor with Caching
```ruby
class DashboardController < ApplicationController
  def show
    @stats = StatsService.new.calculate
    render json: @stats
  end
end

class StatsService
  def calculate
    users = User.all
    posts = Post.all
    comments = Comment.all
    { users: users.count, posts: posts.count, comments: comments.count }
  end
end
```
**Improved Version:**
```ruby
class DashboardController < ApplicationController
  def show
    stats = Rails.cache.fetch("dashboard_stats", expires_in: 10.minutes) do
      StatsService.new.calculate
    end
    render json: stats
  end
end

class StatsService
  def calculate
    {
      users: User.count,
      posts: Post.count,
      comments: Comment.count
    }
  end
end
```
**Why these changes matter:**
- Rails.cache reduces DB load.
- Short expiration ensures freshness.
- Direct count queries avoid loading entire tables.

### Exercise 4: Transactional Safety
```ruby
class InvoiceProcessor
  def process(invoice_id)
    invoice = Invoice.find(invoice_id)
    invoice.update(status: "processed")
    send_email(invoice.user.email)
  end

  def send_email(email)
    Mailer.send(email)
  end
end
```
**Improved Version:**
```ruby
class InvoiceProcessor
  def process(invoice_id)
    ActiveRecord::Base.transaction do
      invoice = Invoice.lock.find(invoice_id)
      invoice.update!(status: "processed")
      NotificationService.new.send_email(invoice.user.email)
    end
  rescue ActiveRecord::RecordInvalid => e
    Rails.logger.error("Invoice processing failed: #{e.message}")
  end
end

class NotificationService
  def send_email(email)
    Mailer.send(email)
  end
end
```
**Why these changes matter:**
- Transactions ensure atomicity.
- Lock prevents race conditions.
- Service object improves separation of concerns.

### Exercise 5: Background Job with File Handling
```ruby
class ReportJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    report = generate_report(user)
    File.open("report.txt", "w") { |f| f.write(report) }
  end

  def generate_report(user)
    "Name: #{user.name}\nEmail: #{user.email}"
  end
end
```
**Improved Version:**
```ruby
class ReportJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)
    report = generate_report(user)
    path = Rails.root.join("tmp", "reports", "report_#{user_id}.txt")
    File.write(path, report)
  rescue ActiveRecord::RecordNotFound
    Rails.logger.error("User not found: #{user_id}")
  end

  private

  def generate_report(user)
    "Name: #{user.name}\nEmail: #{user.email}"
  end
end
```
**Why these changes matter:**
- Rails.root.join ensures correct paths.
- Error handling prevents job failures.
- Private method encapsulates logic.

---

## Part 3: Interview Checklist & Async Collaboration Tips
- Prepare clear examples for each competency area.
- Use STAR method (Situation, Task, Action, Result) for behavioral questions.
- Communicate trade-offs clearly and tie them to business impact.
- Document decisions and assumptions during the code exercise.
- For async collaboration: use shared docs, clear commit messages, and structured PRs.

## Part 4: Oyster-Specific Context & Study Resources
- Oyster is a global HR platform: expect questions on time zones, compliance, and distributed systems.
- Study Rails Guides: Security, Performance, ActiveRecord Querying.
- Learn Sidekiq, Redis, and caching strategies.
- Practice writing RSpec tests and implementing CI/CD pipelines.
