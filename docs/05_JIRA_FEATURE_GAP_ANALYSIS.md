# Curatal WorkHub — Phase 0: Jira Feature Gap Analysis

**Document ID:** `docs/05_JIRA_FEATURE_GAP_ANALYSIS.md`  
**Platform Version:** OpenProject Community Edition v17.9.1  
**Target Organization:** Curatal  
**Author:** Senior Software Architect & Product Systems Engineer  

---

## 1. Executive Summary

This gap analysis establishes the direct mapping between Jira project management capabilities and OpenProject Community Edition v17.9.1. It distinguishes what is natively available out of the box, what requires administrative configuration, what requires Community-compatible custom development, and what is restricted behind OpenProject Enterprise licensing.

**Strategic Conclusion:**
OpenProject Community Edition provides ~80% of Curatal's core Jira requirements (work package hierarchy, custom workflows, issue relations/blockers, backlogs, versions/releases, time tracking, and API integrations) natively or via configuration. The primary gaps relate to Jira-like automated board actions, specialized QA test cycle management, and native two-way GitHub pull request status transitions—all of which can be addressed via Curatal's planned extensibility roadmap without violating GPL-3.0 licenses.

---

## 2. Jira vs. OpenProject Capability Matrix

### Classification Legend
1. **Available out of the box (OOTB):** Ready immediately with default installation.
2. **Available through configuration (Config):** Supported natively via admin settings, type definitions, custom fields, or workflow transition maps.
3. **Available but limited (Limited):** Functional, but lacks specific Jira conveniences or advanced automation without extensions.
4. **Requires custom development (Custom):** Not present in core Community Edition; planned for Curatal WorkHub extensions.
5. **Enterprise-only (EE):** Restricted upstream to paid licenses; cannot be assumed available.

---

### 2.1 Core Issue & Work Package Management

| Jira Feature | OpenProject Entity / Counterpart | Classification | Curatal WorkHub Assessment & Configuration Strategy |
| :--- | :--- | :--- | :--- |
| **Projects & Subprojects** | Projects & Nested Subprojects | **Available out of the box** | Unlimited hierarchical nesting with inherited or isolated permissions. |
| **Project Keys (e.g. CORE-101)** | Work Package ID (`#101`) | **Available but limited** | OpenProject identifies work packages globally by ID (`#123`). Project identifier prefixes (e.g. `CORE-123`) require custom display formatter or configuration. |
| **Epics** | Work Package Type: `Epic` | **Available through configuration** | Enabled via Admin → Work Packages → Types. Formatted with color badges and parent-child hierarchy. |
| **User Stories** | Work Package Type: `User Story` | **Available through configuration** | Seeded or configured as primary agile requirement item. |
| **Tasks & Subtasks** | Tasks & Subtasks (`Parent-Child` relation) | **Available out of the box** | Fully hierarchical; parent automatically aggregates dates and estimates. |
| **Bugs & Enhancements** | Work Package Types: `Bug`, `Feature` | **Available out of the box** | Default seeded types in OpenProject. |
| **Custom Work Package Types** | Custom Types | **Available out of the box** | Unlimited custom types with customizable form schemas and attributes. |
| **Priorities** | Work Package Priorities | **Available out of the box** | Configurable levels: Low, Normal, High, Urgent, Immediate. |
| **Custom Statuses & Workflows** | Statuses & Workflow Transition Matrix | **Available through configuration** | Role-based status transition matrix maps exact Curatal development-to-QA lifecycle. |
| **Story Points** | Custom Field / Backlogs Module | **Available through configuration** | Configured as integer custom field or via Backlogs plugin. |

---

### 2.2 Agile Boards & Sprints

