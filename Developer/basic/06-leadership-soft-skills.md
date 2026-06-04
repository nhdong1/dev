# Leadership & Soft Skills - Interview Questions

## Question 1: Bạn tiếp cận technical decision making như thế nào?

**Answer:**

**Framework cho Technical Decision Making:**

**1. Define the Problem**
```
- Vấn đề thực sự là gì?
- Ai bị ảnh hưởng?
- Scope và timeline là gì?
- Success criteria là gì?
```

**2. Gather Context**
```
- Current system constraints
- Business requirements
- Team capabilities
- Timeline và resources
- Existing technical debt
```

**3. Identify Options**
```
Option A: Build custom solution
Option B: Use existing library/framework
Option C: Buy/integrate third-party service
Option D: Do nothing (là một option hợp lệ)
```

**4. Evaluate Trade-offs**
```
| Criteria        | Option A | Option B | Option C |
|-----------------|----------|----------|----------|
| Development time| Long     | Medium   | Short    |
| Maintenance     | High     | Medium   | Low      |
| Flexibility     | High     | Medium   | Low      |
| Cost            | High     | Medium   | Medium   |
| Risk            | High     | Low      | Medium   |
```

**5. Document the Decision (ADR - Architecture Decision Record)**
```markdown
# ADR-001: Choose PostgreSQL over MongoDB

## Status
Accepted

## Context
We need to store user transaction data with ACID requirements.

## Decision
We will use PostgreSQL for transaction data.

## Consequences
- Team needs PostgreSQL expertise
- Strong consistency guaranteed
- May need sharding at scale
```

**6. Communicate & Get Buy-in**
```
- Present to stakeholders
- Address concerns
- Be open to feedback
- Document dissenting opinions
```

**7. Evaluate & Iterate**
```
- Set up monitoring
- Schedule review
- Be willing to revisit decisions
- Learn from outcomes
```

**Example Response:**
"Khi cần quyết định migrate từ monolith sang microservices, tôi bắt đầu bằng việc document các pain points hiện tại. Sau đó, tôi đã prototype một service nhỏ để đánh giá complexity. Cuối cùng, tôi present findings cho team với clear trade-offs và timeline estimates."

---

## Question 2: Làm thế nào để mentor junior developers effectively?

**Answer:**

**Mentoring Principles:**

**1. Establish Trust & Psychological Safety**
```
- Tạo environment nơi mentee cảm thấy safe to ask questions
- Acknowledge rằng không ai biết everything
- Share your own learning journey và mistakes
```

**2. Adapt to Learning Styles**
```
Visual learner: Diagrams, screen sharing, whiteboarding
Hands-on learner: Pair programming, labs
Reading learner: Documentation, code reviews
Auditory learner: Discussions, explanations
```

**3. Balance Support và Independence**
```
KHÔNG làm:
- Viết code cho họ
- Micromanage
- Rush họ

NÊN làm:
- Hướng dẫn approach
- Ask leading questions
- Provide resources
- Let them struggle (trong giới hạn hợp lý)
```

**4. Code Review as Teaching Tool**
```python
# Instead of:
"This is wrong, use X instead"

# Try:
"I see you used approach A. Have you considered approach B?
It could help with X because Y. What do you think?"
```

**5. Set Clear Expectations**
```
Week 1-4: Learn codebase, development workflow
Week 5-8: Own small features với guidance
Week 9-12: Independent work với code review
Month 4+: Gradually increase complexity
```

**6. Regular 1:1s with Structure**
```
Agenda:
1. What you're working on (updates)
2. What's blocking you (help needed)
3. What you've learned (reflection)
4. Career goals discussion (monthly)
```

**7. Give Feedback (SBI Model)**
```
Situation: "Trong PR hôm qua..."
Behavior: "Bạn đã implement error handling rất thoroughly..."
Impact: "Điều này giúp team catch bugs sớm và improve code quality."

For constructive feedback:
Situation: "Trong meeting sáng nay..."
Behavior: "Khi discuss solution, bạn đã dismiss ý kiến khác khá nhanh..."
Impact: "Điều này có thể khiến teammates không muốn share ideas."
```

