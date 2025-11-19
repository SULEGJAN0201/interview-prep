# Team Mentoring & Coaching

## 📝 Overview

As a Tech Lead, mentoring is one of your most critical responsibilities. It's about growing your team's capabilities, accelerating career development, and building a culture of continuous learning.

**Key Principle**: Your success as a Tech Lead is measured by your team's growth, not just your individual contributions.

---

## 🎯 Core Mentoring Competencies

### 1. One-on-One Meetings (1:1s)

**Purpose**: Build trust, provide guidance, understand individual goals

#### Structure of Effective 1:1s

```
Weekly/Biweekly 30-60 minute sessions

Template:
1. Personal Check-in (5 min)
   - How are you doing? Any blockers?

2. Career Development (15 min)
   - Progress on goals
   - Skills they want to develop
   - Challenges they're facing

3. Project Updates (10 min)
   - Current work status
   - Technical challenges

4. Feedback (Both Directions) (10 min)
   - Your feedback to them
   - Their feedback to you/team/process

5. Action Items (5 min)
   - Specific next steps
   - Resources or support needed
```

#### 1:1 Best Practices

✅ **DO:**
- Let them set the agenda
- Listen more than you talk (70/30 rule)
- Take notes on career goals and follow up
- Be consistent - don't cancel unless absolutely necessary
- Create psychological safety - make it a judgment-free zone
- Focus on the person, not just the work

❌ **DON'T:**
- Turn it into a status update meeting
- Do all the talking
- Skip them when "too busy"
- Only focus on current projects
- Use it only for criticism
- Check your phone/laptop during the meeting

### 2. Career Development Planning

#### Creating Individual Development Plans (IDPs)

```markdown
## Individual Development Plan Template

**Engineer**: [Name]
**Current Level**: Senior Engineer
**Target Level**: Staff Engineer
**Timeline**: 12 months

### Technical Skills

| Skill | Current | Target | Action Plan | Due Date |
|-------|---------|--------|-------------|----------|
| System Design | 3/5 | 5/5 | Lead design of new service, review system design docs | Q2 2025 |
| Kubernetes | 2/5 | 4/5 | Complete K8s certification, migrate 2 services | Q3 2025 |

### Leadership Skills

| Skill | Current | Target | Action Plan | Due Date |
|-------|---------|--------|-------------|----------|
| Mentoring | 2/5 | 4/5 | Mentor 1 junior engineer, lead tech talks | Q2 2025 |
| Tech Talks | 1/5 | 3/5 | Present at team meetings, external conference | Q4 2025 |

### Quarterly Goals
- Q1: Complete system design for new authentication service
- Q2: Lead implementation with 2 engineers
- Q3: Present learnings to broader engineering team
- Q4: Write blog post about the experience
```

#### Growth Opportunities to Provide

1. **Technical Challenges**
   - Assign stretch projects slightly above current level
   - Pair on complex architecture decisions
   - Let them lead design reviews

2. **Visibility**
   - Have them present at team meetings
   - Introduce them to stakeholders
   - Let them represent the team in cross-functional meetings

3. **Ownership**
   - Give ownership of features/services
   - Let them make decisions (with guidance)
   - Support them when they make mistakes

### 3. Skill Gap Analysis

```csharp
// Example: Assessing Technical Skills

public class SkillAssessment
{
    public enum SkillLevel
    {
        Beginner = 1,      // Needs significant guidance
        Intermediate = 2,  // Can work independently on standard tasks
        Advanced = 3,      // Can handle complex tasks
        Expert = 4,        // Can teach others, makes architectural decisions
        Master = 5         // Industry expert, thought leader
    }

    public class EngineerSkillMatrix
    {
        public string EngineerName { get; set; }
        public Dictionary<string, SkillLevel> TechnicalSkills { get; set; }
        public Dictionary<string, SkillLevel> SoftSkills { get; set; }

        public List<SkillGap> IdentifyGaps(string targetRole)
        {
            var requiredSkills = GetRequiredSkills(targetRole);
            var gaps = new List<SkillGap>();

            foreach (var required in requiredSkills)
            {
                var current = TechnicalSkills.GetValueOrDefault(required.Key, 0);
                if (current < required.Value)
                {
                    gaps.Add(new SkillGap
                    {
                        Skill = required.Key,
                        CurrentLevel = current,
                        RequiredLevel = required.Value,
                        Priority = CalculatePriority(required.Key, required.Value - current)
                    });
                }
            }

            return gaps.OrderByDescending(g => g.Priority).ToList();
        }
    }
}
```

