# Architecture Decisions & Trade-offs

## 📝 Overview

As a Tech Lead, one of your most critical responsibilities is making sound architectural decisions. This involves evaluating trade-offs, documenting decisions, and ensuring the team understands and follows the chosen direction.

**Key Principle**: There are no "perfect" architectures, only informed trade-offs based on your specific context.

---

## 🎯 Core Concepts

### What Makes a Good Architecture Decision?

1. **Context-Aware**: Fits your specific constraints (team, timeline, scale)
2. **Well-Documented**: Clear reasoning that survives team changes
3. **Reversible Where Possible**: Avoid one-way doors when you can
4. **Team Buy-in**: Engineers understand and support it
5. **Measurable**: Can verify it's solving the problem

### Types of Architectural Decisions

```
Strategic (Long-term):
- Microservices vs. Monolith
- Cloud provider selection
- Programming language/framework
- Database technology

Tactical (Medium-term):
- API design patterns
- Caching strategy
- Authentication approach
- Deployment strategy

Implementation (Short-term):
- Library choices
- Code organization
- Testing approach
- Error handling patterns
```

---

## 🏗️ Architecture Decision Records (ADRs)

### What is an ADR?

A lightweight document capturing important architectural decisions and their context.

### ADR Template

```markdown
# ADR-001: Use PostgreSQL for Primary Database

## Status
Accepted

## Context
We need to choose a database for our new e-commerce platform.

Requirements:
- ACID transactions for orders and payments
- Complex queries for reporting
- Relational data (users, products, orders)
- Expected scale: 10K orders/day initially, 100K in 2 years
- Team has SQL experience, no NoSQL experience
- Budget: Prefer open-source

Options Considered:
1. PostgreSQL
2. MySQL
3. MongoDB
4. DynamoDB

## Decision
We will use PostgreSQL as our primary database.

## Rationale

### Pros
- ✅ ACID compliance (critical for financial transactions)
- ✅ Excellent support for complex queries and joins
- ✅ JSON support for flexibility where needed
- ✅ Strong consistency guarantees
- ✅ Team has 5+ years PostgreSQL experience
- ✅ Rich ecosystem and tooling
- ✅ Open source with managed options (RDS, Cloud SQL)

### Cons
- ❌ Vertical scaling limitations (addressed below)
- ❌ More complex setup than managed NoSQL

### Trade-offs Accepted
- **Scalability**: Accepting vertical scaling limits initially
  - Mitigation: Read replicas for scaling reads
  - Future: Can add caching layer, partition if needed

- **Flexibility**: Stricter schema vs. NoSQL flexibility
  - Mitigation: Use JSONB columns where schema is evolving

## Alternatives Considered

### MySQL
- Similar pros/cons to PostgreSQL
- Rejected: PostgreSQL has better JSON support and advanced features we'll need

### MongoDB
- Pro: Flexible schema, horizontal scaling
- Rejected: No ACID transactions at time of decision (2019), team expertise gap

### DynamoDB
- Pro: Fully managed, excellent scalability
- Rejected: Learning curve, limited query flexibility, vendor lock-in concerns

## Consequences

### Positive
- Development velocity from team expertise
- Strong data integrity
- Flexible enough for our needs

### Negative
- Will need to address scaling proactively
- Operational overhead vs. fully managed solutions

### Action Items
- [ ] Set up read replicas from day one
- [ ] Monitor query performance and set up alerts
- [ ] Document schema migration process
- [ ] Plan for connection pooling (PgBouncer)

## Review Date
Review this decision in 12 months or when we hit 50K orders/day.

## References
- PostgreSQL Documentation: https://www.postgresql.org/docs/
- Internal Wiki: Database Standards
- Benchmark Results: [link to internal doc]

---
**Date**: 2025-01-15
**Author**: Tech Lead - John Smith
**Stakeholders**: CTO, Backend Team, DevOps
```

### When to Write an ADR

✅ **DO write an ADR for**:
- Technology selections (languages, frameworks, databases)
- Significant architecture patterns (microservices, event-driven, etc.)
- API design standards
- Security approach
- Data modeling decisions
- Cross-cutting concerns (logging, monitoring, error handling)

❌ **DON'T write an ADR for**:
- Trivial library choices that can be easily swapped
- Coding style (use linters/formatters instead)
- Temporary workarounds
- Decisions that are easily reversible with low cost

---

## ⚖️ Decision-Making Frameworks

### 1. The Weighted Decision Matrix

