# Contribution 1: Add evidence-checklist command to iso27001 plugin

**Contribution Number:** 1  
**Student:** Srujana Kethamukkala  
**Issue:** [[GitHub issue link](https://github.com/GRCEngClub/claude-grc-engineering/issues/162)]  
**Status:** Phase I Complete

---

## Why I Chose This Issue
This issue is a great fit for getting familiar with the codebase structure. The task is well-scoped, I just need to follow an existing pattern from GDPR/HITRUST plugins to build a parallel command for ISO 27001. Since audit prep is central to GRC engineering, this also gives me a chance to understand how evidence collection workflows are modeled in the plugin system.

---

## Understanding the Issue

### Problem Description

The iso27001 framework plugin is missing an evidence-checklist command, even though other major framework plugins (GDPR, HITRUST, CMMC, NIST CSF 2.0) all ship one. This command is the most-requested feature for ISO 27001:2022 audit prep.

### Expected Behavior

Running /iso27001:evidence-checklist with an Annex A control ID (e.g., A.5.15) or domain (e.g., Organizational, Technological) should generate an audit-ready evidence collection list, optionally exportable as markdown, json, or csv. The output format and file structure should match the conventions already established by the GDPR and HITRUST plugins.

### Current Behavior

The iso27001 plugin has no evidence-checklist.md command file. The command doesn't exist, so users working on ISO 27001:2022 audits have no plugin-native way to generate structured evidence checklists.

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
