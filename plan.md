# Multi-Department Ticketing System with Service Catalogue

## 1. Document Status
- **Status:** Draft v0.1
- **Language:** English
- **Purpose:** Planning and implementation guidance for the database-first design of a multi-department ticketing system with Service Catalogue support.
- **Audience:** Database engineers, system analysts, technical leads, reviewers, and implementation tools such as OpenCode.

---

## 2. Executive Summary
This project will deliver a **database-centered ticketing system** for multiple departments, divisions, and sections, with support for a **Service Catalogue**, **hierarchical routing**, **arbitration**, **clarification requests**, **child tickets**, **SLA tracking**, and **quality verification before final closure**.

The system must remain aligned with the existing enterprise organizational model and security architecture. It will not introduce new user, distributor, or role identity models. Instead, it will reuse the current organizational hierarchy and authorization structures, especially:
- `Idara -> Department -> Division/Section`
- `DSDID` as the true routing reference
- `Distributor` as the organizational receiver / queue identity
- `UserDistributor` as the bridge between a real person and their organizational position or function

The first version must stay practical and controlled:
- simple for end users
- structured for management
- auditable for operations
- extensible for future workflow maturity

---

## 3. Business Problem
The business needs a ticketing system that can:
1. Receive requests from either **residents/beneficiaries** or **internal users**.
2. Route known services automatically based on predefined organizational ownership.
3. Route unknown services (`Other`) to the appropriate arbitrator.
4. Allow departments, divisions, and sections to challenge wrong routing.
5. Distinguish between:
   - wrong responsibility assignment
   - insufficient information
   - dependency-based delays
6. Support operational execution by internal staff.
7. Support parent/child ticket relationships where one ticket may block another.
8. Prevent unfair accountability when delays are caused by another unit, missing data, approvals, or dependent work.
9. Preserve a complete audit trail for all important actions.
10. Improve the service catalogue over time based on repeated real-world cases.

---

## 4. Scope

### 4.1 In Scope
- Database design only
- New `[Tickets]` schema objects
- Master tables, transaction tables, history tables, lookup tables, views, DL procedures, and SP procedures
- Ticket lifecycle management
- Service catalogue and service routing rules
- Arbitration workflow for scope disputes
- Clarification workflow for missing information
- Parent-child ticket model
- SLA clocks and pause/resume behavior
- Quality review before final closure
- Reporting support and dashboard-oriented read models
- Full audit logging through stored procedures

### 4.2 Out of Scope for V1
- Full application UI implementation
- CMDB / advanced asset management
- Dynamic per-service form templates
- Automatic escalations
- Backlog / claim mode inside sections
- Notification engine implementation
- File attachment subsystem
- Customer satisfaction surveys

---

## 5. Key Design Principles
1. **Database-first implementation**
   - All business write operations must be executed via stored procedures.
   - Read operations should use DL procedures and views where appropriate.

2. **No new identity model**
   - Reuse existing `User`, `Distributor`, `UserDistributor`, and organizational hierarchy structures.

3. **`DSDID` is the source of truth for routing**
   - Routing level labels must not replace the real target organizational node.

4. **Queue routing is different from execution assignment**
   - Tickets arrive first to an organizational queue.
   - Execution is later assigned to a real user.

5. **Do not mix different operational problems into one state**
   - Wrong routing
   - Missing information
   - Dependency/blocking
   must be modeled separately.

6. **Auditability is mandatory**
   - Every meaningful change must be traceable.

7. **Start simple, remain extensible**
   - V1 should be implementable and usable.
   - Future capabilities should be enabled by structure, not forced immediately.

---

## 6. Confirmed Functional Model

### 6.1 Request Sources
A ticket may be opened by:
- a **resident / beneficiary**
- an **internal user**

The design must support both, even if the first rollout activates only one source type initially.

### 6.2 Service Selection
A requester may:
1. choose a known service from the catalogue, or
2. choose **Other** and describe the issue manually.

### 6.3 Default Routing
If the service exists in the catalogue, the system should route the ticket using a predefined **default routing rule**, which may target:
- a section directly
- a division directly
- a department directly

### 6.4 Arbitrated Routing
If the requester selects **Other**, or if the target is unknown, the ticket must be routed to an **arbitrator** at the appropriate level.

### 6.5 Organizational Receiving Model
The first organizational receiver is treated as a **queue/inbox receiver**, not necessarily the final owner.

This means:
- the ticket arrives at the proper organizational level
- the manager/supervisor reviews it
- then it is assigned to an eligible real user if appropriate

### 6.6 Execution Assignment
A ticket may be assigned directly to a `UserID`, but only if that user is valid for the current organizational scope through `UserDistributor`.

Examples:
- a division manager may assign directly to a user attached to the division itself
- if the real worker belongs to a section, the ticket should first be routed to that section, then assigned by the section supervisor

### 6.7 Rejection / Challenge Rules
- An execution user may reject a ticket back to their supervisor/manager.
- The execution user may not send the ticket directly to arbitration.
- Scope disputes must be escalated through the supervisory chain.

### 6.8 Clarification Rules
Insufficient information must be treated separately from scope disputes.

Examples:
- the ticket belongs to the correct unit, but required technical details are missing
- the missing details may need to be supplied by the parent ticket owner or the sending unit
- end beneficiaries are not expected to provide technical specifications

### 6.9 Child Tickets
A user may request creation of a child ticket, but approval from the direct supervisor/manager is required.

