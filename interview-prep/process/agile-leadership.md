# Agile Leadership for Tech Leads

## 📝 Overview

As a Tech Lead, you're not just a participant in Agile ceremonies - you're a **facilitator, technical advisor, and team advocate**. You bridge the gap between product management, the team, and technical execution.

**Key Responsibility**: Ensure the team delivers value iteratively while maintaining technical excellence.

---

## 🎯 Tech Lead's Role in Agile Ceremonies

### Sprint Planning

**Your Responsibilities**:

1. **Technical Feasibility Assessment**
```markdown
When Product brings stories:

❌ Don't: "That sounds fine, the team can figure it out."
✅ Do: "Let me break down the technical complexity:
- Story A: 5 points (straightforward)
- Story B: 13 points (needs architecture discussion first)
- Story C: Blocked by missing API access

We should prioritize A, discuss B in tech design meeting, and
resolve C's blocker before committing."
```

2. **Breakdown of Technical Tasks**
- Help team decompose user stories into technical tasks
- Identify dependencies and risks upfront
- Ensure stories are sized appropriately

3. **Capacity Planning**
```csharp
// Example: Track team capacity realistically

public class SprintCapacity
{
    public int TotalDays { get; set; } // 10 days in 2-week sprint
    public int TeamSize { get; set; } // 5 engineers

    public int TheoricalCapacity => TotalDays * TeamSize; // 50 person-days

    public int ActualCapacity
    {
        get
        {
            var capacity = TheoricalCapacity;
            capacity -= VacationDays; // John out 2 days
            capacity -= MeetingOverhead; // ~20% for meetings, code review
            capacity -= OnCallRotation; // ~10% for support
            capacity -= TechnicalDebt; // Reserve 20% for debt/bugs

            return capacity; // More like 25-30 person-days
        }
    }

    // Don't over-commit!
    public bool CanCommitToPoints(int storyPoints)
    {
        var historicalVelocity = 28; // What we've actually done
        return storyPoints <= historicalVelocity;
    }
}
```

4. **Push Back When Needed**
```markdown
Product Manager: "Can we add these 5 more stories?"

❌ Bad Response: "Sure, we'll try."
(Over-commit → burnout → missed sprint → trust erosion)

✅ Good Response:
"Our velocity is 30 points. We're at 28 now. We can take 2 more points.

For the others, let's prioritize:
- Which is most critical for the business?
- Can any wait for next sprint?
- Should we drop a lower-priority story we already committed to?

I want to make sure we deliver what we commit to rather than
over-committing and under-delivering."
```

---

### Daily Standup

**Your Role**: Facilitator, unblocked, connector

**Anti-patterns to Avoid**:
❌ Status report to you
❌ Problem-solving in standup (take offline)
❌ You doing all the talking

**Best Practices**:
✅ Keep it 15 minutes max
✅ Identify blockers quickly
✅ Facilitate connections: "Mike, can you help Sarah with that API issue after standup?"
✅ Note action items

**Example Facilitation**:

```markdown
## Good Standup Flow

Engineer: "I'm blocked on the database migration script."

❌ Bad Tech Lead Response:
"Let me look at that. I think you need to..."
[Proceeds to solve it in standup - 10 minutes wasted]

✅ Good Tech Lead Response:
"Got it - blocker on DB migration. Let's sync after standup.
Anyone else familiar with migrations who can join?
[Moves on to next person]

[After standup: 15-minute focused session with relevant people]
```

---

### Sprint Review / Demo

**Your Responsibilities**:

1. **Prepare the Team**
```markdown
Day Before Demo Checklist:
- [ ] Ensure demos are in a working demo environment
- [ ] Practice run-through with team
- [ ] Prepare rollback plan if demo breaks
- [ ] Coach junior engineers on presenting their work
- [ ] Prepare technical Q&A responses
```

2. **Showcase Technical Excellence**
```markdown
Don't just demo features - explain the HOW:

"Here's the new payment flow [demo].

From a technical perspective, we:
- Reduced latency from 800ms to 200ms using Redis caching
- Improved reliability with circuit breakers
- Added comprehensive monitoring

This sets us up well for the Black Friday traffic we're expecting."

(Educates stakeholders, shows technical value)
```