**8. Celebrate Wins**
```
- Public recognition trong team meeting
- Document growth progress
- Recommend for stretch assignments
```

---

## Question 3: Giải thích approach của bạn với cross-team collaboration?

**Answer:**

**Cross-team Collaboration Framework:**

**1. Establish Communication Channels**
```
Async Communication:
- Shared Slack channels
- Confluence/Wiki documentation
- JIRA/Linear for tracking

Sync Communication:
- Regular sync meetings (weekly/bi-weekly)
- Ad-hoc calls for urgent issues
- Joint planning sessions
```

**2. Create Shared Understanding**
```
- Shared documentation của integration points
- API contracts với versioning
- Shared glossary (alignment on terminology)
- System diagrams visible to all teams
```

**3. Define Interfaces & Contracts**
```yaml
# API Contract Example
openapi: 3.0.0
info:
  title: User Service API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'

# Both teams agree on this contract
# Each team can develop independently
```

**4. Handle Dependencies**
```
Strategies:
- Decouple where possible (async messaging)
- Mock/stub for development
- Feature flags for independent deployment
- Contract testing

Communication:
- Early warning về breaking changes
- Deprecation timeline (3+ sprints)
- Migration support
```

**5. Conflict Resolution**
```
Step 1: Seek to understand
- Why does the other team have this constraint?
- What are their priorities?

Step 2: Find common ground
- Shared goals and objectives
- Win-win solutions

Step 3: Escalate constructively
- Bring options, not just problems
- Data-driven arguments
- Involve leadership if needed
```

**6. Build Relationships**
```
- Informal interactions (coffee chats, lunch)
- Celebrate shared wins
- Cross-team knowledge sharing
- Understand their challenges
```

**Example Response:**
"Trong dự án integration với team Platform, tôi đã establish weekly sync meeting để discuss blockers. Chúng tôi cùng tạo API contract documentation và sử dụng contract testing để ensure compatibility. Khi có disagreement về ownership, tôi đã arrange meeting với cả hai team leads để discuss và reach consensus dựa trên long-term maintenance considerations."

---

## Question 4: Trade-off analysis - Làm thế nào để evaluate và communicate trade-offs?

**Answer:**

**Trade-off Analysis Framework:**

**1. Identify the Trade-off Dimensions**
```
Common trade-offs:
- Speed vs Quality
- Flexibility vs Simplicity
- Cost vs Performance
- Short-term vs Long-term
- Build vs Buy
- Consistency vs Availability (CAP)
```

**2. Quantify When Possible**
```
Option A: Custom Solution
- Development: 3 months, 2 engineers
- Maintenance: 20% time ongoing
- Risk: Medium (unknown unknowns)
- Flexibility: High

Option B: Third-party Service
- Integration: 2 weeks, 1 engineer
- Cost: $500/month
- Risk: Low (proven solution)
- Flexibility: Limited to vendor features
```

**3. Consider Multiple Perspectives**
```
Engineering:
- Technical debt implications
- Scalability concerns
- Team expertise

Product:
- Time to market
- User experience
- Feature flexibility

Business:
- Total cost of ownership
- Strategic alignment
- Risk tolerance
```

**4. Document với DACI Framework**
```
D - Driver: Ai driving decision này?
A - Approver: Ai có final approval?
C - Contributors: Ai provides input?
I - Informed: Ai cần được inform?
```

**5. Present Trade-offs Clearly**
```markdown
## Decision: Caching Strategy

### Option 1: Redis Cluster
**Pros:**
- Proven at scale
- Rich data structures
- Team familiar

**Cons:**
- Additional infrastructure
- Operational overhead

**Recommendation:** [Your recommendation với reasoning]

### Option 2: In-memory Caching
**Pros:**
- Simple implementation
- No external dependencies

**Cons:**
- Memory constraints
- Cache invalidation complexity
```

