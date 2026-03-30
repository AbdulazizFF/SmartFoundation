# Ticketing System Implementation Checklist

Actionable checklist derived from `system-analysis.md` section 16 and `plan.md` specs 01-12.

Each spec is a checkpoint. Do not start the next spec until the current one passes its acceptance criteria.

---

## Legend

- `[ ]` = not started
- `[~]` = in progress
- `[x]` = done
- Category tags: `DB-Structure` `DB-SP` `DB-DL-View` `UI` `Test`

---

## Spec 01: Lookup Foundations

**Goal:** Establish static reference data required by the domain.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[TicketStatus]` lookup table
- [ ] `DB-Structure` Create `[Tickets].[TicketClass]` lookup table
- [ ] `DB-Structure` Create `[Tickets].[Priority]` lookup table
- [ ] `DB-Structure` Create `[Tickets].[RequesterType]` lookup table
- [ ] `DB-Structure` Create `[Tickets].[PauseReason]` lookup table
- [ ] `DB-Structure` Create `[Tickets].[ArbitrationReason]` lookup table
- [ ] `DB-Structure` Create `[Tickets].[ClarificationReason]` lookup table
- [ ] `DB-Structure` Create `[Tickets].[QualityReviewResult]` lookup table

### Seed Data

- [ ] `DB-SP` Prepare seed scripts for all lookup values
- [ ] `DB-SP` Validate unique codes and basic constraints
- [ ] `Test` Run basic select tests against all lookup tables

### Acceptance Criteria

- [ ] All lookup tables exist with correct schema
- [ ] All seed values inserted successfully
- [ ] Lookup codes are unique within each table
- [ ] Basic select queries return expected rows

---

## Spec 02: Service Catalogue Foundations

**Goal:** Establish the catalogue, routing, and SLA base.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[Service]`
- [ ] `DB-Structure` Create `[Tickets].[ServiceRoutingRule]`
- [ ] `DB-Structure` Create `[Tickets].[ServiceSLAPolicy]`
- [ ] `DB-Structure` Create `[Tickets].[ServiceCatalogSuggestion]`
- [ ] `DB-Structure` Add constraints for active routing rule date ranges

### Write Procedures

- [ ] `DB-SP` Create `[Tickets].[ServiceSP]`
- [ ] `DB-SP` Implement `INSERT_SERVICE` action
- [ ] `DB-SP` Implement `UPDATE_SERVICE` action
- [ ] `DB-SP` Implement `DELETE_SERVICE` action (soft delete)
- [ ] `DB-SP` Implement `INSERT_ROUTING_RULE` action
- [ ] `DB-SP` Implement `CLOSE_ROUTING_RULE` action
- [ ] `DB-SP` Implement `UPSERT_SLA_POLICY` action
- [ ] `DB-SP` Implement `APPROVE_SERVICE_SUGGESTION` action
- [ ] `DB-SP` Implement `REJECT_SERVICE_SUGGESTION` action

### Read Procedures

- [ ] `DB-DL-View` Create `[Tickets].[V_ServiceFullDefinition]`
- [ ] `DB-DL-View` Create `[Tickets].[ServiceDL]`

### Application Layer

- [ ] Add `ServiceSP` and `ServiceDL` mappings to `ProcedureMapper` if using gateway routing
- [ ] OR add dedicated `TicketingService` in `SmartFoundation.Application` if bypassing gateway

### UI

- [ ] `UI` Create admin screen for service catalogue list
- [ ] `UI` Create admin screen for service create/edit
- [ ] `UI` Create admin screen for routing rule maintenance
- [ ] `UI` Create admin screen for SLA policy maintenance
- [ ] `UI` Create admin screen for service suggestion review

### Tests

- [ ] `Test` Service insert/update/deactivate through `ServiceSP`
- [ ] `Test` Routing rule insert with valid `TargetDSDID_FK`
- [ ] `Test` Historical routing rule replacement
- [ ] `Test` SLA policy retrieval per service and priority

### Acceptance Criteria

- [ ] Services created and updated only through `ServiceSP`
- [ ] Routing rules added with valid `TargetDSDID_FK`
- [ ] Historical routing rule replacement works correctly
- [ ] SLA policies retrievable per service and priority

---

## Spec 03: Core Ticket Backbone

