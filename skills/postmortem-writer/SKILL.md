---
name: postmortem-writer
description: Assists the user in writing high-quality, blameless, and actionable incident postmortems (RCA reports) by conducting an interactive interview and generating a structured markdown file.
---

# Postmortem Writer Skill

This skill guides you (the AI agent) in collaborating with a user to author a "gold standard" incident postmortem. Your goal is to produce a highly structured, blameless, and actionable markdown document that focuses on system failure analysis and prevention.

## Core Principles to Enforce

1. **Blamelessness (Non-Negotiable)**
   * Avoid language that attributes fault to individuals. Use systemic phrasing.
   * *Bad*: "The engineer accidentally wiped the primary database."
   * *Good*: "The database directory on the primary host was deleted during a manual replication recovery process due to identical shell prompts on the primary and secondary terminals."
   * Assume competent people worked with incomplete information. Focus on the conditions and tooling that allowed the action to be harmful.

2. **Clarity and Precision**
   * Keep timestamps clear and consistent (prefer UTC with explicit offsets).
   * Focus on key decision points, state changes, and discovery milestones rather than verbose narrative diaries.

3. **Causality over Chronology**
   * Differentiate between the **trigger event** (what set off the failure) and **systemic contributing factors** (what made the failure possible or worse).
   * Map dependencies, silent failures, and why defenses/safeguards failed to function as intended.

4. **Actionability**
   * Every action item must be concrete, verifiable, assigned to an owner, and given a priority and target completion timeline.

---

## Postmortem Formats & Styles

This skill supports two distinct templates depending on the user's needs or the nature of the incident:

1. **Standard Form (Operational/Metric-Driven):**
   * *Best For:* Single isolated incidents with clear timelines, discrete root causes, and concrete operational metrics.
   * *Style:* Structural, concise, metadata-heavy, table-driven.
   * *Example Reference:* [gitlab_database_outage_2017.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/postmortem-writer/examples/gitlab_database_outage_2017.md)

2. **Narrative / Long-Form Retrospective (Story-Driven):**
   * *Best For:* Multi-incident sequences, complex systemic cascades, operations under continuous scale shifts/migrations, and team/organizational learning.
   * *Style:* Storytelling, episodic breakdowns, structured as an unfolding mystery (revealing clues in stages to keep the reader engaged, saving the actual root cause for the end), focus on developer cognitive load, team pace/tempo, and combinatorial scale limitations.
   * *Example Reference:* [honeycomb_twenty_fires_2021.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/postmortem-writer/examples/honeycomb_twenty_fires_2021.md)

---

## Interactive Writing Workflow

You must **not** generate a completed postmortem with placeholder values or make assumptions about the incident. Instead, guide the user through a structured, section-by-section interview:

### Phase 1: Format Selection & Setup
Ask the user which style is most appropriate for their retrospective:
* **Standard Form** or **Narrative / Long-Form**?
* Request the basic metadata (Incident Date, Severity, Systems involved).

### Phase 2: Fact Gathering (Interactive Interview)

#### If Standard Form is selected:
Ask questions for each of the following in sequence:
1. **Summary & Impact**: User-facing symptoms, start/end/recovery times, quantitative metrics.
2. **Timeline**: Chronology of events, alerts, human decisions, and key state changes.
3. **Causal Analysis**: Trigger event, systemic causes, contributing factors, and why safeguards failed.
4. **Detection & Response**: Detection alerts, runbook accuracy, on-call coordination.
5. **Lessons Learned & Action Items**: What went well, what went wrong, and owned, verifiable action items.