**6. Make Reversibility Clear**
```
Level 1: Easily reversible (config change)
Level 2: Moderate effort to reverse (refactor)
Level 3: Difficult to reverse (rewrite)

"This decision is Level 2 - we can change framework
but would require 2-3 sprints of work."
```

**7. Follow Up**
```
- Document actual outcomes
- Compare with predictions
- Learn for future decisions
```

---

## Question 5: Project estimation và planning approach?

**Answer:**

**Estimation Techniques:**

**1. Bottom-up Estimation**
```
Break down to small tasks (< 1 day ideally)
Estimate each task
Sum up and add buffer

Example:
- Setup project structure: 4h
- Implement API endpoints: 16h
- Database schema: 8h
- Testing: 12h
- Integration: 8h
- Buffer (20-30%): 10h
Total: 58h ≈ 1.5 weeks
```

**2. Three-Point Estimation**
```
Optimistic (O): Best case scenario
Pessimistic (P): Worst case scenario
Most Likely (M): Realistic estimate

Expected = (O + 4M + P) / 6

Example:
Feature X:
O = 3 days, M = 5 days, P = 10 days
Expected = (3 + 20 + 10) / 6 = 5.5 days
```

**3. T-shirt Sizing**
```
XS: < 1 day
S: 1-2 days
M: 3-5 days
L: 1-2 weeks
XL: > 2 weeks (break down further)

Use for high-level planning, refine later
```

**4. Reference-based Estimation**
```
"Feature Y took 2 weeks last quarter"
"This is similar but with additional complexity"
"Estimate: 3 weeks"
```

**Dealing with Uncertainty:**

```
Cone of Uncertainty:
Initial: 0.25x - 4x actual
After requirements: 0.5x - 2x
After design: 0.8x - 1.25x

Communication:
"Based on current information, estimate is 3-5 weeks.
This will be refined after technical investigation."
```

**Planning Best Practices:**

**1. Iterate and Refine**
```
Sprint 0: High-level estimate (±50%)
After design: Refined estimate (±25%)
After prototype: Final estimate (±10%)
```

**2. Include Hidden Work**
```
- Code review time
- Testing (unit, integration, e2e)
- Documentation
- Deployment and monitoring
- Bug fixes and iterations
- Meetings and communication
```

**3. Account for Velocity Factors**
```
Team availability:
- PTO, sick leave (~10-15%)
- Meetings (~20%)
- Interrupt-driven work

Actual coding: 50-60% of time

5 developers × 2 weeks ≠ 10 engineer-weeks
Reality: ~6-7 engineer-weeks of output
```

**4. Risk-Based Planning**
```
Identify risks:
- External dependencies
- New technology
- Unclear requirements

Mitigation:
- Spike/prototype for unknowns
- Early integration testing
- Frequent stakeholder feedback
```

**5. Communicate Confidently**
```
Bad: "I think maybe 2-3 weeks, possibly more?"

Good: "Based on our initial analysis, I estimate
3-4 weeks. The main risks are API integration
with third-party service. I'll have a more
refined estimate after the spike next week."
```

---

## Question 6: Làm thế nào để handle disagreements trong technical discussions?

**Answer:**

**Approach to Technical Disagreements:**

**1. Assume Positive Intent**
```
- Everyone wants the best outcome
- Different perspectives are valuable
- Disagreement ≠ personal attack
```

**2. Seek to Understand First**
```
Ask clarifying questions:
- "Help me understand why you prefer this approach?"
- "What concerns do you have about the other option?"
- "What's driving this requirement?"

Listen actively:
- Don't interrupt
- Summarize their point before responding
- Acknowledge valid points
```

**3. Focus on Facts, Not Opinions**
```
Replace:
"I think X is better"

With:
"Looking at our benchmark data, X handles
10x more requests per second than Y in similar
use cases. Here's the test setup..."
```

**4. Use Decision Criteria**
```
Agree on criteria first:
1. Performance requirements
2. Maintenance burden
3. Team expertise
4. Time to market

Then evaluate options against criteria
```