Use when comparing multiple options against clear criteria.

```markdown
## Decision Matrix: Caching Solution

| Criteria | Weight | Redis | Memcached | In-Memory | Hazelcast |
|----------|--------|-------|-----------|-----------|-----------|
| **Performance** | 30% | 9 | 10 | 10 | 8 |
| **Persistence** | 25% | 10 | 0 | 0 | 8 |
| **Team Expertise** | 20% | 8 | 6 | 10 | 2 |
| **Scalability** | 15% | 9 | 9 | 4 | 10 |
| **Cost** | 10% | 7 | 8 | 10 | 5 |
| **Weighted Score** | - | **8.5** | 6.7 | 7.4 | 6.6 |

Decision: **Redis** - Best overall fit given our need for persistence and team expertise.
```

### 2. The Pre-Mortem Technique

Imagine the decision failed spectacularly. What went wrong?

```markdown
## Pre-Mortem: Microservices Migration

"It's 1 year from now. Our microservices migration failed. What happened?"

Potential Failures:
1. ❌ "Team couldn't handle operational complexity - too many services down"
   - Mitigation: Start with 3 services, heavy investment in monitoring

2. ❌ "Distributed transactions caused data inconsistency"
   - Mitigation: Use eventual consistency patterns, saga pattern

3. ❌ "Network issues killed performance"
   - Mitigation: Careful service boundaries, batch operations

4. ❌ "Team lost 6 months rewriting everything"
   - Mitigation: Strangler pattern - incremental migration

5. ❌ "DevOps couldn't keep up with deployment complexity"
   - Mitigation: Invest in CI/CD automation first
```

### 3. One-Way vs. Two-Way Doors (Jeff Bezos)

**One-Way Doors**: Hard to reverse, need careful analysis
- Technology platform changes
- Database choices
- Programming language selection
- Major architecture patterns

**Two-Way Doors**: Easy to reverse, move fast
- Library choices
- Feature flags
- Internal API design
- Refactoring approaches

**Strategy**:
- One-way doors: Gather data, write ADR, get stakeholder input
- Two-way doors: Make quick decision, try it, iterate

---

## 💡 Common Architecture Scenarios

### Scenario 1: Monolith vs. Microservices

**Context**: Building a new e-commerce platform

#### Decision Framework

```markdown
## Choose Monolith If:
- ✅ Small team (< 10 engineers)
- ✅ Tight timeline (< 6 months to launch)
- ✅ Limited DevOps maturity
- ✅ Unclear domain boundaries
- ✅ Low initial scale requirements

## Choose Microservices If:
- ✅ Large team (> 20 engineers)
- ✅ Well-understood domain
- ✅ Independent scaling needs for different components
- ✅ Different technology requirements per service
- ✅ Strong DevOps capabilities

## Hybrid Approach (Recommended for Many):
Start with **Modular Monolith**:
- Clear module boundaries
- Separate databases per module (if possible)
- Module-level APIs
- Easy to extract to microservices later

Benefits:
- ✅ Fast initial development
- ✅ Simple deployment
- ✅ Clear path to microservices when needed
```

#### Real Example

```csharp
// Modular Monolith Structure
// Each module can later become a microservice

// 1. Clear module boundaries
namespace ECommerce.Catalog
{
    public interface ICatalogService
    {
        Task<Product> GetProductAsync(int id);
        Task<IEnumerable<Product>> SearchProductsAsync(string query);
    }

    // Internal implementation
    internal class CatalogService : ICatalogService
    {
        private readonly CatalogDbContext _dbContext;
        // Only Catalog module accesses Catalog database
    }
}

namespace ECommerce.Orders
{
    public interface IOrderService
    {
        Task<Order> CreateOrderAsync(CreateOrderRequest request);
    }

    internal class OrderService : IOrderService
    {
        private readonly OrderDbContext _dbContext;
        private readonly ICatalogService _catalogService; // Cross-module via interface

        public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
        {
            // Get product info through interface (easy to replace with HTTP call later)
            var product = await _catalogService.GetProductAsync(request.ProductId);

            // Business logic
        }
    }
}

// Startup configuration
public void ConfigureServices(IServiceCollection services)
{
    // Each module registers its services
    services.AddCatalogModule();
    services.AddOrdersModule();
    services.AddPaymentsModule();
}
```

---

### Scenario 2: SQL vs. NoSQL

**Context**: Choosing database for user analytics system

#### Decision Matrix