A child ticket may target:
- a department
- a division
- a section

A child ticket belongs to **one parent ticket only**.

### 6.10 Blocking Logic
A parent ticket may be paused because of:
- a dependent child ticket
- arbitration
- clarification
- warehouse delay
- approval delay
- external dependency

### 6.11 Closure Model
Closure is two-stage:
1. **Operational closure** by the executor
2. **Final closure / verification** by quality, monitoring, or another authorized reviewer

---

## 7. Confirmed Data Architecture Decisions

### 7.1 Schema Strategy
All new tables will be created under:
- `[Tickets]`

Existing reference tables remain in their current schemas.

### 7.2 Key Types
- Core transaction and history tables use `BIGINT`
- Lookup tables use `INT`

### 7.3 `IdaraID_FK`
`IdaraID_FK` is a design requirement in most:
- master tables
- transaction tables
- history tables

This is intentional to support filtering, reporting, dashboards, and standard procedure patterns.

### 7.4 Ticket Tree Fields
The `Ticket` table must include:
- `ParentTicketID_FK`
- `RootTicketID_FK`

`RootTicketID_FK` is required for easier tree reporting and cross-level aggregation.

### 7.5 Routing Truth
`TargetDSDID_FK` is the routing truth.

`RoutingLevelCode` is not required as a source-of-truth field in V1.
If later needed, it should be derived or denormalized for display only.

---

## 8. Final V1 Table Set

### 8.1 Master Tables
- `[Tickets].[Service]`
- `[Tickets].[ServiceRoutingRule]`
- `[Tickets].[ServiceSLAPolicy]`
- `[Tickets].[ServiceCatalogSuggestion]`

### 8.2 Transaction Tables
- `[Tickets].[Ticket]`
- `[Tickets].[ArbitrationCase]`
- `[Tickets].[ClarificationRequest]`
- `[Tickets].[QualityReview]`
- `[Tickets].[TicketPauseSession]`
- `[Tickets].[TicketSLA]`

### 8.3 History Tables
- `[Tickets].[TicketHistory]`
- `[Tickets].[TicketSLAHistory]`
- `[Tickets].[CatalogRoutingChangeLog]`

### 8.4 Lookup Tables
- `[Tickets].[TicketStatus]`
- `[Tickets].[TicketClass]`
- `[Tickets].[Priority]`
- `[Tickets].[RequesterType]`
- `[Tickets].[PauseReason]`
- `[Tickets].[ArbitrationReason]`
- `[Tickets].[ClarificationReason]`
- `[Tickets].[QualityReviewResult]`

---

## 9. Final V1 Rules That Remain Inside Stored Procedures
The following should remain implementation rules inside stored procedures, not independent V1 tables:
- assignment eligibility validation through `UserDistributor`
- routing permission checks
- visibility filtering by organizational scope and authorization
- SLA pause/resume calculation
- ticket tree propagation logic
- JSON audit logging to the central audit mechanism

---

## 10. Deferred Features
The following are intentionally postponed from V1:
- auto-escalation configuration and execution
- dynamic service form templates
- section backlog / self-claim mode
- CMDB / asset integration
- dedicated queue tables
- dedicated assignment tables
- dedicated parent-child link tables

---

## 11. Spec-Kit-Oriented Delivery Strategy
This planning file should be executed using a **small-scope, dependency-aware delivery model**.

The recommended implementation style is:
1. define a **single narrow spec**
2. generate a **small task set**
3. implement only that scope
4. validate with SQL tests
5. merge only after the validation passes
6. move to the next spec

This means the project should **not** be executed as one giant database generation task.
Instead, it should be split into tightly controlled implementation slices.

### 11.1 Recommended Execution Rule
Each implementation slice should contain:
- one clear objective
- a limited number of tables or procedures
- explicit dependency boundaries
- acceptance criteria
- rollback awareness

### 11.2 Recommended Spec Size
Each spec should ideally target **one coherent capability**, for example:
- lookup foundations
- service catalogue foundations
- core ticket creation
- assignment flow
- arbitration flow
- clarification flow
- child ticket flow
- SLA flow
- quality closure flow
- reporting flow

### 11.3 Task Granularity Rule
Within each spec:
- database structure tasks must be separated from stored procedure tasks
- stored procedure tasks must be separated from reporting/view tasks
- UI tasks must only appear when the database capability they depend on is already defined

---

## 12. Example Business Scenarios

### Scenario A: Known Service
1. A resident selects **Water Faucet Repair**.
2. The service has a default routing rule to a specific section.
3. The ticket arrives in that section queue.
4. The supervisor assigns it to an eligible technician.
5. The technician completes the work.
6. The ticket is operationally resolved.
7. Quality/monitoring performs final closure if required.

### Scenario B: Unknown Service
1. A requester selects **Other**.
2. The ticket is sent to the appropriate arbitrator.
3. The arbitrator determines the responsible organizational level.
4. The ticket is redirected to the correct department/division/section.
5. If this service repeats in the future, it may be added to the catalogue.

### Scenario C: Parent-Child Dependency
1. A maintenance ticket is opened for an air conditioner.
2. The technician discovers the AC must be replaced.
3. A child ticket is created to request a replacement unit from stores.
4. The parent ticket is paused while waiting for the child ticket outcome.
5. Once the dependent work is completed, the parent ticket resumes.

