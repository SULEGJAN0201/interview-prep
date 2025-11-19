# Tech Lead Leadership Scenarios

## 📝 Overview

These are **realistic scenarios** you'll face as a Tech Lead. Interviewers use these to assess your judgment, communication, and leadership skills.

**Key to Success**: Use the **STAR method** (Situation, Task, Action, Result) and show:
- How you analyze the situation
- Multiple options you considered
- Why you chose your approach
- The outcome and what you learned

---

## 🎯 Team Conflicts & Dynamics

### Scenario 1: Two Senior Engineers Disagree on Architecture

**Question**: "Two senior engineers on your team strongly disagree about whether to use microservices or a monolith for a new project. The disagreement is causing tension. How do you handle this?"

**Good Answer**:

"This is a common scenario. Here's how I'd approach it:

**Step 1: Understand Both Perspectives**

I'd schedule separate 1:1s with each engineer:

*Engineer A (Microservices advocate)*:
- 'Why do you believe microservices is the right choice?'
- 'What problems does it solve?'
- 'What are the risks/costs?'

*Engineer B (Monolith advocate)*:
- Same questions

**Goal**: Understand their technical reasoning AND emotional investment

---

**Step 2: Define Decision Criteria**

Bring them together and establish **objective criteria**:

```markdown
## Decision Criteria (Weighted)

1. **Team Size/Expertise** (25%)
   - Do we have skills for distributed systems?
   - Can we support multiple services?

2. **Scale Requirements** (20%)
   - Current: 1K users
   - Target: 10K users in 12 months
   - Need: Independent scaling?

3. **Time to Market** (20%)
   - Deadline: 3 months
   - Monolith: Faster initial delivery
   - Microservices: Slower initial, faster later

4. **Deployment Complexity** (15%)
   - Current: Manual deploys (2 hrs)
   - Future: CI/CD pipeline?

5. **Cost** (10%)
   - Infrastructure cost
   - Development cost

6. **Risk** (10%)
   - What if we're wrong?
   - Can we change later?
```

---

**Step 3: Evaluate Together**

Facilitate a meeting where both engineers score each approach:

| Criteria | Monolith | Microservices | Weight |
|----------|----------|---------------|--------|
| Team Size (6 devs, moderate skills) | 9 | 4 | 25% |
| Scale (10K users) | 7 | 9 | 20% |
| Time to Market (3 months) | 9 | 5 | 20% |
| Deployment (manual) | 8 | 4 | 15% |
| Cost | 8 | 5 | 10% |
| Risk (reversibility) | 6 | 7 | 10% |
| **Weighted Score** | **8.0** | **5.9** | |

**Conclusion**: Monolith wins based on our context

---

**Step 4: Make the Decision**

'Based on our criteria:
- Monolith scores higher (8.0 vs 5.9)
- **Key factors**: Small team, tight deadline, limited distributed systems experience
- **Decision**: Start with modular monolith
- **Future**: Re-evaluate in 12 months when we have scale needs'

---

**Step 5: Disagree and Commit**

To Engineer A (who wanted microservices):

'I know you're passionate about microservices and I value your input. Based on our current context (team size, timeline, expertise), monolith is the right choice now.

I need you to fully commit to this approach. Can you do that?

**Also**: Let's document your microservices proposal in an ADR. When we hit scale issues, we'll have a head start on the migration.'

---

**Step 6: Document the Decision**

Create Architecture Decision Record (ADR):

```markdown
# ADR-001: Start with Modular Monolith

## Status: Accepted

## Context
- Team: 6 engineers (moderate distributed systems experience)
- Timeline: 3 months to launch
- Scale: 1K current, 10K target (12 months)

## Decision
Start with modular monolith, not microservices

## Rationale
- Faster time to market (critical for business)
- Team size and expertise better suited for monolith
- Lower operational complexity
- Can migrate to microservices later if needed

## Consequences
- Might hit scaling challenges at 10K+ users
- Will need to ensure modules are well-bounded (prepare for future extraction)
- Will re-evaluate in 12 months
```

---

**Real Example**:

At my last company, same situation.

Two senior engineers: heated arguments in Slack, pulling team into camps.