---

## 💡 Mentoring Scenarios & Examples

### Scenario 1: Junior Developer Struggling with Code Quality

**Situation**: Junior developer's code reviews frequently require many changes.

**❌ Poor Approach**:
"Your code quality needs to improve. Please rewrite this following best practices."

**✅ Good Approach**:
1. **Pair Programming Session**
   - "Let's pair on this feature together. I'll help you think through the design."
   - Work through it together, explaining your thought process

2. **Provide Resources**
   - "I noticed we're working on exception handling. Here's a great article on the topic."
   - Share the team's coding standards document

3. **Set Clear Expectations**
   - "For this level, we expect: clear variable names, single responsibility, unit tests"
   - Create a checklist for them to use before submitting PRs

4. **Positive Reinforcement**
   - "I really like how you handled error cases here. Let's apply that pattern to this section too."

**Follow-up**:
- Check in after each PR
- Gradually reduce pairing as they improve
- Celebrate progress

### Scenario 2: Mid-Level Developer Ready for More

**Situation**: A solid engineer who's plateaued and needs new challenges.

**✅ Approach**:

```markdown
## Growth Conversation

"I've noticed you're consistently delivering high-quality work on your assigned tasks.
I think you're ready for the next level. What interests you?

Options:
1. **Technical Depth**: Lead the design of our new caching layer
2. **Breadth**: Work with Product team on roadmap planning
3. **Leadership**: Mentor our new junior engineer
4. **Innovation**: Spend 20% time on a technical improvement project

Let's pick 1-2 and create a plan together."
```

**Implementation**:
- Start small with increased responsibility
- Provide support but let them drive
- Schedule check-ins to unblock
- Give public credit for their work

### Scenario 3: Senior Engineer with Performance Issues

**Situation**: A senior engineer's output has declined, missing deadlines.

**❌ Poor Approach**:
"You're not meeting expectations. Your performance needs to improve or we'll have to take action."

**✅ Good Approach**:

1. **Private, Caring Conversation**
   ```
   "I've noticed some changes recently and I'm concerned.
   Is everything okay? How can I help?"
   ```

2. **Listen First**
   - Might be personal issues, burnout, unclear expectations
   - Show empathy and support

3. **Collaborate on Solution**
   ```
   "Let's work together on this. What would help you most?
   - Reduced scope?
   - Clearer priorities?
   - Different project?
   - Time off?
   ```

4. **Create Improvement Plan** (if needed)
   - Specific, measurable goals
   - Regular check-ins
   - Document progress

---

## 🎤 Common Interview Questions

### Q1: "Tell me about a time you successfully mentored someone."

**STAR Framework**:

**Situation**:
"I was mentoring a junior developer, Sarah, who joined our team 6 months ago. She was struggling with understanding our microservices architecture and felt overwhelmed."

**Task**:
"My goal was to help her become productive and confident in contributing to our services within 3 months."

**Action**:
"I took several approaches:
1. **Structured Learning**: Created a 6-week learning plan starting with our simplest service
2. **Pairing Sessions**: Paired with her 2x per week to work through real features
3. **Safe Environment**: Gave her a 'safe to fail' project - a new internal tool
4. **Gradual Complexity**: Started with small bug fixes, then features, then design
5. **Regular Feedback**: Weekly 1:1s to discuss progress and challenges"

**Result**:
"Within 3 months, Sarah:
- Independently delivered 3 features to production
- Led the design of a new reporting service
- Presented her work at team demo
- Became confident enough to mentor our next junior hire

She's now one of our strongest mid-level engineers."

**Learning**:
"I learned that consistent, patient mentoring with gradual complexity increase is more effective than throwing someone in the deep end."

---

### Q2: "How do you handle underperformance on your team?"

**Good Answer**:

"I address underperformance with empathy and structure:

**Step 1: Understand**
- Private conversation to understand root cause
- Personal issues? Unclear expectations? Wrong role? Technical gaps?