### Scenario D: Missing Information
1. A ticket is sent to an execution unit.
2. The unit confirms the ticket belongs to them.
3. However, key details are missing.
4. A clarification request is opened.
5. The parent or sending unit supplies the missing details.
6. The ticket resumes execution.

---

## 13. Next Planning Output
The next version of this document should expand into:
1. finalized table-by-table design decisions
2. DDL-ready structure decisions
3. stored procedure map
4. view / DL design map
5. business rule matrix
6. implementation sequence with dependencies
7. testing strategy
8. deployment and rollback considerations

---

## 14. Current Planning Note
This file is intentionally written as a **planning document**, not as a final DDL script.
Its purpose is to lock business and architectural decisions before implementation begins.

---

## 15. Final Table-by-Table Design Decisions

### 15.1 `[Tickets].[Service]`
**Decision:** Include in V1.

**Responsibility:**
- Defines catalogue services visible to requesters.
- Stores service-level defaults and behavioral flags.

**Must include:**
- `serviceID` as `BIGINT`
- `idaraID_FK`
- service code and names
- `ticketClassID_FK`
- `defaultPriorityID_FK`
- location and workflow flags
- standard audit columns

**Do not include:**
- history of routing changes
- current ticket metrics

**Notes:**
- Soft delete only.
- Service lifecycle should be handled via `[Tickets].[ServiceSP]`.

### 15.2 `[Tickets].[ServiceRoutingRule]`
**Decision:** Include in V1.

**Responsibility:**
- Stores the default routing target for each service.
- Supports historical change over time.

**Must include:**
- `serviceRoutingRuleID` as `BIGINT`
- `serviceID_FK`
- `idaraID_FK`
- `targetDSDID_FK` as the true routing target
- optional `queueDistributorID_FK`
- `effectiveFrom`, `effectiveTo`
- approval and change reason fields
- standard audit columns

**Design rule:**
- `targetDSDID_FK` is mandatory.
- `RoutingLevelCode` is not required in V1 as a source-of-truth field.

### 15.3 `[Tickets].[ServiceSLAPolicy]`
**Decision:** Include in V1.

**Responsibility:**
- Defines SLA targets for a service under a given priority.

**Must include:**
- `serviceSLAPolicyID` as `BIGINT`
- `idaraID_FK`
- `serviceID_FK`
- `priorityID_FK`
- four SLA targets:
  - first response
  - assignment
  - operational completion
  - final closure
- date range and active flag
- standard audit columns

### 15.4 `[Tickets].[ServiceCatalogSuggestion]`
**Decision:** Include in V1.

**Responsibility:**
- Captures proposals to convert `Other` requests into formal catalogue services.

**Must include:**
- source ticket reference
- `idaraID_FK`
- proposed service name and description
- optional proposed DSD target and priority
- approval status fields
- optional created service reference
- standard audit columns

### 15.5 `[Tickets].[Ticket]`
**Decision:** Include in V1.

**Responsibility:**
- Stores the current state of the ticket only.
- Acts as the central transactional table.

**Must include:**
- `ticketID` as `BIGINT`
- unique `ticketNo`
- `idaraID_FK`
- `parentTicketID_FK`
- `rootTicketID_FK`
- `serviceID_FK` nullable for `Other`
- `ticketClassID_FK`
- `requesterTypeID_FK`
- `requesterUserID_FK` / `requesterResidentID_FK`
- title and description
- suggested and effective priority
- status
- `currentDSDID_FK`
- `currentQueueDistributorID_FK`
- `assignedUserID_FK`
- location fields
- operational and closure timestamps
- `requiresQualityReview`
- `isOtherService`
- `isParentBlocked`
- standard audit columns

**Design rules:**
- one parent only
- requester type validation must be enforced
- root ticket reference is mandatory for all tree-aware logic
- assignment eligibility is validated in SPs

### 15.6 `[Tickets].[ArbitrationCase]`
**Decision:** Include in V1.

**Responsibility:**
- Tracks wrong-scope disputes.

**Must include:**
- ticket reference
- `idaraID_FK`
- who raised the case
- from which DSD it was raised
- arbitration reason
- current arbitrator distributor
- status, decision type, decision target, and decision metadata
- standard audit columns

### 15.7 `[Tickets].[ClarificationRequest]`
**Decision:** Include in V1.

**Responsibility:**
- Tracks missing information requests separately from disputes.

**Must include:**
- ticket reference
- `idaraID_FK`
- requester of clarification
- requested user or requested DSD target
- clarification reason
- request/response notes and dates
- clarification status
- standard audit columns

### 15.8 `[Tickets].[QualityReview]`
**Decision:** Include in V1.

**Responsibility:**
- Tracks final quality/monitoring verification after operational resolution.

**Must include:**
- ticket reference
- `idaraID_FK`
- reviewer
- review scope
- review result
- notes and review date
- optional return-to user
- finalized flag
- standard audit columns

### 15.9 `[Tickets].[TicketPauseSession]`
**Decision:** Include in V1.

**Responsibility:**
- Tracks pause windows and blocking causes.

**Must include:**
- ticket reference
- `idaraID_FK`
- pause reason
- optional related child ticket
- optional related arbitration case
- optional related clarification request
- pause start and end timestamps
- SLA pause flag
- notes
- standard audit columns

### 15.10 `[Tickets].[TicketSLA]`
**Decision:** Include in V1.