I used this approach:
- 1:1s to understand perspectives
- Defined objective criteria (team, timeline, scale)
- Facilitated decision matrix
- Chose modular monolith
- Documented in ADR

**Result**:
- Team aligned within 1 week
- Launched on time
- After 18 months, extracted 2 services (as predicted)
- Both engineers respected the process and outcome

**Key**: Remove ego, focus on data and context."

---

### Scenario 2: Junior Developer Making Too Many Mistakes

**Question**: "A junior developer on your team has introduced several bugs in production over the past month. The team is frustrated. How do you handle this?"

**Good Answer**:

"This requires both **support** (help them improve) and **accountability** (ensure quality).

**Step 1: Investigate**

Before talking to the junior dev, I'd analyze the situation:

```markdown
## Bug Analysis

### Bugs Introduced (Last Month)
1. **Bug #234**: Null reference exception
   - **Root Cause**: Missing null check
   - **Impact**: High (500 errors/hour)
   - **Code Review**: Approved by Senior Dev A

2. **Bug #567**: Off-by-one error in loop
   - **Root Cause**: Logic error
   - **Impact**: Medium (data inconsistency)
   - **Code Review**: Approved by Senior Dev B

3. **Bug #890**: SQL injection vulnerability
   - **Root Cause**: Unsanitized input
   - **Impact**: Critical (security risk)
   - **Code Review**: Self-merged (!)

### Pattern Analysis
- **Common themes**: Basic programming errors (null checks, edge cases)
- **Code review**: 2/3 bugs passed code review → reviewer problem too
- **Tests**: No unit tests written → testing gap
- **Knowledge**: Lacks understanding of security, edge cases
```

**Observation**: This isn't just the junior's problem. It's a **system problem**.

---

**Step 2: 1:1 with Junior Developer**

Use **supportive coaching approach**, not blame:

**Me**: 'Hey [Name], I wanted to chat about the recent bugs. I'm not here to blame you, but to understand what's happening and how I can help. Walk me through what's been challenging.'

**Junior**: 'I feel overwhelmed. I'm not sure what good code looks like. Reviews happen fast, I assume if it's approved it's good. I don't know what to test.'

**Me**: 'That's really helpful feedback. Let me share what I'm seeing:'

```markdown
## What I Observed

**Bugs**: 3 in past month (null check, off-by-one, SQL injection)

**Patterns**:
- Missing edge case handling
- No unit tests
- Security vulnerabilities

**Good News**: These are all learnable skills!

**My Role**: I haven't set you up for success. Let me fix that.
```

---

**Step 3: Create Support Plan**

```markdown
## 30-Day Improvement Plan

### Week 1-2: Fundamentals
**Goal**: Build strong foundation

**Actions**:
- [ ] Pair programming (2 hours/day) with Senior Dev
- [ ] Review: Error handling patterns
- [ ] Review: Input validation and security
- [ ] Review: Edge case thinking

**Homework**:
- [ ] Complete 'Secure Coding' course (internal)
- [ ] Read: 'Clean Code' chapters 2-4

---

### Week 3-4: Apply & Reinforce
**Goal**: Practice with oversight

**Actions**:
- [ ] Take smaller, well-defined tasks
- [ ] Write unit tests BEFORE code (TDD)
- [ ] Mandatory code review by Senior Dev (detailed feedback)
- [ ] Pre-commit checklist (null checks, edge cases, tests)

**Checklist** (before submitting PR):
```checklist
✅ All edge cases handled (null, empty, boundary)
✅ Input validated and sanitized
✅ Unit tests written (min 80% coverage)
✅ Error handling added
✅ Logging added for debugging
✅ Self-code review completed
```

---

### Metrics to Track
- Bug count (target: 0 in next 30 days)
- Code review iterations (target: decrease from 5 to 2)
- Test coverage (target: 80%+)
- Code review approval rating (quality)

### Check-ins
- Weekly 1:1s (discuss progress, challenges)
- 30-day review (evaluate improvement)
```

---

**Step 4: Fix the System**

This isn't just about the junior. The **system** failed:

**To the Team**:

```markdown
## Process Improvements

**Problem**: Bugs are passing code review

**Changes**:

1. **Code Review Standards** (updated)
   - Reviewers must check for:
     ✅ Edge cases (null, empty, boundary)
     ✅ Security (input validation)
     ✅ Tests (coverage, edge cases)
     ✅ Error handling
   - **Action**: No approval without explicit checklist confirmation

2. **No Self-Merging**
   - All PRs require at least 1 approval
   - **Action**: Added GitHub branch protection

3. **Automated Quality Gates**
   - CI/CD must pass:
     ✅ 80% code coverage
     ✅ 0 security vulnerabilities (SonarQube)
     ✅ 0 critical code smells
   - **Action**: Configured in pipeline

4. **Mentorship Program**
   - Junior devs paired with Senior dev
   - 2 hours/week pairing sessions
   - **Action**: Added to sprint planning

**Accountability**: These bugs are on ME as Tech Lead. I didn't set us up for success. Let's fix the system.
```

---

**Step 5: Monitor Progress**

**Week 1**: Junior dev pairs with senior, learns patterns

**Week 2**: Submits first PR with tests, gets detailed feedback, iterates

**Week 3**: PR quality improving, only 1 round of feedback

**Week 4**: Submits clean PR, approved with minor comments

**30-Day Review**:
- 0 bugs in production
- Test coverage: 85%
- Code review iterations: 5 → 2
- Confidence: Low → Medium

**Me**: 'Huge improvement! You've shown you can learn and grow. Let's keep this momentum.'

---

**Real Example**:

At my last company, junior dev introduced 4 critical bugs in 2 weeks. Team wanted them fired.

I investigated:
- **Root cause**: No mentorship, poor code review, no testing culture
- **System problem**: Not individual problem

Created support plan:
- Paired with senior dev
- Implemented quality gates
- Weekly coaching

**Result**:
- After 2 months: 0 bugs, became reliable contributor
- After 6 months: Promoted, mentoring other juniors
- Team learned: Support > blame

**Key**: Fix the system, not just the person."

---

## 📊 Project & Delivery Challenges

### Scenario 3: Project is Behind Schedule

**Question**: "Your team committed to delivering a feature in 2 weeks. With 3 days left, you realize you're only 50% done. The PM is counting on this for a demo to a major client. What do you do?"

**Good Answer**:

"This is a critical situation requiring immediate action and transparency.

**Step 1: Assess the Situation (30 minutes)**

Gather facts:

```markdown
## Current State
- **Committed**: Feature X in 2 weeks (10 business days)
- **Progress**: 50% complete (Day 7)
- **Remaining**: 3 days
- **Realistic Completion**: 6 more days (total: 13 days vs. 10 committed)

## Why We're Behind
1. **Underestimated complexity**: Didn't account for third-party API integration (2 days)
2. **Unexpected bug**: Critical production bug consumed 1.5 days
3. **Scope creep**: PM added 2 'small' features mid-sprint (1 day)

## Business Context
- **Demo**: Major client, potential $500K deal
- **Stakeholder**: VP Sales, PM, CEO attending
- **Impact**: Missing this could lose the deal
```

---

**Step 2: Define Options (30 minutes)**

I'd present PM with **clear options**:

```markdown
## Option A: Deliver Core Feature Only (Recommended)
**Timeframe**: 3 days (on time)

**Scope**:
✅ Core workflow (happy path)
❌ Error handling (basic only)
❌ Edge cases (handle post-demo)
❌ Polish (UI improvements)

**Quality**: MVP quality, but functional for demo

**Risk**: Low (achievable)

**Demo Impact**: Can demonstrate core value proposition

**Post-Demo Work**: 3 more days to productionize (error handling, edge cases, polish)

---

## Option B: Delay Demo by 3 Days
**Timeframe**: 6 days (3 days late)

**Scope**: 100% of original commitment

**Quality**: Production-ready

**Risk**: Medium (lose demo slot)

**Demo Impact**: Can demo later, but may lose client engagement window

---

## Option C: Add Resources (Not Recommended)
**Timeframe**: Still 5 days (2 days late)

**Scope**: 100% of original commitment

**Quality**: Rushed, more bugs

**Risk**: High (throwing people at late project makes it later - Brooks' Law)

**Demo Impact**: Miss demo anyway

---

## Recommendation: Option A
- Deliver core feature for demo
- Demo shows value
- Complete productionization post-demo
- Total time: 6 days (same as Option B), but demo happens on time
```

