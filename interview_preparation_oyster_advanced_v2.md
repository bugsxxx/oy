
# Oyster Panel Interview Preparation Guide (Advanced - New Examples)

## Oyster Context
Oyster operates globally, enabling remote work across multiple jurisdictions. Key challenges:
- **Compliance**: Handling tax, payroll, and GDPR.
- **Scalability**: Supporting thousands of concurrent users.
- **Security**: Protecting sensitive HR data.

---

## Part 1: 10 New High-Difficulty Questions Aligned with Competencies

### Engineering Excellence
**Q1: How would you design a Rails API to handle 1 million requests per hour with minimal latency?**
- Use Puma with clustered workers.
- Implement caching at multiple layers (HTTP, DB, application).
- Use CDN for static assets and edge caching.

**Q2: How do you ensure database migrations are safe in a high-traffic production environment?**
- Use strong_migrations gem.
- Break migrations into smaller steps.
- Perform zero-downtime migrations using background jobs.

### Communication
**Q3: How do you communicate complex architectural changes to a non-technical stakeholder?**
- Use diagrams and flowcharts.
- Explain trade-offs in terms of business impact.
- Provide incremental rollout plans.

### Customer Centricity
**Q4: How do you design features that anticipate compliance needs for global customers?**
- Research regional laws.
- Implement flexible rule engines.
- Provide audit logs and transparency.

### Impact & Results
**Q5: How do you measure the success of a performance optimization initiative?**
- Define KPIs (response time, throughput).
- Use A/B testing and monitoring tools.
- Validate improvements with customer feedback.

### Use of Data – Engineering
**Q6: How do you use logs and metrics to detect anomalies in a distributed Rails system?**
- Implement structured logging.
- Use anomaly detection algorithms.
- Integrate alerts with monitoring dashboards.

### Code & Architecture Quality
**Q7: How do you enforce modular architecture in a large Rails monolith?**
- Use Rails engines for domain separation.
- Apply service objects and POROs.
- Implement code ownership and review policies.

### Delivery
**Q8: How do you implement canary deployments in Rails?**
- Use feature flags for selective rollout.
- Deploy to a subset of servers.
- Monitor metrics before full release.

### How We Work
**Q9: How do you maintain async collaboration during incident response?**
- Use shared incident channels.
- Document steps in real-time.
- Schedule follow-up retrospectives.

### Advanced Topics
**Q10: How do you design a secure multi-tenant architecture for sensitive HR data?**
- Use separate schemas or databases per tenant.
- Encrypt data at rest and in transit.
- Implement strict RBAC and audit trails.

---

## Part 2: 4 New Advanced Code Improvement Exercises

### Exercise 1: Optimize Slow Query in Dashboard
**Original Code:**
```ruby
class DashboardController < ApplicationController
  def show
    @users = User.all
    @active_users = @users.select { |u| u.last_login > 30.days.ago }
    render json: { active_users: @active_users.count }
  end
end
```
**Issues:**
- Loads all users into memory.
- Inefficient filtering in Ruby.

**Improved Version:**
```ruby
class DashboardController < ApplicationController
  def show
    active_users_count = User.where("last_login > ?", 30.days.ago).count
    render json: { active_users: active_users_count }
  end
end
```
**Why it matters:**
- Uses SQL for filtering.
- Reduces memory usage and improves performance.

### Exercise 2: Secure API Key Handling
**Original Code:**
```ruby
class ApiController < ApplicationController
  def call_service
    api_key = ENV['API_KEY']
    response = HTTParty.get("https://service.com?key=#{api_key}")
    render json: response
  end
end
```
**Issues:**
- Exposes API key in logs and URLs.

**Improved Version:**
```ruby
class ApiController < ApplicationController
  def call_service
    api_key = Rails.application.credentials.api_key
    response = HTTParty.get("https://service.com", headers: { 'Authorization' => "Bearer #{api_key}" })
    render json: response
  end
end
```
**Why it matters:**
- Uses secure credentials.
- Avoids exposing sensitive data in URLs.

### Exercise 3: Distributed Job Scheduling
**Original Code:**
```ruby
class ReportJob < ApplicationJob
  def perform
    generate_report
  end
end
```
**Issues:**
- No scheduling logic.
- Risk of duplicate execution.

**Improved Version:**
```ruby
class ReportJob < ApplicationJob
  def perform
    lock_key = "report_job_lock"
    if Redis.current.set(lock_key, 1, nx: true, ex: 300)
      generate_report
    ensure
      Redis.current.del(lock_key)
    end
  end

  private

  def generate_report
    # Heavy report generation logic
  end
end
```
**Why it matters:**
- Prevents duplicate job execution.
- Ensures concurrency safety.

### Exercise 4: Advanced Caching Strategy
**Original Code:**
```ruby
class ProductsController < ApplicationController
  def index
    @products = Product.all
    render json: @products
  end
end
```
**Issues:**
- No caching.
- High DB load for frequent requests.

**Improved Version:**
```ruby
class ProductsController < ApplicationController
  def index
    products = Rails.cache.fetch("products_list", expires_in: 15.minutes) do
      Product.select(:id, :name, :price).to_a
    end
    render json: products
  end
end
```
**Why it matters:**
- Reduces DB load.
- Improves response time for frequent queries.

---

## Checklist
- Prepare examples for each competency.
- Review advanced Rails patterns and distributed system strategies.
- Practice implementing secure, scalable, and maintainable code.