**Responsibility:**
- Stores the current state of each SLA clock per ticket.

**Must include:**
- ticket reference
- `idaraID_FK`
- SLA type code
- target / elapsed / remaining minutes
- breach flag
- start / stop / completion dates
- last calculated date
- standard audit columns

### 15.11 `[Tickets].[TicketHistory]`
**Decision:** Include in V1.

**Responsibility:**
- Immutable operational audit history for ticket actions.

**Must include:**
- ticket reference
- `idaraID_FK`
- action type code
- old/new status
- old/new DSD
- old/new assigned user
- performer
- notes and action date

**Special rule:**
- no soft delete behavior for history records

### 15.12 `[Tickets].[TicketSLAHistory]`
**Decision:** Include in V1.

**Responsibility:**
- Immutable SLA event history.

**Must include:**
- ticket SLA reference
- `idaraID_FK`
- SLA event type
- event date
- notes
- performer if applicable

### 15.13 `[Tickets].[CatalogRoutingChangeLog]`
**Decision:** Include in V1.

**Responsibility:**
- Stores approved routing-rule changes for historical accountability.

**Must include:**
- service reference
- `idaraID_FK`
- old and new routing rule references
- change reason
- optional source arbitration case
- approver
- effective-from and logged date

### 15.14 Lookup Tables
**Decision:** Include in V1.

**Required lookup tables:**
- `TicketStatus`
- `TicketClass`
- `Priority`
- `RequesterType`
- `PauseReason`
- `ArbitrationReason`
- `ClarificationReason`
- `QualityReviewResult`

**Rules:**
- Use `INT` keys.
- Use soft delete only if truly needed.
- Seed through controlled scripts or SPs.

---

## 16. Stored Procedure Map

### 16.1 Core Write Procedures (`SP`)
The system should follow the multiplexer pattern using `@Action` for DML operations.

**Planned procedures:**
- `[Tickets].[ServiceSP]`
- `[Tickets].[TicketSP]`
- `[Tickets].[ArbitrationSP]`
- `[Tickets].[ClarificationSP]`
- `[Tickets].[QualityReviewSP]`
- `[Tickets].[TicketSLASP]` (optional if SLA operations are separated)

### 16.2 Suggested `@Action` Groups

#### `[Tickets].[ServiceSP]`
- `INSERT_SERVICE`
- `UPDATE_SERVICE`
- `DELETE_SERVICE`
- `INSERT_ROUTING_RULE`
- `CLOSE_ROUTING_RULE`
- `UPSERT_SLA_POLICY`
- `APPROVE_SERVICE_SUGGESTION`
- `REJECT_SERVICE_SUGGESTION`

#### `[Tickets].[TicketSP]`
- `INSERT_TICKET`
- `ASSIGN_TICKET`
- `MOVE_TO_IN_PROGRESS`
- `REJECT_TO_SUPERVISOR`
- `CREATE_CHILD_TICKET`
- `PAUSE_TICKET`
- `RESUME_TICKET`
- `RESOLVE_OPERATIONALLY`
- `CLOSE_TICKET`
- `REOPEN_TICKET`

#### `[Tickets].[ArbitrationSP]`
- `OPEN_ARBITRATION_CASE`
- `DECIDE_REDIRECT`
- `DECIDE_OVERRULE`
- `CANCEL_ARBITRATION_CASE`

#### `[Tickets].[ClarificationSP]`
- `OPEN_CLARIFICATION_REQUEST`
- `RESPOND_TO_CLARIFICATION`
- `CLOSE_CLARIFICATION_REQUEST`

#### `[Tickets].[QualityReviewSP]`
- `OPEN_QUALITY_REVIEW`
- `APPROVE_FINAL_CLOSURE`
- `RETURN_FOR_CORRECTION`
- `REJECT_CLOSURE`

### 16.3 Procedure Standards
All SP procedures must:
- follow transaction control (`BEGIN TRAN`, `COMMIT`, `ROLLBACK`)
- use `THROW` for business validation errors
- write JSON audit entries to `dbo.AuditLog`
- update current-state tables and corresponding history tables within the same transaction
- keep security and scope validation inside the procedure body

---

## 17. View / DL Design Map

### 17.1 Core Views
Planned views:
- `[Tickets].[V_TicketFullDetails]`
- `[Tickets].[V_TicketLastAction]`
- `[Tickets].[V_TicketCurrentSLA]`
- `[Tickets].[V_ServiceFullDefinition]`

### 17.2 Optional Queue/Inbox Views
Planned read abstractions:
- `[Tickets].[V_TicketInboxByScope]`
- `[Tickets].[V_TicketArbitrationInbox]`
- `[Tickets].[V_TicketQualityInbox]`

### 17.3 DL Procedures
Planned DL procedures:
- `[Tickets].[TicketDL]`
- `[Tickets].[ServiceDL]`
- `[Tickets].[ArbitrationDL]`
- `[Tickets].[DashboardDL]`

### 17.4 DL Responsibilities

#### `[Tickets].[TicketDL]`
Should cover:
- ticket lists by status
- inbox by scope
- ticket full details
- parent/child tree loading
- assignment history timeline

#### `[Tickets].[ServiceDL]`
Should cover:
- service catalogue listing
- routing rule lookup
- SLA policy lookup
- service suggestion review listing

#### `[Tickets].[ArbitrationDL]`
Should cover:
- open disputes
- dispute history
- routing correction candidates