3. **Manage Stakeholder Expectations**
```markdown
Stakeholder: "Why doesn't it have X feature? I thought that was included."

❌ Bad: "That wasn't in the story."
(Defensive, not helpful)

✅ Good: "Good question. The original story focused on Y.
Adding X would require changes to Z system.

Can we discuss priorities? Options:
1. Add X to backlog for next sprint
2. I can give you an estimate for X
3. We could do a simplified version of X in this sprint

What's most valuable for the business?"
```

---

### Sprint Retrospective

**Your Role**: Facilitator of continuous improvement

**Retrospective Format** (Rotate to keep fresh):

#### Format 1: Start, Stop, Continue

```markdown
Start:
- Pair programming on complex features
- Architecture review for 8+ point stories

Stop:
- Last-minute changes before deployment
- Skipping code review for "urgent" fixes

Continue:
- Tech talks every Friday
- 20% time for technical debt
```

#### Format 2: 4 L's

```markdown
Liked:
- New deployment pipeline saved us hours
- Team collaboration was great

Learned:
- Need to involve QA earlier
- Underestimated database migration complexity

Lacked:
- Clear acceptance criteria on some stories
- Time for documentation

Longed For:
- Better testing environment
- More pairing time
```

**Your Actions Post-Retro**:

```markdown
## Retrospective Action Items

Problem: "Code reviews taking 2+ days"

✅ What I'll Do:
1. Set team SLA: Reviews within 4 hours
2. Rotate "reviewer of the day" responsibility
3. Add Slack reminder bot
4. Track metrics and review in 2 weeks

Problem: "Too many meetings"

✅ What I'll Do:
1. Audit all recurring meetings
2. Cancel meetings with unclear purpose
3. Move standups from 30min to 15min
4. Institute "No Meeting Fridays"

The key: Turn complaints into concrete action items with owners and timelines.
```

---

## 💡 Common Agile Scenarios

### Scenario 1: Product Wants to Change Mid-Sprint

**Situation**:
"Day 3 of sprint, Product Manager: 'Customer needs feature X urgently. Can we swap it in?'"

**❌ Poor Response**:
"Sure, we'll add it."
(Disrupts sprint, team loses focus, original commitments missed)

**✅ Good Response**:

```markdown
"Let me understand the urgency and impact.

Questions:
1. How critical is this? What's the business impact?
2. What's the deadline?
3. What can we drop to make room?

Options:
A) True emergency: We stop everything and swarm on it
   (Rare - only for production outages or revenue blockers)

B) Important but not urgent: Add to next sprint
   (Most common - can wait 1-2 weeks)

C) Important and urgent: Swap for equal-sized story
   - We drop Story X (lowest priority we committed to)
   - We add Feature X
   - Risk: Original Story X moves to next sprint

Which makes most sense for the business?"
```

**Follow-up**:
"If this happens often, let's discuss in retro how we can improve our planning to avoid mid-sprint changes."

---

### Scenario 2: Team Missing Sprint Commitments

**Situation**:
"3rd sprint in a row we've missed our commitment"

**Analysis Framework**:

```markdown
## Root Cause Analysis

1. Are we over-committing?
   - Compare estimated vs. actual velocity
   - Factor in meetings, support, etc.

2. Are estimates inaccurate?
   - Track estimation accuracy
   - Team may need calibration

3. Are there frequent blockers?
   - Dependencies on other teams?
   - Infrastructure issues?
   - Unclear requirements?

4. Is scope creeping mid-sprint?
   - Unplanned work?
   - "Just one more thing" syndrome?

5. Is team capacity changing?
   - Vacations, new hires, departures?
```

**Action Plan Example**:

```markdown
Problem: Over-committing by 30%

Solutions:
1. Reduce commitment to 70% of theoretical capacity
2. Reserve 20% for bugs/technical debt
3. Account for meeting overhead
4. Track actual velocity for 3 sprints, then adjust

Problem: Unclear requirements causing rework

Solutions:
1. Require acceptance criteria before story enters sprint
2. Three Amigos session (Dev, QA, Product) before sprint
3. No story over 8 points without tech design doc
```