**5. Propose Experiments**
```
"Instead of debating, let's prototype both
approaches for 2 days and compare results"

Spike/POC resolves many technical debates
```

**6. Disagree and Commit**
```
When decision is made, support it fully:
- Document dissenting view (for future reference)
- Commit to making chosen approach successful
- Don't undermine after decision
```

**7. Escalate Constructively**
```
When:
- Team cannot reach consensus
- Decision has significant impact
- Blocking progress

How:
- Present both options clearly
- Show analysis done
- Ask for guidance, not arbitration
```

**Example Scenario:**
```
Colleague: "We should use NoSQL for this"
You: "I hear you prefer NoSQL. Can you help me
understand the specific benefits you see for our
use case? I have some concerns about the
transaction requirements we discussed."

After discussion:
"Both approaches have merit. Let's agree on
the evaluation criteria: consistency requirements,
query patterns, and team expertise. Then we can
score each option and make a data-driven decision."
```

**8. Know When to Let Go**
```
Not every hill is worth dying on
Consider:
- Impact of decision
- Reversibility
- Your confidence level

Sometimes: "I've shared my concerns. If the team
prefers this approach, I'll support it fully."
```

---

## Question 7: Time management và prioritization cho tech leads?

**Answer:**

**Time Management Framework:**

**1. Eisenhower Matrix**
```
                  Urgent          Not Urgent
Important    │ DO NOW         │ SCHEDULE      │
             │ Production     │ Architecture  │
             │ incident       │ planning      │
─────────────┼────────────────┼───────────────│
Not Important│ DELEGATE       │ ELIMINATE     │
             │ Routine        │ Unnecessary   │
             │ meetings       │ meetings      │
```

**2. Time Blocking**
```
Sample Tech Lead Schedule:

AM Block (Focus):
9:00-11:00 - Deep work (code, architecture)
11:00-12:00 - 1:1s with direct reports

PM Block (Collaboration):
13:00-15:00 - Team meetings, reviews
15:00-16:00 - Cross-team collaboration
16:00-17:00 - Admin, emails, planning
```

**3. Protect Maker Time**
```
- No meetings before 11am (if possible)
- Batch similar tasks
- Use "Focus time" blocks in calendar
- Communicate availability windows
```

**4. Delegation Framework**
```
TASK Assessment:
- Can someone else do this? → Delegate
- Can someone learn from this? → Delegate with coaching
- Am I the only one who can do this? → Keep it

Delegation steps:
1. Clear outcome definition
2. Provide context and resources
3. Agree on check-in points
4. Give authority to make decisions
5. Follow up but don't micromanage
```

**5. Prioritization Techniques**
```
MoSCoW:
- Must have: Absolutely required
- Should have: Important but not critical
- Could have: Nice to have
- Won't have: Out of scope (for now)

Impact/Effort Matrix:
Quick wins (High impact, Low effort) → Do first
Major projects (High impact, High effort) → Plan carefully
Fill-ins (Low impact, Low effort) → When time permits
Thankless tasks (Low impact, High effort) → Avoid/delegate
```

**6. Manage Interrupts**
```
Strategies:
- Batch similar interrupts
- "Office hours" for team questions
- Slack delay (not always-on)
- Handoff to team members when possible

Response time expectations:
Severity 1: Immediate
Slack DM: 1-2 hours
Email: Same day
```

**7. Weekly Review & Planning**
```
Friday:
- What did I accomplish?
- What's still pending?
- What did I learn?

Monday:
- What are the top 3 priorities?
- What meetings can I decline?
- Who needs my support?
```

**8. Energy Management**
```
Know your peak hours:
- Complex work during peak energy
- Routine tasks during low energy
- Take breaks (Pomodoro: 25min work, 5min break)
```

---

## Question 8: Làm thế nào để build và maintain high-performing engineering team?

**Answer:**

**Building High-Performing Teams:**

**1. Hiring Right**
```
Technical skills are necessary but not sufficient
Look for:
- Growth mindset
- Collaboration ability
- Communication skills
- Cultural fit
- Problem-solving approach

Red flags:
- Cannot admit not knowing something
- Blames others for failures
- Not curious about how things work
```