#### `[Tickets].[DashboardDL]`
Should cover:
- counts by status
- SLA breaches
- arbitration load
- clarification load
- service frequency
- overdue operational tickets
- overdue final closure tickets

---

## 18. Business Rule Matrix

| Rule ID | Rule | Enforcement Location |
|---|---|---|
| BR-01 | A ticket requester can be either resident or internal user, not both at the same time. | `TicketSP` + table constraint |
| BR-02 | `Other` tickets may be opened without `ServiceID_FK`. | `TicketSP` |
| BR-03 | Known services must resolve to one active routing rule for the request date. | `ServiceDL` / `TicketSP` |
| BR-04 | `TargetDSDID_FK` is mandatory for routing rules. | table design + `ServiceSP` |
| BR-05 | Organizational queue receiving is different from execution assignment. | `TicketSP` |
| BR-06 | Direct user assignment is only valid if the user is eligible through `UserDistributor`. | `TicketSP` |
| BR-07 | An execution user may reject to supervisor, not directly to arbitration. | `TicketSP` |
| BR-08 | Scope disputes must create `ArbitrationCase`. | `ArbitrationSP` |
| BR-09 | Missing information must create `ClarificationRequest`, not `ArbitrationCase`. | `ClarificationSP` |
| BR-10 | A child ticket belongs to one parent only. | `Ticket` design + `TicketSP` |
| BR-11 | Parent tickets may be paused due to a blocking child ticket. | `TicketSP` + `TicketPauseSession` |
| BR-12 | Parent tickets must not be finally closed while blocking child tickets remain open. | `TicketSP` |
| BR-13 | Operational closure is separate from final closure. | `TicketSP` + `QualityReviewSP` |
| BR-14 | Quality review cannot start before operational resolution. | `QualityReviewSP` |
| BR-15 | SLA clocks must pause during valid blocking windows. | `TicketSP` / `TicketSLASP` |
| BR-16 | Every meaningful state change must be written to `TicketHistory`. | all write SPs |
| BR-17 | Every DML action must write JSON audit to `dbo.AuditLog`. | all write SPs |
| BR-18 | Routing-rule changes must be historically preserved, not overwritten silently. | `ServiceSP` |
| BR-19 | Service suggestions can only be approved by the handling arbitrator or a higher authority. | `ServiceSP` |
| BR-20 | End beneficiaries are not expected to provide detailed technical specs. | `ClarificationSP` / business flow |

---

## 19. Spec-Kit Implementation Sequence with Dependencies

The implementation should be decomposed into **small Spec-Kit friendly specs**.
Each spec below is intentionally narrow enough to become:
- one spec document
- one task list
- one implementation branch or short sequence of commits
- one validation checkpoint

### Spec 01: Lookup Foundations
**Goal:** establish static reference data required by the domain.

**Database tasks:**
- create `[Tickets].[TicketStatus]`
- create `[Tickets].[TicketClass]`
- create `[Tickets].[Priority]`
- create `[Tickets].[RequesterType]`
- create `[Tickets].[PauseReason]`
- create `[Tickets].[ArbitrationReason]`
- create `[Tickets].[ClarificationReason]`
- create `[Tickets].[QualityReviewResult]`
- prepare seed scripts for all lookup values
- validate unique codes and basic constraints

**UI tasks (only if UI work starts this early):**
- none required yet

**Acceptance criteria:**
- all lookup tables exist
- all seed values are inserted successfully
- lookup codes are unique
- lookup records can be read through basic select tests

### Spec 02: Service Catalogue Foundations
**Goal:** establish the catalogue and routing base.

**Database tasks:**
- create `[Tickets].[Service]`
- create `[Tickets].[ServiceRoutingRule]`
- create `[Tickets].[ServiceSLAPolicy]`
- create `[Tickets].[ServiceCatalogSuggestion]`
- add constraints for active routing rule date ranges
- create `[Tickets].[ServiceSP]`
- implement actions for service insert/update/deactivate
- implement actions for routing rule insert/close
- implement actions for SLA policy upsert
- create `[Tickets].[V_ServiceFullDefinition]`
- create `[Tickets].[ServiceDL]`

**UI tasks:**
- create admin screen for service catalogue list
- create admin screen for service create/edit
- create admin screen for routing rule maintenance
- create admin screen for SLA policy maintenance
- create admin screen for service suggestion review

**Acceptance criteria:**
- services can be created and updated only through `ServiceSP`
- routing rules can be added with valid `TargetDSDID_FK`
- historical routing rule replacement works correctly
- SLA policies are retrievable per service and priority

### Spec 03: Core Ticket Backbone
**Goal:** enable ticket creation and current-state storage.

**Database tasks:**
- create `[Tickets].[Ticket]`
- create `[Tickets].[TicketHistory]`
- implement `INSERT_TICKET` in `[Tickets].[TicketSP]`
- implement requester-type validation
- implement root-ticket initialization logic
- create `[Tickets].[V_TicketFullDetails]`
- create `[Tickets].[V_TicketLastAction]`
- create `[Tickets].[TicketDL]` for ticket details and basic lists

**UI tasks:**
- create requester ticket creation screen
- create ticket details screen
- create basic ticket list screen
- show current status, priority, and organizational queue data

**Acceptance criteria:**
- tickets can be created for resident or internal user
- `Other` tickets can be created without `ServiceID_FK`
- `TicketHistory` receives creation events
- `rootTicketID_FK` is set correctly