**Goal:** Enable ticket creation and current-state storage.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[Ticket]`
- [ ] `DB-Structure` Create `[Tickets].[TicketHistory]`

### Write Procedures

- [ ] `DB-SP` Create `[Tickets].[TicketSP]`
- [ ] `DB-SP` Implement `INSERT_TICKET` action
- [ ] `DB-SP` Implement requester-type validation (resident vs internal, mutual exclusivity)
- [ ] `DB-SP` Implement root-ticket initialization logic
- [ ] `DB-SP` Implement audit logging for ticket creation

### Read Procedures

- [ ] `DB-DL-View` Create `[Tickets].[V_TicketFullDetails]`
- [ ] `DB-DL-View` Create `[Tickets].[V_TicketLastAction]`
- [ ] `DB-DL-View` Create `[Tickets].[TicketDL]` (details and basic lists)

### Application Layer

- [ ] Add ticket-related procedure mappings to `ProcedureMapper`
- [ ] Add ticket data-load methods to `MastersServies` or a new `TicketingService`

### UI

- [ ] `UI` Create requester ticket creation screen
- [ ] `UI` Create ticket details screen
- [ ] `UI` Create basic ticket list screen
- [ ] `UI` Show current status, priority, and organizational queue data

### Tests

- [ ] `Test` Ticket creation for resident requester
- [ ] `Test` Ticket creation for internal user requester
- [ ] `Test` `Other` ticket creation without `ServiceID_FK`
- [ ] `Test` `TicketHistory` receives creation events
- [ ] `Test` `rootTicketID_FK` set correctly

### Acceptance Criteria

- [ ] Tickets created for both resident and internal user
- [ ] `Other` tickets work without `ServiceID_FK`
- [ ] `TicketHistory` receives creation events
- [ ] `rootTicketID_FK` initialized correctly

---

## Spec 04: Assignment and Work Start

**Goal:** Support organizational queue handling and direct execution assignment.

### Write Procedures

- [ ] `DB-SP` Implement `ASSIGN_TICKET` in `[Tickets].[TicketSP]`
- [ ] `DB-SP` Implement `MOVE_TO_IN_PROGRESS` in `[Tickets].[TicketSP]`
- [ ] `DB-SP` Implement `REJECT_TO_SUPERVISOR` in `[Tickets].[TicketSP]`
- [ ] `DB-SP` Implement assignment eligibility validation through `UserDistributor`
- [ ] `DB-SP` Write corresponding ticket history entries for each action

### Read Procedures

- [ ] `DB-DL-View` Extend `TicketDL` for inbox-style reads by current queue and assignee

### UI

- [ ] `UI` Create queue inbox screen by organizational scope
- [ ] `UI` Create assignment action UI
- [ ] `UI` Create start-work action UI
- [ ] `UI` Create rejection-to-supervisor action UI

### Tests

- [ ] `Test` Assignment within allowed scope only
- [ ] `Test` Assignment outside scope rejected
- [ ] `Test` Ticket status changes logged to history
- [ ] `Test` Inbox queries return correct tickets by scope

### Acceptance Criteria

- [ ] Eligible users assigned only within allowed scope
- [ ] Ticket status changes logged to history
- [ ] Inbox queries return correct tickets by scope

---

## Spec 05: Clarification Flow

**Goal:** Support missing-information handling without mixing it with scope disputes.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[ClarificationRequest]`

### Write Procedures

- [ ] `DB-SP` Create `[Tickets].[ClarificationSP]`
- [ ] `DB-SP` Implement `OPEN_CLARIFICATION_REQUEST`
- [ ] `DB-SP` Implement `RESPOND_TO_CLARIFICATION`
- [ ] `DB-SP` Implement `CLOSE_CLARIFICATION_REQUEST`
- [ ] `DB-SP` Create clarification-related ticket history entries
- [ ] `DB-SP` Support pause session creation when clarification blocks execution

### UI

- [ ] `UI` Create clarification request form
- [ ] `UI` Create clarification response form
- [ ] `UI` Show open clarification requests in ticket details

### Tests

- [ ] `Test` Clarification opened without using arbitration
- [ ] `Test` Clarification response updates ticket flow
- [ ] `Test` Blocking clarification opens a valid pause session

### Acceptance Criteria

- [ ] Clarification separate from arbitration
- [ ] Response updates ticket flow correctly
- [ ] Blocking clarification opens valid pause session

---

## Spec 06: Arbitration Flow