```markdown
## User Analytics Requirements:
- High write volume (10K events/second)
- Flexible schema (event properties vary)
- Time-series data
- Aggregation queries
- Retention: 90 days hot, 2 years archive

## Evaluation:

### PostgreSQL (JSONB)
**Pros**:
- ACID transactions
- Complex queries
- Team expertise
- JSONB for flexible schema

**Cons**:
- Write scaling challenges
- More expensive at scale
- Complex partitioning needed

**Score**: 6/10

### MongoDB
**Pros**:
- Flexible schema (natural fit)
- Horizontal scaling
- Good for write-heavy

**Cons**:
- Aggregation queries complex
- Team learning curve
- Operational complexity

**Score**: 7/10

### ClickHouse (Time-Series DB)
**Pros**:
- Optimized for analytics
- Excellent write performance
- Great for time-series
- Compression for storage

**Cons**:
- Team has no experience
- Limited ecosystem
- No transactions

**Score**: 8/10

### DynamoDB
**Pros**:
- Fully managed
- Excellent scaling
- Flexible schema

**Cons**:
- Limited query capabilities
- Cost at scale
- Vendor lock-in

**Score**: 7/10

## Decision: ClickHouse
Despite learning curve, it's purpose-built for our use case.

## Mitigation:
- 2-week POC to validate
- Dedicated training for team
- Start with single use case
- Keep PostgreSQL for transactional data
```

---

### Scenario 3: REST vs. GraphQL vs. gRPC

```markdown
## Context: API for mobile and web clients

### REST
**Best For**: Public APIs, CRUD operations, simple clients
**Pros**: Universal, cacheable, well-understood
**Cons**: Over-fetching, multiple round trips
**Use When**: External partners, simple mobile apps

### GraphQL
**Best For**: Complex UIs, varied client needs
**Pros**: Exact data fetching, strong typing, single endpoint
**Cons**: Caching complexity, query cost analysis needed
**Use When**: Multiple diverse clients, evolving requirements

### gRPC
**Best For**: Internal service-to-service, high performance
**Pros**: Binary protocol, streaming, type safety
**Cons**: Browser support limited, debugging harder
**Use When**: Microservice communication, real-time data

## Our Decision: Multi-Protocol Approach
- **REST**: Public API for partners
- **GraphQL**: Mobile and web apps
- **gRPC**: Internal service communication

## Reasoning:
Use the right tool for each use case rather than one-size-fits-all.
```

---

## 🎤 Common Interview Questions

### Q1: "Walk me through a significant architectural decision you made."

**STAR Example**:

**Situation**:
"We had a monolithic .NET application handling 100K requests/day. The marketing team and checkout components had very different scaling needs - marketing had traffic spikes during campaigns, while checkout was steady."

**Task**:
"Determine if we should migrate to microservices or optimize the monolith."

**Action**:
"I used a structured decision process:

**1. Gathered Data**:
- Profiled application: Marketing used 60% of resources during spikes
- Interviewed teams: 3 teams, different deployment schedules
- Analyzed failures: Most outages from marketing deployments affecting checkout

**2. Created Decision Matrix**:
| Criteria | Weight | Monolith | Full Microservices | Hybrid |
|----------|--------|----------|-------------------|--------|
| Time to Market | 30% | 9 | 3 | 7 |
| Risk | 25% | 7 | 4 | 8 |
| Team Capability | 20% | 9 | 5 | 8 |
| Solves Problem | 25% | 5 | 10 | 9 |

**3. Decision: Hybrid Approach**
- Extract marketing to separate service
- Keep core e-commerce as modular monolith
- Plan to extract checkout next if successful

**4. Documented in ADR**
- Clear rationale
- Success metrics
- Rollback plan

**5. Execution**:
- Strangler pattern over 3 months
- Feature flags for gradual rollout
- Monitoring before, during, after"

**Result**:
"3 months later:
- Marketing deployments no longer affected checkout (99.9% → 99.99% uptime)
- Marketing could scale independently during campaigns
- 40% reduction in infrastructure costs (right-sized each service)
- Zero customer-impacting incidents during migration

Based on success, extracted 2 more services next quarter."

**Learning**:
"I learned that incremental migration with clear metrics beats big-bang rewrites. Also, starting with the highest pain point built team confidence for future migrations."

---

### Q2: "How do you make technical decisions when there's no clear right answer?"

**Good Answer**:

"I use a framework for ambiguous decisions:

**1. Define Success Criteria**
What problem are we solving? What does success look like?