---

**Step 3: Immediate Communication to PM (within 1 hour)**

**Me**: 'Hey [PM], I need to talk about Feature X. We have a situation.'

**PM**: 'What's going on?'

**Me**: 'We're behind schedule. We committed to 2 weeks, we're 50% done with 3 days left. I take responsibility for the miss – we underestimated complexity.

Here's where we are and what I propose:

**Current State**:
- 50% complete
- Need 6 more days to complete as specced

**Options**:
1. **Deliver core feature for demo (3 days)**: You can demo the happy path. It'll show the value. We finish productionization (error handling, edge cases) after the demo in 3 more days.

2. **Delay demo by 3 days**: We deliver 100% complete, but miss the demo slot.

**My Recommendation**: Option 1
- Demo happens on time
- You can show the core value proposition
- We don't lose the client window
- We finish productionization while you're in contract negotiations

**What do you think?'

---

**Step 4: Execute the Plan**

Assuming PM agrees with Option A:

**To the Team** (immediately):

```markdown
## Emergency Plan: Demo-Driven Development

**Goal**: Deliver demo-able core feature in 3 days

**Scope** (In):
✅ User login
✅ Core workflow (happy path)
✅ Basic UI (functional, not polished)
✅ Demo data seeded

**Scope** (Out - Post-Demo):
❌ Error handling (beyond basics)
❌ Edge cases
❌ UI polish
❌ Performance optimization

**Schedule**:
- **Day 1**: Complete login + workflow step 1-2
- **Day 2**: Complete workflow step 3-4
- **Day 3**: Polish for demo, test demo scenario

**Daily Stand-up** (15 min, 9am):
- What's blocking you?
- Are we on track?

**Daily Sync with PM** (5 min, 5pm):
- Show progress
- Identify any concerns

**Post-Demo** (3 days):
- Add error handling
- Handle edge cases
- UI polish
- Performance testing
```

**To the Team**:
'This is my fault for underestimating. Let's rally and get this demo-ready. I'm removing all other work for the next 3 days. Focus 100% on this. After the demo, we'll take a retrospective and learn from this.'

---

**Step 5: Communicate to Stakeholders**

**Email to VP Sales, CEO** (cc: PM):

```
Subject: Feature X Update - Demo Ready on Schedule

Hi [VP Sales],

Quick update on Feature X for the [Client] demo:

**Status**: On track for demo on [Date]

**Scope for Demo**:
- Core workflow (end-to-end happy path)
- Demonstrates value proposition
- Functional for demo scenarios

**Post-Demo Work** (transparent):
- We'll complete productionization (error handling, edge cases, polish) in 3 days post-demo
- Ready for beta customer use by [Date + 3 days]

**Why**:
- Feature was more complex than initially estimated
- We prioritized getting the demo-ready version done on time rather than delaying

**Confidence**: High - we're on track for a successful demo.

Let me know if you have questions.

[Your Name]
```

---

**Step 6: Execute & Deliver**

**Day 1**: Team focused, login + workflow 1-2 done

**Day 2**: Workflow 3-4 done, starting to come together

**Day 3**: Polish, test demo scenario, ready!

**Demo Day**: Success! Client impressed, deal moves forward.

**Post-Demo**: Team completes productionization in 3 days.

---

**Step 7: Retrospective**

**After completion**, hold retrospective:

```markdown
## Retrospective: Feature X Delivery

**What Went Well**:
- Team rallied and delivered demo on time
- Transparent communication with stakeholders
- Clear prioritization (demo-ready vs. production-ready)

**What Didn't Go Well**:
- Underestimated complexity (third-party API integration)
- Didn't account for production bug time
- Scope creep mid-sprint

**Action Items**:
1. **Estimation**: Add 20-30% buffer for unknowns
2. **Scope Changes**: Mid-sprint changes require re-estimation
3. **Risk Assessment**: Identify integration risks upfront
4. **Retrospective**: Review estimates vs actuals quarterly

**Lessons Learned**:
- Transparency is critical
- Always have Options (A, B, C)
- Focus on business outcome, not just feature completeness
```