**Step 2: Collaborate**
- Work together on a plan
- Set clear, measurable goals
- Identify support needed

**Step 3: Document & Execute**
- Regular check-ins (weekly)
- Document progress
- Adjust plan as needed

**Example**:
I had a senior engineer whose code quality declined. In our 1:1, I learned he was bored with maintenance work.

Solution:
- Moved him to a greenfield project
- Performance immediately improved
- He became re-engaged

**If no improvement**:
- Escalate to manager
- Consider role change
- As last resort, performance improvement plan

The key is addressing it early, with compassion, but also with clear expectations."

---

### Q3: "How do you develop technical skills in your team?"

**Good Answer**:

"I use multiple approaches:

**1. Structured Learning**
- Lunch & Learn sessions (weekly)
- Book club for technical books
- Conference attendance budget
- Online course subscriptions

**2. On-the-Job Learning**
- Stretch assignments
- Rotate ownership of different services
- Lead design reviews
- Present technical topics

**3. Knowledge Sharing**
- Tech talks (bi-weekly)
- Architecture decision documentation
- Code review as teaching tool
- Pair programming on complex problems

**4. Individual Development Plans**
- Quarterly skill assessments
- Personalized growth plans
- Track progress in 1:1s

**Example**:
Our team wanted to learn Kubernetes. I:
- Sponsored K8s certification for 3 engineers
- Migrated one service as learning project
- Had them present learnings to team
- Now they mentor others

**Metrics I Track**:
- Skills developed per engineer per quarter
- Certifications earned
- Internal presentations given
- Promotion rate"

---

### Q4: "Describe your approach to giving feedback."

**Good Answer**:

"I use the SBI (Situation-Behavior-Impact) model:

**Positive Feedback** (Public):
- **Situation**: 'In yesterday's design review...'
- **Behavior**: '...you asked great questions about edge cases'
- **Impact**: '...which helped us avoid a major production issue'

**Constructive Feedback** (Private):
- **Situation**: 'In the code review for PR #123...'
- **Behavior**: '...the error handling was missing try-catch blocks'
- **Impact**: '...which could crash the service if the DB is down'
- **Forward**: 'Let's pair on implementing proper error handling patterns'

**Key Principles**:
1. **Timely**: Within 24-48 hours
2. **Specific**: Concrete examples, not vague
3. **Balanced**: Recognize strengths and areas for growth
4. **Actionable**: Clear next steps
5. **Two-way**: Ask for their perspective
6. **Follow-up**: Check progress

**Example**:
After a junior engineer's first major feature:

Good parts:
- Excellent test coverage
- Clean code structure

Areas for growth:
- Performance optimization opportunity
- Could have involved QA earlier

Next steps:
- Let's review performance profiling together
- I'll introduce you to QA lead for next feature"

---

## 📚 Mentoring Resources & Tools

### Books
- "The Manager's Path" by Camille Fournier
- "Radical Candor" by Kim Scott
- "Thanks for the Feedback" by Douglas Stone
- "Debugging Teams" by Brian Fitzpatrick

### Frameworks
- **GROW Model**: Goal, Reality, Options, Way Forward
- **SBI Feedback**: Situation, Behavior, Impact
- **Skill/Will Matrix**: High Skill/Low Will vs Low Skill/High Will

### Templates
- 1:1 Meeting Template
- Individual Development Plan
- Feedback Document Template
- Career Ladder / Competency Matrix

---

## 🎯 Quick Reference

### Mentoring Do's and Don'ts

| Do ✅ | Don't ❌ |
|------|---------|
| Listen more than talk | Dominate conversations |
| Ask open-ended questions | Give all the answers |
| Provide context and reasoning | Just say "do it this way" |
| Let them make mistakes (safely) | Micromanage |
| Celebrate small wins | Only focus on problems |
| Invest time in their growth | Only care about delivery |
| Be consistent and reliable | Cancel 1:1s frequently |
| Give specific, actionable feedback | Give vague criticism |

---

## 🔗 Related Topics
- [Conflict Resolution](./conflict-resolution.md)
- [Performance Management](./performance-management.md)
- [Code Review Practices](../technical-leadership/code-review-practices.md)
- [Team Building](./team-motivation.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐☆☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