| Jira Feature | OpenProject Entity / Counterpart | Classification | Curatal WorkHub Assessment & Configuration Strategy |
| :--- | :--- | :--- | :--- |
| **Basic Kanban Boards** | Work Package Filtered Views / Basic Boards | **Available out of the box** | Card lists grouped by status; cards can be dragged across status columns. |
| **Advanced Action Boards** | Enterprise Action Boards | **Enterprise-only** | Auto-assignment on drag or boards partitioned by assignee/version is EE-only. In CE, basic status boards work reliably. |
| **Scrum & Sprints** | Backlogs Module (Sprints) | **Available through configuration** | Native Backlogs module provides sprint planning, backlog grooming, and burndown charts. |
| **Backlog Management** | Backlog View | **Available through configuration** | Product backlog with drag-and-drop prioritization into sprint buckets. |

---

### 2.3 Dependencies, Blockers & Releases

| Jira Feature | OpenProject Entity / Counterpart | Classification | Curatal WorkHub Assessment & Configuration Strategy |
| :--- | :--- | :--- | :--- |
| **Issue Dependencies** | Relations: `precedes`, `follows` | **Available out of the box** | Date-shifting dependency logic automatically cascades timeline changes. |
| **Blockers** | Relation: `blocks`, `blocked by` | **Available out of the box** | Clear dependency links showing blocker work packages. |
| **Roadmap & Gantt** | Gantt View & Timelines | **Available out of the box** | Interactive Gantt chart with drag-and-drop bar resizing and milestone markers. |
| **Versions & Releases** | Project Versions | **Available out of the box** | Semantic releases (e.g. `v1.0.0`) with milestone dates, status tracking, and burndown. |

---

### 2.4 Collaboration, QA & Integrations

| Jira Feature | OpenProject Entity / Counterpart | Classification | Curatal WorkHub Assessment & Configuration Strategy |
| :--- | :--- | :--- | :--- |
| **Comments & Attachments** | Activity Stream / Attachments | **Available out of the box** | Rich markdown/WYSIWYG editor with live collaborative editing (Hocuspocus). |
| **Reopening Issues** | Status transition `Closed` → `Reopened` | **Available through configuration** | Configured in workflow matrix to allow QA and engineers to reopen failed bugs. |
| **Notifications** | In-app Notification Center & Email | **Available out of the box** | Real-time bell notification bell, split-screen review, and SMTP email digests. |
| **Time Tracking** | Spent Time & Cost Tracking | **Available out of the box** | Log work hours against work packages, users, and activity categories. |
| **Granular Permissions** | Project Roles & Permissions | **Available out of the box** | Multi-role permissions (e.g. Developer, QA, Scrum Master, Product Owner). |
| **GitHub Integration** | Webhook module / OpenProject GitHub Action | **Requires custom development** | Upstream has basic repository viewer; two-way PR sync and issue auto-closure will be built in Phase 2 using Webhooks and GitHub Actions. |
| **REST API** | OpenProject REST API v3 (HAL+JSON) | **Available out of the box** | Full API coverage for automation and connecting Curatal portals. |
| **Webhooks** | `openproject-webhooks` module | **Available out of the box** | Event delivery on work package and project mutations. |
| **QA Test Case Management** | Custom Type: `Test Case` / `Test Run` | **Requires custom development** | Jira utilizes Xray/Zephyr plugins. In WorkHub, Phase 2 will implement lightweight Test Run custom types with pass/fail executions. |

---

## 3. High-Priority Curatal Customizations Roadmap

To transform OpenProject Community Edition into an optimal Jira replacement for Curatal:

1. **Phase 1: Workflow & Data Model Configuration (Zero-Code)**
   - Configure Jira-standard work package types (`Epic`, `User Story`, `Task`, `Bug`, `Subtask`).
   - Define custom fields (`Story Points`, `Root Cause`, `QA Environment`, `Regression Tested`).
   - Implement Curatal's 10-state Development-to-QA workflow with role transition gates.
2. **Phase 2: Curatal WorkHub Extensions & Integrations (Custom Code)**
   - **GitHub Integration Service:** Webhook listener linking Git branch names and PR descriptions (`feat/CORE-101`) to WorkHub work packages, updating status to `Code Review` upon PR creation and `Ready for QA` on merge.
   - **Project Key Formatter:** UI extension rendering `#101` as `CORE-101` matching project keys.
   - **Lightweight QA Test Suite:** Custom work package relations for executing reproducible test test plans.