### Spec 04: Assignment and Work Start
**Goal:** support organizational queue handling and direct execution assignment.

**Database tasks:**
- implement `ASSIGN_TICKET` in `[Tickets].[TicketSP]`
- implement `MOVE_TO_IN_PROGRESS` in `[Tickets].[TicketSP]`
- implement assignment eligibility validation through `UserDistributor`
- implement supervisor rejection action `REJECT_TO_SUPERVISOR`
- write corresponding ticket history entries
- extend `TicketDL` for inbox-style reads by current queue and assignee

**UI tasks:**
- create queue inbox screen by scope
- create assignment action UI
- create start-work action UI
- create rejection-to-supervisor action UI

**Acceptance criteria:**
- eligible users can be assigned only within allowed scope
- ticket status changes are logged to history
- inbox queries return correct tickets by scope

### Spec 05: Clarification Flow
**Goal:** support missing-information handling without mixing it with scope disputes.

**Database tasks:**
- create `[Tickets].[ClarificationRequest]`
- implement `[Tickets].[ClarificationSP]`
- implement `OPEN_CLARIFICATION_REQUEST`
- implement `RESPOND_TO_CLARIFICATION`
- implement `CLOSE_CLARIFICATION_REQUEST`
- create clarification-related ticket history entries
- support pause session creation when clarification blocks execution

**UI tasks:**
- create clarification request form
- create clarification response form
- show open clarification requests in ticket details

**Acceptance criteria:**
- clarification can be opened without using arbitration
- clarification response updates ticket flow correctly
- blocking clarification opens a valid pause session

### Spec 06: Arbitration Flow
**Goal:** support wrong-scope disputes and controlled redirection.

**Database tasks:**
- create `[Tickets].[ArbitrationCase]`
- implement `[Tickets].[ArbitrationSP]`
- implement `OPEN_ARBITRATION_CASE`
- implement `DECIDE_REDIRECT`
- implement `DECIDE_OVERRULE`
- implement `CANCEL_ARBITRATION_CASE`
- create `[Tickets].[ArbitrationDL]`
- create arbitration-related history entries

**UI tasks:**
- create arbitration inbox screen
- create arbitration decision screen
- show arbitration status inside ticket details

**Acceptance criteria:**
- disputes can be opened only through allowed supervisory flow
- arbitration decisions correctly update target queue and history
- arbitration load can be listed by organizational level

### Spec 07: Parent-Child Ticketing
**Goal:** support dependent child work.

**Database tasks:**
- implement `CREATE_CHILD_TICKET` in `[Tickets].[TicketSP]`
- implement root and parent inheritance rules
- implement validation that each child has one parent only
- extend `TicketDL` to load parent/child tree
- write ticket history for child creation

**UI tasks:**
- create child ticket creation action in ticket details
- create parent/child relationship display
- create tree visualization or related-ticket section

**Acceptance criteria:**
- child tickets inherit the correct root ticket
- parent-child tree loads correctly
- child creation is blocked when approval requirements are not satisfied

### Spec 08: Blocking and Pause Sessions
**Goal:** support controlled pause windows and dependency-based blocking.

**Database tasks:**
- create `[Tickets].[TicketPauseSession]`
- implement `PAUSE_TICKET` in `[Tickets].[TicketSP]`
- implement `RESUME_TICKET` in `[Tickets].[TicketSP]`
- implement pause validation rules
- implement parent blocking due to child tickets
- write pause/resume history entries

**UI tasks:**
- create pause action UI
- create resume action UI
- display active blocking reason on ticket details
- display parent-blocked state clearly

**Acceptance criteria:**
- pause sessions store start and end correctly
- ticket pause reason is visible
- parent tickets resume correctly after valid unblocking

### Spec 09: SLA Engine
**Goal:** support SLA initialization, pause, resume, and breach tracking.

**Database tasks:**
- create `[Tickets].[TicketSLA]`
- create `[Tickets].[TicketSLAHistory]`
- implement SLA initialization from service + priority
- implement pause/resume recalculation behavior
- implement breach detection logic
- create `[Tickets].[V_TicketCurrentSLA]`
- optionally create `[Tickets].[TicketSLASP]`

**UI tasks:**
- show SLA badges / timers in ticket details
- show SLA state in queue lists
- show breach markers in dashboards and ticket lists

**Acceptance criteria:**
- SLA clocks initialize correctly
- valid blocking pauses stop SLA progress
- resuming work continues calculation correctly
- breach state is stored and visible

### Spec 10: Quality Review and Final Closure
**Goal:** support two-stage closure.

**Database tasks:**
- create `[Tickets].[QualityReview]`
- implement `[Tickets].[QualityReviewSP]`
- implement `OPEN_QUALITY_REVIEW`
- implement `APPROVE_FINAL_CLOSURE`
- implement `RETURN_FOR_CORRECTION`
- implement `REJECT_CLOSURE`
- enforce rule that final closure cannot happen before operational resolution

**UI tasks:**
- create quality review inbox
- create final closure approval UI
- create return-for-correction UI
- display operational vs final closure state clearly

**Acceptance criteria:**
- final closure is blocked before operational resolution
- quality review decisions update the ticket correctly
- returned tickets re-enter the appropriate working state