#### If Narrative / Long-Form is selected:
Focus on the storytelling aspects, structured as an unfolding mystery. Ask questions in sequence for:
1. **Continuous Change Context**: What background shifts (migrations, scale growth, feature velocity) was the team navigating before the fires started?
2. **The Incidents / Episodes**: Step through the timeline as "episodes" or interconnected stories. What did responders see? What red herrings did they chase? Where did they hit blind spots? Ensure that clues are revealed in stages, keeping the reader engaged without jumping to the actual cause.
3. **The Reveal / Root Cause**: What was the actual root cause? (This must be saved for the end of the narrative sequence).
4. **Causal / Architectural Dimensions**: How did components collide (combinatorial scaling)? E.g., did horizontal scaling on one side spike latency or noise on the other?
5. **Team Pace & Human Dynamics**: What was the operational tempo/burnout rate? How did cognitive load shift between incidents?
6. **Lessons & Actions**: Systemic findings, goldilocks operational boundary, and owned, verifiable action items (using checkboxes instead of AI-x IDs).

### Phase 3: Drafting the Postmortem
Compile the gathered information into a Markdown file. Follow the corresponding template below.

### Phase 4: Quality Review Check
Before presenting the final postmortem, review it against the **Quality Checklist** below.

---

## Template A: Standard Operational Template

```markdown
# Incident Postmortem: [Incident Name / Brief Title]

**Date of Report:** YYYY-MM-DD
**Incident Date:** YYYY-MM-DD
**Severity:** [e.g., SEV-1 - Critical]
**Total Duration:** [X hours, Y minutes]
**Lead Investigators:** [Names/Teams]

---

## 1. Executive Summary
*A 3-sentence summary of the incident containing what happened, why it happened, the customer/business impact, and how it was resolved.*

* **Start Time (T0):** YYYY-MM-DD HH:MM UTC
* **Detection Time:** YYYY-MM-DD HH:MM UTC
* **Mitigation Time:** YYYY-MM-DD HH:MM UTC
* **Full Recovery Time:** YYYY-MM-DD HH:MM UTC

---

## 2. Customer & Business Impact
* **User-Facing Symptoms:**
* **Affected Services/Dependencies:**
* **Quantitative Metrics:** (e.g., "52% of API requests returned 500 errors for 45 minutes")

---

## 3. Timeline
| Timestamp (UTC) | Event / Action / Decision |
| :--- | :--- |
| **T0** (HH:MM) | **Incident Trigger:** [Description of the triggering event] |
| T+X Min (HH:MM) | **Detection:** [e.g., Alert fires, paging on-call] |
| T+Y Min (HH:MM) | **Mitigation Started:** [e.g., Redirecting traffic] |
| T+Z Min (HH:MM) | **Full Recovery:** [e.g., System healthy] |

---

## 4. Root Cause Analysis
* **The Trigger:** (What initiated the failure?)
* **Systemic Cause:** (Why was the system vulnerable to this trigger?)
* **Contributing Factors:** (What environmental or process conditions worsened the impact?)
* **Why Safeguards Failed:** (Why did existing alerts, rate limiters, or backups fail?)

---

## 5. Detection & Response Analysis
* **Detection Mechanism:** [e.g., Automated Alert / Customer Ticket]
* **Alert Effectiveness:** Did correct alerts fire? Did they fire on time?
* **Runbook & Tooling Adequacy:** Were runbooks available, accurate, and easy to follow?
* **On-Call & Coordination:** How was team coordination and escalation?

---

## 6. Lessons Learned
* **What Went Well:** (Safeguards, tooling, or decisions that worked)
* **Gaps & What Went Wrong:** (Monitoring blindspots, runbook gaps, single points of failure)

---

## 7. Action Items
| Action Item | Owner | Priority | Target Date | Verifiable Outcome / Goal |
| :--- | :--- | :--- | :--- | :--- |
| [ ] Add alert for X | [Name/Team] | High | YYYY-MM-DD | Alert fires in staging under simulation |
```

---

## Template B: Narrative / Long-Form Retrospective Template