---

**Real Example**:

At my last company, exactly this scenario.

**Mistake**: I hid the delay, hoped we'd catch up. We didn't.

**Result**: Missed demo, lost $300K deal, PM and CEO blindsided.

**Lesson**: **Transparency early >> Optimism**

**Next time** (different project):
- Same situation, realized we're behind
- Immediately communicated to PM (Day 5 of 10)
- Proposed options
- Delivered demo-ready version on time
- Completed productionization post-demo

**Result**: Successful demo, won deal, stakeholders appreciated transparency.

**Key**: Bad news early, not late. Always present options, not just problems."

---

## 💻 Technical Decisions Under Pressure

### Scenario 4: Production is Down During Peak Hours

**Question**: "It's Black Friday, your e-commerce site is down, and you're losing $10K/minute. Your team is panicking. How do you handle this?"

**Good Answer**:

"This is a crisis requiring calm leadership and systematic problem-solving.

**Step 1: Take Command (Immediately)**

First 60 seconds:

**Me** (in team Slack):

```
🚨 INCIDENT: Site down. I'm taking command.

ROLES:
- @SeniorDev1: Investigate database
- @SeniorDev2: Check application logs
- @Junior: Monitor user reports
- @DevOps: Check infrastructure (servers, network)

COMMUNICATION:
- Updates every 5 minutes (in this thread)
- @PM: You handle external comms (customers, executives)
- Everyone: No speculation. Facts only.

I'll coordinate. Let's go. 🚀
```

**Key**: Clear roles, clear communication, calm tone.

---

**Step 2: Triage (First 5 minutes)**

**DevOps**: 'Database CPU at 100%, queries timing out'

**SeniorDev1**: 'Seeing massive spike in queries to Orders table'

**SeniorDev2**: 'Application logs show 500 errors, database timeouts'

**Me**: 'Got it. Database is bottleneck. @SeniorDev1: What's causing the query spike?'

**SeniorDev1**: '1 query taking 30 seconds, running 1000x/sec. New feature deployed this morning.'

**Me**: 'Okay. **Root cause**: New feature causing expensive query. **Plan**: Rollback deployment. @DevOps: How fast can we rollback?'

**DevOps**: '5 minutes'

**Me**: '@DevOps: Start rollback now. @SeniorDev1: Prepare hotfix (add index or optimize query). We'll roll forward once ready. Everyone else: Stand by.'

---

**Step 3: Execute Fix (Next 5-10 minutes)**

**5 min later**:

**DevOps**: 'Rollback complete. Database CPU dropping...50%...30%...10%. Site back up!'

**Me**: 'Excellent! @PM: Site is back up. ETA for permanent fix: 30 minutes. @SeniorDev1: What's the hotfix?'

**SeniorDev1**: 'Need to add index to Orders.CustomerId. Will reduce query from 30s to 50ms.'

**Me**: 'Good. Create PR, I'll review. Let's test in staging first. No more surprises today.'

**30 min later**:

**Me**: 'Hotfix ready, tested in staging. Deploying now...'

**DevOps**: 'Deployed. Database queries down to 50ms. No errors.'

**Me**: 'Great work team! Site stable. Now let's do a post-mortem.'

---

**Step 4: Communication**

**To PM** (for external comms):

```
**5-min Update** (while fixing):
'We've identified the issue (database overload from new feature). Rolling back deployment now. ETA: 5 minutes.'

**15-min Update** (after rollback):
'Site is back up. We're deploying a permanent fix. No data loss.'

**60-min Update** (after permanent fix):
'Issue resolved. Root cause: Missing database index in new feature. Permanent fix deployed and tested. Monitoring closely.'
```

**To Leadership** (CEO, CTO):

```
Subject: Incident Report - Site Outage (Resolved)

**Summary**:
- **Duration**: 15 minutes
- **Impact**: $150K revenue loss (15 min × $10K/min)
- **Root Cause**: Missing database index in new feature

**Timeline**:
- 11:00: Site down, alerted
- 11:05: Root cause identified (database overload)
- 11:10: Rollback deployed, site back up
- 11:30: Permanent fix deployed (added index)
- 11:40: Fully resolved, monitoring

**Immediate Actions Taken**:
1. Rolled back deployment
2. Added missing database index
3. Tested and deployed hotfix

**Next Steps**:
- Post-mortem meeting (tomorrow)
- Improve deployment process (prevent future incidents)

**Kudos**: Team responded quickly and professionally.

[Your Name]
```