### Spec 11: Catalogue Learning and Routing Correction
**Goal:** allow repeated real-world cases to improve the catalogue.

**Database tasks:**
- create `[Tickets].[CatalogRoutingChangeLog]`
- implement service suggestion approval/rejection actions in `[Tickets].[ServiceSP]`
- implement routing rule replacement with effective dating
- log approved routing corrections

**UI tasks:**
- create service suggestion review UI
- create routing correction approval UI
- show historical routing changes for a service

**Acceptance criteria:**
- approved suggestions can create real services
- routing corrections do not overwrite historical accountability
- routing change history is queryable

### Spec 12: Reporting and Dashboards
**Goal:** provide read models for operational and leadership visibility.

**Database tasks:**
- create `[Tickets].[V_TicketInboxByScope]`
- create `[Tickets].[V_TicketArbitrationInbox]`
- create `[Tickets].[V_TicketQualityInbox]`
- create `[Tickets].[DashboardDL]`
- support counts by status, overdue lists, SLA breaches, arbitration load, and service frequency

**UI tasks:**
- create dashboard for organizational leadership
- create dashboard widgets for quality and monitoring
- create overdue tickets report screen
- create service frequency / workload report screen

**Acceptance criteria:**
- dashboard queries return correct scoped counts
- overdue and breach reports match expected test data
- leadership views can be filtered by organizational level

---

## 20. Testing Strategy

### 20.1 Structural Testing
- verify PK/FK integrity
- verify unique constraints
- verify active-rule uniqueness behavior
- verify requester-type mutual exclusivity
- verify parent/root ticket consistency

### 20.2 Stored Procedure Testing
Each SP action must be tested for:
- success path
- invalid input
- unauthorized scope
- transaction rollback behavior
- audit insertion behavior
- history insertion behavior

### 20.3 Business Scenario Testing
At minimum, test the following:
1. known service direct routing
2. `Other` service arbitration routing
3. division-level direct assignment
4. section-level routing then section assignment
5. wrong-scope dispute and redirect
6. missing-information clarification and resume
7. child ticket creation with parent blocking
8. parent resume after child completion
9. operational closure followed by quality approval
10. quality rejection returning work for correction

### 20.4 SLA Testing
Test:
- SLA initialization from service + priority
- pause on arbitration
- pause on clarification
- pause on dependency
- resume on dependency removal
- breach marking
- final closure SLA completion

### 20.5 Reporting Testing
Test:
- inbox visibility by scope
- dashboard counts by status
- service frequency reporting
- overdue lists
- audit trace completeness

### 20.6 Spec-Level Acceptance Rule
No spec should be considered complete unless:
- structure deployment succeeds
- related SP actions pass validation tests
- history logging is verified
- audit logging is verified
- required read models return expected data

---

## 21. Deployment and Rollback Considerations

### 21.1 Deployment Order
1. create schema if missing
2. create lookup tables
3. seed lookup values
4. create master tables
5. create transaction tables
6. create history tables
7. create views
8. create DL procedures
9. create SP procedures
10. run smoke tests

### 21.2 Rollback Strategy
- use idempotent deployment scripts where possible
- deploy schema in dependency order
- separate structure deployment from seed deployment
- wrap high-risk migration steps in explicit transaction boundaries where safe
- preserve lookup seeds and history tables unless rollback requires full environment reset

### 21.3 Initial Data Requirements
Before functional testing, ensure availability of:
- at least one valid `Idara`
- valid `DeptSecDiv / DSDID` nodes
- valid `Distributor` records for queue/arbitration roles
- valid `UserDistributor` mappings
- at least one resident and one internal user
- seeded priorities, statuses, classes, and reasons

---

## 22. OpenCode and Spec-Kit Execution Notes
This planning file is intended to be used as a stepwise execution guide.

When using it with OpenCode and Spec-Kit:
1. convert **one spec at a time** into a dedicated implementation brief
2. generate tasks only for that spec, not for the whole project at once
3. keep database tasks separate from UI tasks inside the same spec
4. do not start UI implementation for a capability before its database contract exists
5. preserve project naming patterns strictly
6. preserve audit columns and audit logging in every write procedure
7. validate each spec with executable SQL tests before moving to the next
8. keep business rules inside SPs, not scattered into ad-hoc scripts
9. prefer short-lived implementation branches or tightly scoped commits
10. update the GitHub Project board immediately when a spec or task state changes

### 22.1 Recommended Spec-Kit Workflow
For each spec:
- define scope
- confirm dependencies
- generate tasks
- implement database structure
- implement write procedures
- implement read models
- implement UI only when the contract is stable
- run tests
- update board status
- merge

### 22.2 Recommended Task Naming Style
Use action-oriented task titles, for example:
- `Create TicketStatus lookup table`
- `Implement INSERT_TICKET action in TicketSP`
- `Add ticket creation screen for requester flow`
- `Add arbitration inbox view and DL support`

---

## 23. GitHub Projects Task Board Plan
This section replaces any Trello-oriented planning.

The project board should track both **database** and **UI** tasks, but UI tasks must appear only when their underlying database capability is ready or already planned in the same spec.

### 23.1 Suggested GitHub Project Columns
- Backlog
- Ready
- In Progress
- Review
- Blocked
- Done

### 23.2 Task Grouping Rule
Each task should belong to one of these categories:
- `DB-Structure`
- `DB-SP`
- `DB-DL-View`
- `UI`
- `Test`
- `Docs`