**Goal:** Support wrong-scope disputes and controlled redirection.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[ArbitrationCase]`

### Write Procedures

- [ ] `DB-SP` Create `[Tickets].[ArbitrationSP]`
- [ ] `DB-SP` Implement `OPEN_ARBITRATION_CASE`
- [ ] `DB-SP` Implement `DECIDE_REDIRECT`
- [ ] `DB-SP` Implement `DECIDE_OVERRULE`
- [ ] `DB-SP` Implement `CANCEL_ARBITRATION_CASE`
- [ ] `DB-SP` Create arbitration-related history entries

### Read Procedures

- [ ] `DB-DL-View` Create `[Tickets].[ArbitrationDL]`

### UI

- [ ] `UI` Create arbitration inbox screen
- [ ] `UI` Create arbitration decision screen
- [ ] `UI` Show arbitration status inside ticket details

### Tests

- [ ] `Test` Disputes opened only through allowed supervisory flow
- [ ] `Test` Arbitration decisions update target queue and history
- [ ] `Test` Arbitration load listed by organizational level

### Acceptance Criteria

- [ ] Disputes opened only through supervisory flow
- [ ] Decisions correctly update target queue and history
- [ ] Arbitration load queryable by organizational level

---

## Spec 07: Parent-Child Ticketing

**Goal:** Support dependent child work.

### Write Procedures

- [ ] `DB-SP` Implement `CREATE_CHILD_TICKET` in `[Tickets].[TicketSP]`
- [ ] `DB-SP` Implement root and parent inheritance rules
- [ ] `DB-SP` Validate each child has one parent only
- [ ] `DB-SP` Write ticket history for child creation

### Read Procedures

- [ ] `DB-DL-View` Extend `TicketDL` to load parent/child tree

### UI

- [ ] `UI` Create child ticket creation action in ticket details
- [ ] `UI` Create parent/child relationship display
- [ ] `UI` Create tree visualization or related-ticket section

### Tests

- [ ] `Test` Child tickets inherit correct root ticket
- [ ] `Test` Parent-child tree loads correctly
- [ ] `Test` Child creation blocked when approval requirements not satisfied

### Acceptance Criteria

- [ ] Child tickets inherit correct root
- [ ] Parent-child tree loads correctly
- [ ] Approval requirements enforced

---

## Spec 08: Blocking and Pause Sessions

**Goal:** Support controlled pause windows and dependency-based blocking.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[TicketPauseSession]`

### Write Procedures

- [ ] `DB-SP` Implement `PAUSE_TICKET` in `[Tickets].[TicketSP]`
- [ ] `DB-SP` Implement `RESUME_TICKET` in `[Tickets].[TicketSP]`
- [ ] `DB-SP` Implement pause validation rules
- [ ] `DB-SP` Implement parent blocking due to child tickets
- [ ] `DB-SP` Write pause/resume history entries

### UI

- [ ] `UI` Create pause action UI
- [ ] `UI` Create resume action UI
- [ ] `UI` Display active blocking reason on ticket details
- [ ] `UI` Display parent-blocked state clearly

### Tests

- [ ] `Test` Pause sessions store start and end correctly
- [ ] `Test` Ticket pause reason visible
- [ ] `Test` Parent tickets resume correctly after valid unblocking

### Acceptance Criteria

- [ ] Pause sessions store timestamps correctly
- [ ] Pause reason visible
- [ ] Parent resume works after unblocking

---

## Spec 09: SLA Engine

**Goal:** Support SLA initialization, pause, resume, and breach tracking.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[TicketSLA]`
- [ ] `DB-Structure` Create `[Tickets].[TicketSLAHistory]`

### Logic

- [ ] `DB-SP` Implement SLA initialization from service + priority
- [ ] `DB-SP` Implement pause/resume recalculation behavior
- [ ] `DB-SP` Implement breach detection logic
- [ ] `DB-SP` Optionally create `[Tickets].[TicketSLASP]`

### Read Procedures

- [ ] `DB-DL-View` Create `[Tickets].[V_TicketCurrentSLA]`

### UI

- [ ] `UI` Show SLA badges/timers in ticket details
- [ ] `UI` Show SLA state in queue lists
- [ ] `UI` Show breach markers in dashboards and ticket lists

### Tests

- [ ] `Test` SLA clocks initialize correctly
- [ ] `Test` Valid blocking pauses stop SLA progress
- [ ] `Test` Resuming work continues calculation correctly
- [ ] `Test` Breach state stored and visible

### Acceptance Criteria

- [ ] SLA clocks initialize correctly
- [ ] Pauses stop SLA progress
- [ ] Resume continues calculation
- [ ] Breach state stored and visible

---

## Spec 10: Quality Review and Final Closure

**Goal:** Support two-stage closure.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[QualityReview]`

### Write Procedures