**2. Psychological Safety**
```
Definition: Team members feel safe to take risks

How to build:
- Admit your own mistakes
- Celebrate learning from failures
- Never punish for raising issues
- Encourage dissenting opinions

Example:
"I made a mistake last week by rushing the
deployment. Here's what I learned and how
we'll prevent it. Has anyone else faced
similar situations?"
```

**3. Clear Goals & Alignment**
```
Team understands:
- Company mission/vision
- Team's role in bigger picture
- Individual contribution to team goals

OKRs/Goals should be:
- Specific and measurable
- Challenging but achievable
- Team-owned (not imposed)
```

**4. Engineering Excellence**
```
Practices:
- Code review culture
- Testing standards
- Documentation habits
- Blameless post-mortems
- Technical debt management

Metrics (use carefully):
- Deployment frequency
- Lead time for changes
- Mean time to recovery
- Change failure rate
```

**5. Growth & Development**
```
For each team member:
- Understand career goals
- Provide stretch assignments
- Allocate learning time
- Give regular feedback
- Support conference/training

Growth paths:
- IC track (Senior → Staff → Principal)
- Management track (Tech Lead → EM → Director)
```

**6. Team Rituals**
```
- Daily standups (keep short)
- Sprint planning/retro
- Demo/show & tell
- Tech talks
- Team celebrations
```

**7. Handle Low Performance**
```
Step 1: Direct feedback (specific examples)
Step 2: Create improvement plan với clear metrics
Step 3: Regular check-ins on progress
Step 4: Make decision (extend, adjust role, exit)

Document everything
```

**8. Measure Team Health**
```
Surveys (quarterly):
- Do you feel heard?
- Do you understand team goals?
- Are you growing professionally?
- Would you recommend this team?

Watch for signals:
- Increased turnover
- Decreased engagement
- Quality decline
- Conflict patterns
```

---

## Question 9: Stakeholder management cho engineering leaders?

**Answer:**

**Stakeholder Management Framework:**

**1. Identify Stakeholders**
```
Map stakeholders:
- Direct stakeholders: Product, Design, QA
- Executive stakeholders: CTO, VP Engineering
- Cross-functional: Marketing, Sales, Customer Success
- External: Customers, Partners

Power/Interest Grid:
                High Power
                    │
    Keep Satisfied  │  Manage Closely
    (Executives)    │  (Key partners)
─────────────────────┼─────────────────────
    Monitor         │  Keep Informed
    (Low priority)  │  (Team updates)
                    │
                Low Power
```

**2. Understand Their Priorities**
```
Product Manager:
- Time to market
- Feature completeness
- Customer satisfaction

Business/Sales:
- Revenue impact
- Competitive advantage
- Client commitments

Finance:
- Budget compliance
- ROI

Executive:
- Strategic alignment
- Risk management
- Big picture progress
```

**3. Communication Strategies**
```
Tailor message to audience:

To engineers:
"We're implementing a distributed caching layer
using Redis Cluster for horizontal scaling"

To product:
"This improvement will reduce load times by 50%
and enable us to handle Black Friday traffic"

To executives:
"This $50K investment in infrastructure will
support 10x user growth and prevent $500K
revenue loss from outages"
```

**4. Manage Expectations**
```
Under-promise, over-deliver:
- Provide realistic estimates
- Communicate risks early
- Update frequently on progress

Bad news early:
"I want to flag that we're at risk of missing
the deadline. Here's why and here are our
options: [A, B, C]. My recommendation is B."
```

**5. Build Credibility**
```
- Deliver on commitments
- Be transparent about challenges
- Provide solutions, not just problems
- Follow up on action items
- Share credit widely
```

**6. Handle Conflicting Demands**
```
When stakeholders want different things:

Step 1: Document conflicts clearly
Step 2: Facilitate discussion between parties
Step 3: If no resolution, escalate with options
Step 4: Get decision documented

"Product wants Feature A, but Sales committed
to Client X for Feature B. We can only do one
this quarter. I need a prioritization decision."
```