### 23.3 Suggested GitHub Project Items by Spec

#### Spec 01: Lookup Foundations
- `Create TicketStatus lookup table`
- `Create TicketClass lookup table`
- `Create Priority lookup table`
- `Create RequesterType lookup table`
- `Create PauseReason lookup table`
- `Create ArbitrationReason lookup table`
- `Create ClarificationReason lookup table`
- `Create QualityReviewResult lookup table`
- `Seed all lookup values`
- `Test lookup uniqueness and seed integrity`

#### Spec 02: Service Catalogue Foundations
- `Create Service table`
- `Create ServiceRoutingRule table`
- `Create ServiceSLAPolicy table`
- `Create ServiceCatalogSuggestion table`
- `Implement ServiceSP service actions`
- `Implement ServiceSP routing rule actions`
- `Implement ServiceSP SLA policy actions`
- `Create V_ServiceFullDefinition view`
- `Create ServiceDL procedure`
- `Test service catalogue foundation flows`
- `Build service catalogue admin list screen`
- `Build service editor screen`
- `Build routing rule maintenance screen`
- `Build SLA policy maintenance screen`
- `Build service suggestion review screen`

#### Spec 03: Core Ticket Backbone
- `Create Ticket table`
- `Create TicketHistory table`
- `Implement INSERT_TICKET in TicketSP`
- `Implement requester type validation`
- `Implement root ticket initialization`
- `Create V_TicketFullDetails view`
- `Create V_TicketLastAction view`
- `Create TicketDL basic detail/list actions`
- `Test core ticket creation flow`
- `Build ticket creation screen`
- `Build ticket details screen`
- `Build basic ticket list screen`

#### Spec 04: Assignment and Work Start
- `Implement ASSIGN_TICKET in TicketSP`
- `Implement MOVE_TO_IN_PROGRESS in TicketSP`
- `Implement REJECT_TO_SUPERVISOR in TicketSP`
- `Implement assignment eligibility validation`
- `Extend TicketDL for queue inbox reads`
- `Test assignment and work start flow`
- `Build scope-based inbox screen`
- `Build assignment action UI`
- `Build start work action UI`
- `Build reject-to-supervisor action UI`

#### Spec 05: Clarification Flow
- `Create ClarificationRequest table`
- `Implement ClarificationSP actions`
- `Link clarification flow to ticket history`
- `Link blocking clarification to pause sessions`
- `Test clarification flow`
- `Build clarification request UI`
- `Build clarification response UI`
- `Show clarification state in ticket details`

#### Spec 06: Arbitration Flow
- `Create ArbitrationCase table`
- `Implement ArbitrationSP actions`
- `Create ArbitrationDL procedure`
- `Link arbitration flow to ticket history`
- `Test arbitration flow`
- `Build arbitration inbox screen`
- `Build arbitration decision screen`
- `Show arbitration status in ticket details`

#### Spec 07: Parent-Child Ticketing
- `Implement CREATE_CHILD_TICKET in TicketSP`
- `Implement parent/root inheritance logic`
- `Extend TicketDL for ticket tree loading`
- `Test parent-child ticket flow`
- `Build child ticket creation UI`
- `Build parent-child tree UI`

#### Spec 08: Blocking and Pause Sessions
- `Create TicketPauseSession table`
- `Implement PAUSE_TICKET in TicketSP`
- `Implement RESUME_TICKET in TicketSP`
- `Implement parent blocking rules`
- `Test pause and resume flow`
- `Build pause action UI`
- `Build resume action UI`
- `Display active blocking reason in ticket details`

#### Spec 09: SLA Engine
- `Create TicketSLA table`
- `Create TicketSLAHistory table`
- `Implement SLA initialization logic`
- `Implement SLA pause and resume logic`
- `Implement SLA breach detection`
- `Create V_TicketCurrentSLA view`
- `Test SLA engine`
- `Show SLA timers and breach markers in ticket UI`
- `Show SLA state in queue lists`

#### Spec 10: Quality Review and Final Closure
- `Create QualityReview table`
- `Implement QualityReviewSP actions`
- `Implement final closure validation`
- `Test quality review and final closure flow`
- `Build quality review inbox screen`
- `Build final closure approval UI`
- `Build return-for-correction UI`

#### Spec 11: Catalogue Learning and Routing Correction
- `Create CatalogRoutingChangeLog table`
- `Implement service suggestion approval flow`
- `Implement routing rule replacement with effective dating`
- `Log routing corrections historically`
- `Test catalogue learning and routing correction`
- `Build service suggestion approval UI`
- `Build routing correction review UI`
- `Show routing change history for a service`

#### Spec 12: Reporting and Dashboards
- `Create V_TicketInboxByScope view`
- `Create V_TicketArbitrationInbox view`
- `Create V_TicketQualityInbox view`
- `Create DashboardDL procedure`
- `Test dashboard and reporting queries`
- `Build organizational leadership dashboard`
- `Build quality and monitoring dashboard widgets`
- `Build overdue tickets report screen`
- `Build service frequency report screen`

### 23.4 Board Usage Rule
- Move tasks to `Ready` only after dependencies are satisfied.
- Move UI tasks to `Ready` only when their supporting database structure and procedure contract already exist or are locked in the same spec.
- Link each UI task to the database tasks it depends on.
- Prefer one spec milestone at a time rather than mixing unrelated tasks across many specs.