- [ ] `DB-SP` Create `[Tickets].[QualityReviewSP]`
- [ ] `DB-SP` Implement `OPEN_QUALITY_REVIEW`
- [ ] `DB-SP` Implement `APPROVE_FINAL_CLOSURE`
- [ ] `DB-SP` Implement `RETURN_FOR_CORRECTION`
- [ ] `DB-SP` Implement `REJECT_CLOSURE`
- [ ] `DB-SP` Enforce final closure blocked before operational resolution

### UI

- [ ] `UI` Create quality review inbox
- [ ] `UI` Create final closure approval UI
- [ ] `UI` Create return-for-correction UI
- [ ] `UI` Display operational vs final closure state clearly

### Tests

- [ ] `Test` Final closure blocked before operational resolution
- [ ] `Test` Quality review decisions update ticket correctly
- [ ] `Test` Returned tickets re-enter appropriate working state

### Acceptance Criteria

- [ ] Final closure blocked before operational resolution
- [ ] Quality review decisions update ticket
- [ ] Returned tickets re-enter working state

---

## Spec 11: Catalogue Learning and Routing Correction

**Goal:** Allow repeated real-world cases to improve the catalogue.

### Database Structure

- [ ] `DB-Structure` Create `[Tickets].[CatalogRoutingChangeLog]`

### Write Procedures

- [ ] `DB-SP` Implement service suggestion approval/rejection in `[Tickets].[ServiceSP]`
- [ ] `DB-SP` Implement routing rule replacement with effective dating
- [ ] `DB-SP` Log approved routing corrections

### UI

- [ ] `UI` Create service suggestion review UI
- [ ] `UI` Create routing correction approval UI
- [ ] `UI` Show historical routing changes for a service

### Tests

- [ ] `Test` Approved suggestions create real services
- [ ] `Test` Routing corrections preserve historical accountability
- [ ] `Test` Routing change history queryable

### Acceptance Criteria

- [ ] Approved suggestions create real services
- [ ] Routing corrections preserve history
- [ ] Routing change history queryable

---

## Spec 12: Reporting and Dashboards

**Goal:** Provide read models for operational and leadership visibility.

### Read Procedures

- [ ] `DB-DL-View` Create `[Tickets].[V_TicketInboxByScope]`
- [ ] `DB-DL-View` Create `[Tickets].[V_TicketArbitrationInbox]`
- [ ] `DB-DL-View` Create `[Tickets].[V_TicketQualityInbox]`
- [ ] `DB-DL-View` Create `[Tickets].[DashboardDL]`

### UI

- [ ] `UI` Create dashboard for organizational leadership
- [ ] `UI` Create dashboard widgets for quality and monitoring
- [ ] `UI` Create overdue tickets report screen
- [ ] `UI` Create service frequency/workload report screen

### Tests

- [ ] `Test` Dashboard queries return correct scoped counts
- [ ] `Test` Overdue and breach reports match expected test data
- [ ] `Test` Leadership views filterable by organizational level

### Acceptance Criteria

- [ ] Dashboard queries return correct scoped counts
- [ ] Overdue/breach reports accurate
- [ ] Leadership views filterable by organizational level

---

## Cross-Spec Requirements

These apply across all specs and should be validated at every checkpoint.

### Audit and History

- [ ] Every DML action writes JSON audit to `dbo.AuditLog`
- [ ] Every meaningful state change written to `TicketHistory`
- [ ] History records are immutable (no soft delete)

### Security and Scope

- [ ] Routing permission checks inside SPs
- [ ] Visibility filtering by organizational scope
- [ ] Assignment eligibility validation through `UserDistributor`

### Conventions

- [ ] All new tables use `BIGINT` for core IDs, `INT` for lookups
- [ ] `idaraID_FK` included where required by convention
- [ ] Soft delete only where explicitly designed
- [ ] `SET NOCOUNT ON` and `SET XACT_ABORT ON` in all write procedures
- [ ] `BEGIN TRY / BEGIN CATCH` in all write procedures
- [ ] Business errors use `THROW 50001`, unexpected errors use `THROW 50002`
- [ ] Success branches end with `SELECT 1 AS IsSuccessful, N'...' AS Message_`

### Pattern Compliance

- [ ] New pages follow Housing-style controller pattern
- [ ] New pages use `SmartRenderer` for rendering
- [ ] New pages use `SmartTableDS` for data tables
- [ ] New forms use `SmartForm` for modal inputs
- [ ] New dashboards use `SmartCharts` for visual cards
- [ ] `ProcedureMapper` maps entry procedures only, not every downstream SP