```markdown
# Engineering Retrospective: [Title / Narrative Theme]

**Date of Report:** YYYY-MM-DD
**Incident Period:** [e.g., September - October 2021]
**System Background:** [Brief summary of the architecture and key systems involved]

---

## 1. Context of Continuous Change
*Describe the background pressure and environmental context. E.g., What organic growth, infrastructure migrations, or deploy velocities were compounding operational load before the incidents began?*

---

## 2. Chronological Narrative & Deep Dives (The Episodes)
*Present the incident timeline as a series of connected storytelling episodes. Describe what engineers saw, the hypotheses they tested, red herrings, cascades, and local mitigations. Reveal clues in stages to build suspense and keep the reader engaged. Do not jump to the actual cause; keep the reader guessing until the final reveal.*

### Episode 1: [Episode Title, e.g., The Silent Throttling]
* *The Story & Clues:*
* *The Debugging Path:*
* *The Red Herrings:*
* *Immediate Resolution:*

### Episode 2: [Episode Title, e.g., Cascade of the Omitted Cronjob]
* *The Story & Clues:*
* *The Debugging Path:*

---

## 3. The Reveal & Root Cause
*Pull the clues together and reveal the actual root cause of the incident at the end of the narrative.*

---

## 4. Lessons Learned and Architectural Dimensions
*Step back from the specific episodes and analyze the systemic, architectural, and organizational dimensions of the operational month.*

### Dimension A: Component Scaling
*How did individual component behaviors scale? Were there limits that were poorly understood across the team?*

### Dimension B: Combinatorial Scaling & Component Interplay
*Describe how scaling or mitigating one component caused downstream constraints or silent failures on another. (E.g., horizontal scaling on writes bottlenecked reads).*

### Dimension C: Team Pace, Cognitive Load, and Tempo
*How did the pace of alerts affect team health and coordination? What did the experience reveal about the Goldilocks operational zone (operating close to vs. far from system limits)?*

---

## 5. Action Items
*Action items resulting from both operational fixes and systemic optimizations. Use checkboxes instead of AI-x IDs.*

| Action Item | Owner | Priority | Target Date | Verifiable Outcome / Goal |
| :--- | :--- | :--- | :--- | :--- |
| [ ] Automate GC check | [Name/Team] | High | YYYY-MM-DD | Verified via Kubernetes cron migration |
```

---

## Quality Checklist

Before finalizing the postmortem:
- [ ] **Blameless Check**: Is any failure framed as "human error" or an individual's mistake? If so, rephrase it to focus on why the system allowed the mistake (e.g., lack of confirmation prompt, lack of privilege isolation).
- [ ] **Timestamps**: Are all timestamps in a consistent timezone (preferably UTC) and mapped to the timeline?
- [ ] **systemic Causes**: Does the root cause explanation go deeper than the immediate trigger? (e.g., "Out of disk space" is a trigger; "Lack of automated cleanup cron or disk capacity alerts" is the systemic cause).
- [ ] **Action Items**: Ensure every action item uses checkboxes (`[ ]` or `- [ ]`) instead of `AI-` labels. Verify each is assigned to a specific owner and is verifiable. Do not use vague items like "Investigate improving database queries." Instead, use "Optimize query X to run under 50ms and verify with load testing."
- [ ] **Narrative Mystery (for Narrative style)**: Ensure the timeline/episodes do not jump to the actual cause early. Reveal clues in stages to build suspense, and save the actual root cause reveal for the end.

---

## Key References
* **Google SRE Book - Incident Writeups**: [https://sre.google/sre-book/postmortem-culture/](https://sre.google/sre-book/postmortem-culture/)
* **GitLab Database Outage (Jan 31, 2017)**: [https://about.gitlab.com/blog/postmortem-of-database-outage-of-january-31/](https://about.gitlab.com/blog/postmortem-of-database-outage-of-january-31/)
* **Honeycomb 20 Fires Retrospective (Nov 2021)**: [Do You Remember the 20 Fires of September](file:///Users/luiz1361/gh_jae-labs/skills/skills/postmortem-writer/examples/honeycomb_twenty_fires_2021.md)
* **PagerDuty Incident Review Guide**: [https://response.pagerduty.com/review/](https://response.pagerduty.com/review/)