**2. Identify Constraints**
- Timeline
- Team skills
- Budget
- Scale requirements
- Technical debt we can accept

**3. Gather Data**
- Benchmarks
- POCs for critical unknowns
- Industry case studies
- Team input

**4. Analyze Trade-offs**
Every decision has trade-offs. Make them explicit:
- 'Approach A is faster to build but harder to scale'
- 'Approach B is more scalable but requires 3-month learning curve'

**5. Use Decision Framework**
- Weighted decision matrix for 3+ options
- Pre-mortem: Imagine it failed, why?
- Reversibility: Is this a one-way door?

**6. Document with ADR**
- Context, decision, rationale, alternatives, consequences
- Helps future team understand WHY

**7. Set Review Points**
'Let's try this for 2 sprints and review'

**Example**:
Choosing between REST and GraphQL for a new API.

No clear winner - both had merits.

**My Process**:
- Built POC of both (2 days each)
- Had team vote on which they preferred
- Analyzed maintenance burden
- Considered client team's preference

**Decision**: GraphQL
- Better for our evolving mobile app needs
- Team preferred it after POC
- Willing to accept learning curve

**Safety Net**:
- Kept internal REST endpoints
- Could add REST facade if needed
- Reviewed after 2 months

**Result**: Successful, no regrets.

**Key**: When there's no perfect answer, make the best decision with available information, document it, and set review points."

---

### Q3: "Tell me about a time an architectural decision you made didn't work out."

**STAR Example** (Shows humility and learning):

**Situation**:
"Two years ago, I decided we should adopt a message queue (RabbitMQ) for all inter-service communication to improve reliability."

**Task**:
"We had issues with services going down and requests being lost. I wanted to add resilience."

**Decision**:
"I mandated all service calls go through RabbitMQ instead of direct HTTP calls."

**What Went Wrong**:
"After 3 months:
- System became significantly more complex
- Debugging was much harder (messages lost in queues)
- Latency increased from 50ms to 200ms for simple calls
- Team spent more time fighting RabbitMQ than building features
- We had added complexity without clear benefit for most use cases"

**Action to Fix**:
"I realized I had over-engineered the solution.

**1. Acknowledged the Mistake**:
Told the team: 'I pushed this decision and it's not working. That's on me.'

**2. Analyzed the Problem**:
- Only 2 out of 10 services actually needed async messaging
- Synchronous calls were fine for most cases
- We needed circuit breakers, not queues

**3. Corrected Course**:
- Kept RabbitMQ for the 2 services that benefited (background jobs)
- Reverted others to HTTP with circuit breakers (Polly)
- Documented when to use each pattern

**4. Established Principle**:
'Use asynchronous messaging only when you need: background processing, fire-and-forget, or delayed processing. Default to simplest solution.'"

**Result**:
"- Latency back to normal
- System complexity reduced
- Team happier
- Appropriate use of message queues where beneficial

More importantly, team saw I could admit mistakes and change course."

**Learning**:
"Three key lessons:
1. **Start simple**: Don't over-engineer upfront
2. **Listen to feedback**: Team was saying it was too complex
3. **No ego**: Wrong decision + quick fix > stubbornly sticking with bad decision

Now I use the principle: 'Choose the simplest solution that could work, then iterate.'"

---

## 📚 Decision-Making Checklist

### Before Making an Architecture Decision

- [ ] Have I clearly defined the problem?
- [ ] Do I understand the constraints (time, team, budget)?
- [ ] Have I consulted team members who will implement it?
- [ ] Have I researched how others solved similar problems?
- [ ] Have I considered at least 3 alternatives?
- [ ] Have I made the trade-offs explicit?
- [ ] Is this a one-way door or two-way door?
- [ ] Can we test this on a small scale first?
- [ ] Will I document this decision (ADR)?
- [ ] Have I set review points to validate?

### Red Flags in Decision Making

- 🚩 Choosing technology because it's trendy
- 🚩 Not consulting the team
- 🚩 Ignoring constraints ("we'll figure it out later")
- 🚩 No documented rationale
- 🚩 No plan B or rollback strategy
- 🚩 Solving problems you don't have
- 🚩 Not considering operational burden
- 🚩 Ego-driven decisions ("I want to use X")

---

## 🔗 Related Topics
- [System Design Trade-offs](../system-design/trade-offs-guide.md)
- [Code Review Practices](./code-review-practices.md)
- [Technical Debt Management](./technical-debt.md)
- [Technology Selection](./technology-selection.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐☆☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