**7. Regular Updates**
```
Weekly status (for key stakeholders):
- What we accomplished
- What's next
- Blockers/needs

Monthly/Quarterly:
- Progress against goals
- Team health
- Resource needs
- Technical roadmap
```

**8. Saying No Effectively**
```
Not: "No, we can't do that"

Better: "We can't do X in the requested timeline
because [reason]. However, we could:
1. Do a smaller version by [date]
2. Do the full version by [later date]
3. Deprioritize Y to make room for X

What would work best for you?"
```

---

## Question 10: Incident management và blameless post-mortems?

**Answer:**

**Incident Management Process:**

**1. Detection & Triage**
```
Detection sources:
- Monitoring alerts
- Customer reports
- Internal reports

Severity levels:
SEV1: Service down, major customer impact
SEV2: Degraded service, significant impact
SEV3: Minor issue, limited impact
SEV4: Cosmetic/minor bug

Response time by severity:
SEV1: Immediate, all hands
SEV2: Within 30 minutes
SEV3: Same day
SEV4: Next sprint
```

**2. Incident Response Roles**
```
Incident Commander (IC):
- Coordinates response
- Makes decisions
- Communicates status

Technical Lead:
- Leads investigation
- Proposes mitigations

Communications Lead:
- Updates stakeholders
- Drafts customer communications
```

**3. During Incident**
```
DO:
- Focus on mitigation first (restore service)
- Document timeline and actions
- Communicate frequently
- Keep war room focused

DON'T:
- Look for root cause while service is down
- Blame individuals
- Make unnecessary changes
- Go silent on updates
```

**4. Communication Template**
```
Status Update:
- Current status: [Investigating/Identified/Monitoring/Resolved]
- Impact: [What's affected]
- Timeline: [When it started]
- Current actions: [What we're doing]
- Next update: [When]

Example:
"[10:30 AM] Status: Identified
Impact: API latency increased for 30% of requests
Cause: Database connection pool exhausted
Action: Scaling database, ETA 15 minutes
Next update: 10:45 AM"
```

**5. Blameless Post-mortem**
```
Goals:
- Learn from incident
- Prevent recurrence
- Improve processes
- NOT to assign blame

Schedule within 48-72 hours of resolution
```

**Post-mortem Template:**
```markdown
# Post-mortem: Database Outage 2024-01-15

## Summary
[2-3 sentence summary of what happened]

## Impact
- Duration: 45 minutes
- Users affected: 10,000
- Revenue impact: ~$5,000

## Timeline (all times UTC)
- 10:00 - Alert fires for high DB latency
- 10:05 - On-call engineer acknowledges
- 10:15 - Root cause identified
- 10:30 - Mitigation applied
- 10:45 - Full recovery confirmed

## Root Cause
Connection pool size not scaled with traffic growth

## Contributing Factors
- No alerting on connection pool utilization
- Recent traffic increase not communicated to infra team

## What Went Well
- Quick detection (5 min)
- Effective war room coordination
- Clear communication to customers

## What Went Wrong
- 15 min to identify root cause
- No runbook for this scenario

## Action Items
| Action | Owner | Due Date |
|--------|-------|----------|
| Add connection pool monitoring | @alice | 2024-01-20 |
| Update capacity planning doc | @bob | 2024-01-22 |
| Create runbook for DB issues | @carol | 2024-01-25 |
```

**6. Post-mortem Meeting**
```
Facilitation tips:
- "How" not "Who" - "How did this slip through?" not "Who missed this?"
- Psychological safety - No blame, focus on systems
- Everyone's voice matters
- Focus on learning, not punishment

Questions to ask:
- What circumstances led to this?
- What information would have helped?
- What would you do differently?
- What systemic changes would prevent this?
```

**7. Follow-up**
```
- Track action items to completion
- Review recurring incidents for patterns
- Share learnings across teams
- Update documentation and runbooks
- Adjust monitoring/alerting
```

---