---

**Step 5: Post-Mortem (Next Day)**

```markdown
## Post-Mortem: Black Friday Outage

**Date**: November 24, 2024
**Duration**: 15 minutes
**Impact**: $150K revenue loss

### Timeline
- **10:55**: New feature deployed
- **11:00**: Site down (database overload)
- **11:05**: Root cause identified
- **11:10**: Rollback complete, site up
- **11:30**: Hotfix deployed (added index)

### Root Cause
Missing database index on `Orders.CustomerId` in new feature.

Query took 30s (should be 50ms), ran 1000x/sec during peak traffic.

### Why Did This Happen?
1. **Code Review**: Reviewer didn't catch missing index
2. **Staging Test**: Load testing not done (staging has small dataset)
3. **Deployment**: Deployed during peak hours (bad timing)

### What Went Well
- Quick triage (5 min to root cause)
- Clear command structure
- Calm, focused team
- Fast rollback (5 min)
- Good external communication

### What Didn't Go Well
- Deployed during peak hours (should have waited)
- Missing load testing
- No database query review in PR

### Action Items
1. **Immediate**:
   - [ ] Add `Orders.CustomerId` index (DONE)
   - [ ] Audit all queries in new feature (DONE)

2. **Short Term** (1 week):
   - [ ] Database query review checklist for PRs
   - [ ] Explain plan analysis for slow queries (> 100ms)
   - [ ] Load testing required for features touching high-traffic tables

3. **Long Term** (1 month):
   - [ ] Automated query performance testing in CI/CD
   - [ ] Deployment blackout during peak hours (10am-2pm, 7pm-10pm)
   - [ ] Database monitoring dashboard (query performance)

4. **Communication**:
   - [ ] Share post-mortem with leadership (DONE)
   - [ ] Team training: Database performance best practices

### Lessons Learned
- **Prevention > Reaction**: Load testing would have caught this
- **Timing Matters**: Don't deploy during peak hours
- **Communication**: Clear roles reduced chaos
```

---

**Real Example**:

At my previous company, similar situation.

**Difference**: I panicked, team thrashed, took 2 hours to fix.

**Lesson**: **Calm leadership is critical in a crisis.**

**Next incident** (different company):
- Stayed calm
- Assigned roles
- Clear communication
- Fixed in 10 minutes

**Team feedback**: 'Your calm demeanor kept us focused.'

**Key**: In a crisis, team looks to you for calm and clarity."

---

## 📚 Quick Reference

### Leadership Scenario Framework

**1. Conflicts**:
- Listen to all perspectives (1:1s)
- Define objective criteria
- Facilitate data-driven decision
- Document decision (ADR)
- Get commitment from all parties

**2. Performance Issues**:
- Investigate root cause (system vs individual)
- Create support plan with clear goals
- Fix the system, not just the person
- Monitor progress weekly
- Celebrate improvement

**3. Project Delays**:
- Assess situation quickly (within 1 hour)
- Define clear options (A, B, C)
- Communicate transparently (bad news early)
- Present options, not just problems
- Execute with daily check-ins

**4. Production Incidents**:
- Take command immediately
- Assign clear roles
- Communicate updates every 5 minutes
- Fix, then learn (post-mortem)
- Implement preventive measures

### STAR Method Template

```markdown
**Situation**: [Context, what was happening]

**Task**: [Your role, what you needed to do]

**Action**: [Specific steps you took, with reasoning]

**Result**: [Outcome, metrics, what you learned]
```

---

## 🔗 Related Topics
- [Tech Lead Behavioral Questions](./tech-lead-behavioral.md)
- [Conflict Resolution](../leadership/conflict-resolution.md)
- [Team Mentoring](../leadership/team-mentoring.md)
- [Architecture Decisions](../technical-leadership/architecture-decisions.md)
- [Technical Debt Management](../technical-leadership/technical-debt.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