---

## 🎤 Common Interview Questions

### Q1: "How do you handle competing priorities from multiple stakeholders?"

**Good Answer**:

"I use a structured prioritization framework:

**1. Clarify Business Impact**
Talk to each stakeholder:
- What's the business value?
- What's the cost of delay?
- Who's the customer impact?

**2. Assess Technical Dependencies**
- What's blocked by what?
- What's the technical complexity?

**3. Use a Framework** (e.g., RICE):
- **R**each: How many users?
- **I**mpact: How much improvement?
- **C**onfidence: How sure are we?
- **E**ffort: How much work?

Score = (Reach × Impact × Confidence) / Effort

**4. Facilitate Discussion**
Bring stakeholders together:
'Here's what I'm hearing. Given our capacity, we can do 2 of these 5.
Let's decide together based on business value.'

**5. Document and Communicate**
Whatever we decide, document why and communicate to all stakeholders.

**Example**:
Had Product, Support, and Sales all wanting features.

Scored each:
- Product feature: RICE score 45
- Support request: RICE score 30
- Sales feature: RICE score 60

Result: Sales feature first, then product.

Support understood why when shown the scoring.

Key: Transparent, data-driven prioritization with stakeholder input."

---

### Q2: "How do you balance technical debt with feature delivery?"

**Good Answer**:

"I use the '20% rule' with flexibility:

**1. Reserve 20% of Sprint Capacity**
Default allocation:
- 70% new features
- 20% technical debt / quality
- 10% bugs / support

**2. Track Debt Visibility**
Maintain a 'tech debt backlog' with:
- Description and impact
- Effort estimate
- Business risk if not addressed
- Proposed solution

**3. Prioritize High-Impact Debt**
Not all tech debt is equal:
- Security vulnerabilities: Immediate
- Performance issues affecting users: High priority
- Code cleanliness: Lower priority

**4. Tie Debt to Business Value**
When advocating for debt work, use business terms:

❌ 'We need to refactor this code, it's messy.'
✅ 'This module causes 40% of our production bugs. Refactoring it
   would reduce our bug rate and improve velocity by 15%.'

**5. Make It Visible**
- Include tech debt stories in sprint planning
- Show metrics: bug rate, deployment frequency, incident count
- Demonstrate improvement over time

**Example**:
Our API layer had grown unwieldy.

I made the case:
- Current: 2 days to add new endpoint
- After refactor: 2 hours
- 10 new endpoints planned
- ROI: Refactor saves 15 days over next quarter

Got approval for 2-sprint refactor.

Result: New endpoint velocity improved 80%, team happier, less bugs.

Key: Balance is about making technical debt visible and tying it
to business outcomes, not hiding it or ignoring features."

---

## 📚 Quick Reference

### Tech Lead's Agile Checklist

#### Sprint Planning
- [ ] Review stories for technical feasibility
- [ ] Identify dependencies and risks
- [ ] Help break down large stories
- [ ] Ensure team doesn't over-commit
- [ ] Set technical goals for the sprint

#### Daily Standup
- [ ] Keep it to 15 minutes
- [ ] Note blockers and follow up
- [ ] Connect people who can help each other
- [ ] Identify risks to sprint goal

#### Sprint Review
- [ ] Ensure demos are prepared
- [ ] Coach team on presenting
- [ ] Explain technical decisions made
- [ ] Gather feedback from stakeholders

#### Sprint Retrospective
- [ ] Facilitate safe environment
- [ ] Ensure everyone participates
- [ ] Convert feedback to action items
- [ ] Follow through on commitments
- [ ] Track improvements over time

---

## 🔗 Related Topics
- [Team Mentoring](../leadership/team-mentoring.md)
- [Architecture Decisions](../technical-leadership/architecture-decisions.md)
- [Conflict Resolution](../leadership/conflict-resolution.md)
- [Tech Lead Behavioral Questions](../interview-questions/tech-lead-behavioral.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐☆☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
