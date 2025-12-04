# Database Schema Documentation

## Overview

This database schema supports a comprehensive certification and team management system. The schema is designed to be database-agnostic and includes multi-tenancy support through a `tenantid` field present in all tables. The system manages users, certifications, testing, classes, teams, organizational structures, and waivers.

All tables include standard audit columns:
- `tenantid` (VARCHAR(255), NOT NULL) - Multi-tenancy identifier
- `create_timestamp` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP) - Record creation timestamp
- `update_timestamp` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP) - Record update timestamp

> **Note:** This documentation has been reviewed for consistency, best practices, and optimization opportunities. Critical issues and recommendations are documented in [schema-review.md](schema-review.md). Known issues are marked with ⚠️ warnings in the affected table sections.

---

## Table of Contents

1. [System Domain (SYS)](#system-domain-sys)
2. [Certification Domain (CRT)](#certification-domain-crt)
3. [Class Domain (CLS)](#class-domain-cls)
4. [Organization Domain (ORG)](#organization-domain-org)
5. [Team Domain (TMS)](#team-domain-tms)
6. [Waiver Domain (WVR)](#waiver-domain-wvr)
7. [Entity Relationships](#entity-relationships)
8. [Common Design Patterns](#common-design-patterns)

---

## System Domain (SYS)

The System domain contains tables for application configuration, user management, security roles, and user-related documents.

### sys_property

Stores system-wide configuration properties as key-value pairs.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Property name |
| property_value | VARCHAR(255) | No | Property value |

**Constraints:**
- Primary Key: `pk_sys_property` on (id)
- Unique Index: `ux_sys_property_name` on (name, tenantid)
- Index: `ix_tenantid` on (tenantid)

**Purpose:** Maintains configuration settings for the system, supporting multi-tenant deployments where properties can be scoped by tenant.

---

### sys_role

Defines security roles within the system for role-based access control.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Role name |
| description | VARCHAR(255) | No | Role description |

**Constraints:**
- Primary Key: `pk_sys_role` on (id)
- Unique Index: `ux_role_name` on (name)

**Relationships:**
- Referenced by `sys_user_role.role_id`

**Purpose:** Manages role-based access control within the system.

---

### sys_user

Stores user account information.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| username | VARCHAR(255) | No | | Unique username |
| locale_id | VARCHAR(50) | No | 'en_US' | User locale preference |
| last_name | VARCHAR(255) | No | | User's last name |
| first_name | VARCHAR(255) | No | | User's first name |
| email_address | VARCHAR(255) | No | | User's email address |
| certification_id | INTEGER | No | | Reference to user's certification |

**Constraints:**
- Primary Key: `pk_sys_user` on (id)
- Unique Index: `ux_user_username` on (username)
- Unique Index: `ux_user_email_address` on (email_address)
- Foreign Key: `fk_sys_user_certification_id` → crt_certification(id) [NO ACTION on delete/update]

**Relationships:**
- References `crt_certification.id` via certification_id
- Referenced by `sys_user_role.user_id`
- Referenced by `sys_user_bg_check.user_id`
- Referenced by `sys_user_document.user_id`
- Referenced by `cls_class_roster.student_id`
- Referenced by `crt_test_attempt.taker_id`

**Note:** Password column is commented out in the schema, suggesting external authentication.

**Purpose:** Central user account management with support for internationalization and certification tracking.

---

### sys_user_role

Junction table linking users to their assigned roles (many-to-many relationship).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| user_id | INTEGER | No | Reference to user |
| role_id | INTEGER | No | Reference to role |

**Constraints:**
- Primary Key: `pk_sys_user_role` on (id)
- Unique Constraint: `ux_user_user_role` on (user_id, role_id)
- Foreign Key: `fk_sys_user_role_user_id` → sys_user(id) [NO ACTION on delete/update]
- Foreign Key: `fk_sys_user_role_id` → sys_role(id) [NO ACTION on delete/update]

**Purpose:** Implements many-to-many relationship between users and roles, preventing duplicate role assignments.

---

### sys_user_bg_check

Stores background check verification records for users.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| user_id | INTEGER | No | Reference to user |
| issuer | VARCHAR(255) | No | Background check provider |
| verified | DATE | No | Date verification completed |
| expires | DATE | No | Expiration date of background check |

**Constraints:**
- Primary Key: `pk_sys_user_bg_check` on (id)
- Foreign Key: `fk_sys_user_bg_check_user_id` → sys_user(id) [NO ACTION on delete/update]

**Relationships:**
- References `sys_user.id` via user_id

**Purpose:** Tracks background check verifications and expiration dates, supporting compliance requirements.

---

### sys_user_document

Stores binary documents associated with users.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| user_id | INTEGER | No | Reference to user |
| type | VARCHAR(255) | No | Document type |
| document_date | DATE | No | Date of document |
| document | BLOB | No | Binary document content |

**Constraints:**
- Primary Key: `pk_sys_user_document_check` on (id)
- Foreign Key: `fk_sys_user_document_user_id` → sys_user(id) [NO ACTION on delete/update]

**Relationships:**
- References `sys_user.id` via user_id

**Purpose:** Manages user-related documents in binary format with type classification and dating.

---

## Certification Domain (CRT)

The Certification domain manages certifications, tests, question banks, test attempts, and scoring.

### crt_certification

Defines certification types and requirements.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| name | VARCHAR(255) | No | | Certification name |
| description | VARCHAR(255) | No | | Certification description |
| parent_certification_id | INTEGER | Yes | | Reference to parent certification (hierarchical) |
| years_valid | INTEGER | No | 2 | Number of years certification is valid |
| background_check_required | BOOLEAN | No | false | Whether background check is required |

**Constraints:**
- Primary Key: `pk_crt_certification` on (id)
- Unique Index: `ux_crt_certification_name` on (name)
- Index: `ix_crt_certification_tenantid` on (tenantid)
- Foreign Key: `fk_parent_crt_certification_id` → crt_certification(id) [NO ACTION on delete/update] (self-reference)

**Relationships:**
- Self-referencing via parent_certification_id (supports certification hierarchies)
- Referenced by `sys_user.certification_id`
- Referenced by `crt_test.crt_certification_id`
- Referenced by `crt_question.crt_certification_id`
- Referenced by `cls_class_type.certification_id`

**Purpose:** Manages certification types with support for hierarchical relationships and validity periods.

---

### crt_test

Defines tests associated with certifications.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| name | VARCHAR(255) | No | | Test name |
| description | VARCHAR(255) | No | | Test description |
| crt_certification_id | INTEGER | No | | Reference to certification |
| passing_score | INTEGER | No | 90 | Minimum passing score (percentage) |
| number_questions | INTEGER | No | 50 | Number of questions on the test |
| max_attempts | INTEGER | No | 3 | Maximum allowed attempts |
| active_from | DATE | No | | Date test becomes active |
| expires_after | DATE | Yes | | Date test expires (optional) |

**Constraints:**
- Primary Key: `pk_crt_test` on (id)
- Unique Index: `ix_crt_test_name` on (name)
- Foreign Key: `fk_crt_certification_id` → crt_certification(id) [NO ACTION on delete/update]

**Relationships:**
- References `crt_certification.id` via crt_certification_id
- Referenced by `crt_test_question.crt_test_id`
- Referenced by `crt_test_attempt.crt_test_id`

**Purpose:** Defines certification tests with configurable parameters for passing criteria, attempt limits, and validity periods.

---

### crt_question

Stores the master question bank for certifications.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| crt_certification_id | INTEGER | No | | Reference to certification |
| question | VARCHAR(255) | No | | Question text |
| question_type | VARCHAR(255) | No | | Type of question |
| answer_1 | VARCHAR(255) | No | | First answer option (required) |
| answer_2 | VARCHAR(255) | Yes | | Second answer option |
| answer_3 | VARCHAR(255) | Yes | | Third answer option |
| answer_4 | VARCHAR(255) | Yes | | Fourth answer option |
| answer_5 | VARCHAR(255) | Yes | | Fifth answer option |
| answer_6 | VARCHAR(255) | Yes | | Sixth answer option |
| correct_answer | SMALLINT | No | | Index of correct answer (1-6) |
| active | BOOLEAN | No | true | Whether question is active |

**Constraints:**
- Primary Key: `pk_crt_question` on (id)
- Foreign Key: `fk_crt_question_certification_id` → crt_certification(id) [NO ACTION on delete/update]

**Relationships:**
- References `crt_certification.id` via crt_certification_id

**Purpose:** Maintains a pool of questions for certifications with support for multiple-choice answers (up to 6 options) and active/inactive status for question lifecycle management.

---

### crt_test_question

Stores questions assigned to a specific test instance (snapshot).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| crt_test_id | INTEGER | No | Reference to test |
| question | VARCHAR(255) | No | Question text |
| question_type | VARCHAR(255) | No | Type of question |
| answer_1 | VARCHAR(255) | No | First answer option |
| answer_2 | VARCHAR(255) | Yes | Second answer option |
| answer_3 | VARCHAR(255) | Yes | Third answer option |
| answer_4 | VARCHAR(255) | Yes | Fourth answer option |
| answer_5 | VARCHAR(255) | Yes | Fifth answer option |
| answer_6 | VARCHAR(255) | Yes | Sixth answer option |
| correct_answer | SMALLINT | No | Index of correct answer |

**Constraints:**
- Primary Key: `pk_crt_test_question` on (id)
- Foreign Key: `fk_crt_test_id` → crt_test(id) [NO ACTION on delete/update]

**Relationships:**
- References `crt_test.id` via crt_test_id
- Referenced by `crt_test_attempt_answer.crt_test_question_id`

**Purpose:** Snapshot of questions for a specific test instance, allowing questions in the master bank to be modified without affecting existing or in-progress tests.

---

### crt_test_attempt

Records a user's attempt at taking a test.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| started | TIMESTAMP | No | | Timestamp when test was started |
| submitted | TIMESTAMP | No | | Timestamp when test was submitted |
| crt_test_id | INTEGER | No | | Reference to test |
| taker_id | INTEGER | No | | Reference to user taking the test |
| score | SMALLINT | No | 0 | Score achieved |

**Constraints:**
- Primary Key: `pk_crt_test_attempt` on (id)
- Foreign Key: `fk_crt_test_attempt_test_id` → crt_test(id) [NO ACTION on delete/update]
- Foreign Key: `fk_crt_test_attempt_user_id` → sys_user(id) [NO ACTION on delete/update]

**Relationships:**
- References `crt_test.id` via crt_test_id
- References `sys_user.id` via taker_id
- Referenced by `crt_test_attempt_answer.crt_test_attempt_id`

**Purpose:** Tracks individual test attempts with timing information (start/submit) and scoring, enabling enforcement of attempt limits and performance tracking.

---

### crt_test_attempt_answer

Stores individual answers provided during a test attempt.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Answer identifier |
| crt_test_attempt_id | INTEGER | No | Reference to test attempt |
| crt_test_question_id | INTEGER | No | Reference to test question |
| answer | INTEGER | No | Selected answer index |

**Constraints:**
- Primary Key: `pk_crt_test_attempt_answer` on (id)
- Foreign Key: `fk_crt_test_attempt_answer_test_id` → crt_test_attempt(id) [NO ACTION on delete/update]
- Foreign Key: `fk_crt_test_attempt_answer_question_id` → crt_test_question(id) [NO ACTION on delete/update]

**Relationships:**
- References `crt_test_attempt.id` via crt_test_attempt_id
- References `crt_test_question.id` via crt_test_question_id

**Purpose:** Stores individual answers for each question in a test attempt, enabling detailed test analysis and scoring verification.

---

## Class Domain (CLS)

The Class domain manages certification classes, class types, student enrollment, and class locations.

### cls_class_type

Defines types of certification classes.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Class type name |
| certification_id | INTEGER | No | Reference to certification |

**Constraints:**
- Primary Key: `pk_cls_class_type` on (id)
- Unique Index: `ix_cls_class_type_name` on (name)
- Foreign Key: `fk_cls_class_type_certification_id` → crt_certification(id) [NO ACTION on delete/update]

**Relationships:**
- References `crt_certification.id` via certification_id
- Referenced by `cls_class.class_type_id`

**Purpose:** Categorizes classes by type and associates them with specific certifications.

---

### cls_class

Defines individual class instances.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Class name |
| class_type_id | INTEGER | No | Reference to class type |

**Constraints:**
- Primary Key: `pk_cls_class` on (id)
- Unique Index: `ix_cls_class_name` on (name)
- Foreign Key: `fk_cls_class_class_type_id` → cls_class_type(id) [NO ACTION on delete/update]

**Relationships:**
- References `cls_class_type.id` via class_type_id
- Referenced by `cls_class_roster.class_id`
- Referenced by `cls_class_location.class_id`

**Purpose:** Manages individual class offerings tied to specific class types.

---

### cls_class_roster

Tracks students enrolled in or attending a class.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Roster entry name |
| class_id | INTEGER | No | Reference to class |
| student_id | INTEGER | No | Reference to user (student) |
| email | VARCHAR(255) | No | Student email |
| mobile | VARCHAR(255) | No | Student mobile phone |
| attended_class | BOOLEAN | No | Whether student attended |
| walk_in | BOOLEAN | No | Whether student was a walk-in |

**Constraints:**
- Primary Key: `pk_cls_class_roster` on (id)
- Foreign Key: `fk_cls_class_roster_class_id` → cls_class(id) [NO ACTION on delete/update]
- Foreign Key: `fk_cls_class_roster_user_id` → sys_user(id) [NO ACTION on delete/update]

**Relationships:**
- References `cls_class.id` via class_id
- References `sys_user.id` via student_id

**Purpose:** Manages class attendance and enrollment, including walk-in students, with contact information for each enrollee.

---

### cls_class_location

Defines physical locations where classes are held.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Location name |
| class_id | INTEGER | No | Reference to class |
| address | VARCHAR(255) | Yes | Street address |
| city | VARCHAR(255) | Yes | City |
| state | VARCHAR(255) | Yes | State/Province |
| postal_code | VARCHAR(255) | Yes | Postal/ZIP code |
| email | VARCHAR(255) | No | Contact email |
| mobile | VARCHAR(255) | No | Contact phone |

**Constraints:**
- Primary Key: `pk_cls_class_location` on (id)
- Foreign Key: `fk_cls_class_location_id` → cls_class(id) [NO ACTION on delete/update]

**Relationships:**
- References `cls_class.id` via class_id

**Purpose:** Maintains location and contact information for class venues.

---

## Organization Domain (ORG)

The Organization domain defines the hierarchical structure of leagues and divisions.

### org_league

Defines a league organization (top-level organizational entity).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | League name |
| description | VARCHAR(255) | No | League description |

**Constraints:**
- Primary Key: `pk_org_league` on (id)
- Unique Index: `ux_org_league_name` on (name)

**Relationships:**
- Referenced by `org_division.org_league_id`

**Purpose:** Top-level organizational entity representing leagues.

---

### org_division

Defines divisions within a league.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Division name |
| description | VARCHAR(255) | No | Division description |
| org_league_id | INTEGER | No | Reference to parent league |

**Constraints:**
- Primary Key: `pk_org_division` on (id)
- Unique Index: `ux_org_division_name` on (name)
- Foreign Key: `fk_org_division_league` → org_league(id) [CASCADE on delete, RESTRICT on update]

**Relationships:**
- References `org_league.id` via org_league_id
- Referenced by `tms_team.org_division_id`

**Purpose:** Subdivides leagues into divisions, organizing teams hierarchically. Cascade delete ensures divisions are removed when parent league is deleted.

---

## Team Domain (TMS)

The Team domain manages teams, pools, team roles, team contacts, and team statuses.

### tms_pool

Defines swimming pool facilities.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| name | VARCHAR(255) | No | | Pool name |
| address | VARCHAR(255) | Yes | | Street address |
| length | SMALLINT | No | 25 | Pool length |
| length_unit | VARCHAR(1) | No | 'Y' | Length unit (Y=yards, M=meters) |
| lane_count | SMALLINT | No | 6 | Number of lanes |
| bidirectional | BOOLEAN | No | true | Whether pool can be used in both directions |
| parking_rules | VARCHAR(255) | Yes | | Parking instructions |
| setup_rules | VARCHAR(255) | Yes | | Setup instructions |
| restrictions | VARCHAR(255) | Yes | | Usage restrictions |

**Constraints:**
- Primary Key: `pk_tms_pool` on (id)
- Unique Index: `ux_tms_pool_name` on (name)

**Relationships:**
- Referenced by `tms_team.tms_pool_id`

**Purpose:** Manages pool facilities with detailed specifications (dimensions, lanes) and operational rules (parking, setup, restrictions).

---

### tms_team_role

Defines roles that contacts can have within a team.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Role name |

**Constraints:**
- Primary Key: `pk_tms_team_role` on (id)
- Unique Index: `tms_team_role_name_ux` on (name)

**Relationships:**
- Referenced by `tms_team_contact.tms_team_role_id`

**Initial Data:**
- Team Rep
- President
- Clerk of Course
- Head Referee
- Coach

**Purpose:** Defines standard roles for team contacts and officials.

---

### tms_team_status

Defines possible statuses for teams.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| name | VARCHAR(255) | No | | Status name |
| active | BOOLEAN | No | true | Whether status represents an active team |

**Constraints:**
- Primary Key: `pk_tms_team_status` on (id)
- Unique Index: `ux_tms_team_status_name` on (name)

**Relationships:**
- Referenced by `tms_team.tms_team_status_id`

**Initial Data:**
- ACTIVE (active=true)
- INACTIVE (active=false)
- SUSPENDED (active=false)
- PENDING (active=false)

**Purpose:** Manages team lifecycle states with active/inactive indicator.

---

### tms_team_contact

Stores contact information for team personnel.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | INTEGER | No | | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | | Tenant identifier |
| name | VARCHAR(255) | No | | Contact name |
| tms_team_role_id | INTEGER | Yes | | Reference to team role |
| address | VARCHAR(255) | Yes | | Street address |
| city | VARCHAR(255) | Yes | | City |
| state | VARCHAR(255) | Yes | | State/Province |
| postal_code | VARCHAR(255) | Yes | | Postal/ZIP code |
| email | VARCHAR(255) | No | | Email address |
| mobile | VARCHAR(255) | No | | Mobile phone number |
| comments | VARCHAR(255) | Yes | | Additional comments |
| active | BOOLEAN | No | true | Whether contact is active |

**Constraints:**
- Primary Key: `pk_tms_team_contact` on (id)
- Foreign Key: `fk_tms_team_contact_role` → tms_team_role(id) [CASCADE on delete, RESTRICT on update]

**Relationships:**
- References `tms_team_role.id` via tms_team_role_id

**Purpose:** Maintains contact information for team officials and volunteers with optional role assignment.

---

### tms_team

Defines a swim team.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| name | VARCHAR(255) | No | Team name |
| code | VARCHAR(255) | No | Team code/abbreviation |
| mascot | VARCHAR(255) | No | Team mascot |
| tms_team_status_id | INTEGER | No | Reference to team status |
| org_division_id | INTEGER | Yes | Reference to division |
| tms_pool_id | INTEGER | Yes | Reference to home pool |

**Constraints:**
- Primary Key: `pk_tms_team` on (id)
- Unique Index: `ux_tms_team_name` on (name)
- Unique Index: `ux_tms_team_code` on (code)
- Foreign Key: `fk_tms_team_division` → org_division(id) [CASCADE on delete, RESTRICT on update]
- Foreign Key: `fk_tms_team_pool` → tms_pool(id) [CASCADE on delete, RESTRICT on update]
- Foreign Key: `fk_tms_team_status` → tms_team_status(id) [CASCADE on delete, RESTRICT on update]

**Relationships:**
- References `tms_team_status.id` via tms_team_status_id
- References `org_division.id` via org_division_id
- References `tms_pool.id` via tms_pool_id
- Referenced by `wvr_waiver.from_team`
- Referenced by `wvr_waiver.to_team`
- Referenced by `wvr_waiver.submitby_team`

**Purpose:** Central entity for team management with organizational hierarchy, facility assignment, and status tracking. Cascade deletes ensure referential integrity when divisions or pools are removed.

---

## Waiver Domain (WVR)

The Waiver domain manages swimmer transfer requests between teams.

### wvr_waiver_status

Defines possible statuses for waiver requests.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| parent_status_id | INTEGER | Yes | Reference to parent status (hierarchical) |
| name | VARCHAR(255) | No | Status name |
| approved | BOOLEAN | No | Whether status represents approval |

**Constraints:**
- Primary Key: `pk_wvr_waiver_status` on (id)
- Unique Index: `ux_wvr_waiver_status_name` on (name)
- Foreign Key: `fk_wvr_waiver_parent_status` → wvr_waiver_status(id) [CASCADE on delete, RESTRICT on update] (self-reference)

**Relationships:**
- Self-referencing via parent_status_id (supports status hierarchies)
- Referenced by `wvr_waiver.teamrep_status`
- Referenced by `wvr_waiver.org_status`

**Purpose:** Manages waiver workflow states with support for hierarchical status relationships.

---

### wvr_waiver

Stores waiver/transfer requests for swimmers.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| swimmer_name | VARCHAR(255) | No | Swimmer's name |
| swimmer_address | VARCHAR(255) | Yes | Swimmer's address |
| swimmer_city | VARCHAR(255) | Yes | Swimmer's city |
| swimmer_state | VARCHAR(255) | Yes | Swimmer's state |
| swimmer_postal_code | VARCHAR(255) | Yes | Swimmer's postal code |
| swimmer_neighborhood | VARCHAR(255) | Yes | Swimmer's neighborhood |
| from_team | INTEGER | No | Reference to current team |
| to_team | INTEGER | No | Reference to requested team |
| swimmer_parent_name | VARCHAR(255) | No | Parent/guardian name |
| swimmer_parent_phone | VARCHAR(255) | No | Parent/guardian phone |
| swimmer_parent_email | VARCHAR(255) | No | Parent/guardian email |
| submitby_team | INTEGER | No | Reference to submitting team |
| submitby_name | VARCHAR(255) | No | Submitter's name |
| submitby_phone | VARCHAR(255) | No | Submitter's phone |
| submitby_email | VARCHAR(255) | No | Submitter's email |
| mobile | VARCHAR(255) | No | Mobile contact number |
| address | VARCHAR(255) | Yes | Additional address |
| waiver_reason | VARCHAR(255) | Yes | Reason for waiver request |
| teamrep_status | INTEGER | Yes | Team representative status (FK to wvr_waiver_status) |
| org_status | INTEGER | Yes | Organization status (FK to wvr_waiver_status) |
| org_date | DATE | Yes | Organization decision date |

**Constraints:**
- Primary Key: `pk_wvr_waiver` on (id)
- Unique Index: `ux_wvr_waiver_name` on (swimmer_name, from_team, swimmer_parent_name)
- Foreign Key: `fk_wvr_waiver_teamrep_status` → wvr_waiver_status(id) [CASCADE on delete, RESTRICT on update]
- Foreign Key: `fk_wvr_waiver_org_status` → wvr_waiver_status(id) [CASCADE on delete, RESTRICT on update]
- Foreign Key: `fk_wvr_waiver_fromteam` → tms_team(id) [CASCADE on delete, RESTRICT on update]
- Foreign Key: `fk_wvr_waiver_toteam` → tms_team(id) [CASCADE on delete, RESTRICT on update]
- Foreign Key: `fk_wvr_waiver_submitbyteam` → tms_team(id) [CASCADE on delete, RESTRICT on update]

**Relationships:**
- References `tms_team.id` via from_team
- References `tms_team.id` via to_team
- References `tms_team.id` via submitby_team
- References `wvr_waiver_status.id` via teamrep_status
- References `wvr_waiver_status.id` via org_status
- Referenced by `wvr_waiver_approval.waiver_id`

**Purpose:** Manages waiver requests for swimmers transferring between teams, tracking multi-level approval workflow (team rep and organization) with comprehensive contact information.

---

### wvr_waiver_approval

Records waiver approval decisions.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | INTEGER | No | Primary key (auto-increment) |
| tenantid | VARCHAR(255) | No | Tenant identifier |
| waiver_id | INTEGER | No | Reference to waiver request |
| team_rep_id | INTEGER | No | Reference to team representative user |
| approved_date | DATE | No | Date of approval decision |
| comments | VARCHAR(255) | No | Approval comments |

**Constraints:**

- Primary Key: `pk_wvr_waiver_approval` on (id)
- Foreign Key: `fk_wvr_waiver_approval_waiver_id` → wvr_waiver(id) [CASCADE on delete, RESTRICT on update]
- Foreign Key: `fk_wvr_waiver_approval_team_rep_id` → sys_user(id) [NO ACTION on delete/update]

**Relationships:**

- References `wvr_waiver.id` via waiver_id
- References `sys_user.id` via team_rep_id

**Purpose:** Tracks approvals and associated comments from team representatives for specific waiver requests.

---

## Entity Relationships

### Organization Hierarchy

```
org_league (1) ──< (M) org_division (1) ──< (M) tms_team
                                                    │
                                                    ├──> tms_team_status (M) ──< (1)
                                                    ├──> org_division (M) ──< (1)
                                                    └──> tms_pool (M) ──< (1)
```

### Certification & Testing Flow

```
crt_certification (self-referencing)
        │
        ├──> crt_test (1) ──< (M) ──> crt_test_question (1) ──< (M)
        │                                                           │
        │                                                           ▼
        │                                                    crt_test_attempt (1) ──< (M)
        │                                                           │
        │                                                           ▼
        │                                                    crt_test_attempt_answer
        │
        ├──> crt_question (1) ──< (M)
        │
        └──> cls_class_type (1) ──< (M) ──> cls_class (1) ──< (M) ──> cls_class_roster
                                                    │                            │
                                                    │                            ▼
                                                    │                        sys_user
                                                    │
                                                    └──> cls_class_location (1) ──< (M)
```

### User Management

```
sys_user ──> crt_certification (M) ──< (1)
    │
    ├──> sys_user_role (1) ──< (M) ──> sys_role (M) ──< (1)
    │
    ├──> sys_user_bg_check (1) ──< (M)
    │
    ├──> sys_user_document (1) ──< (M)
    │
    ├──> cls_class_roster (1) ──< (M)
    │
    └──> crt_test_attempt (1) ──< (M)
```

### Waiver Process

```
tms_team ──> wvr_waiver (via from_team, to_team, submitby_team)
                │
                ├──> wvr_waiver_status (via teamrep_status)
                │
                ├──> wvr_waiver_status (via org_status)
                │
                └──> wvr_waiver_approval (1) ──< (M)
                            │
                            └──> sys_user (via team_rep_id)

wvr_waiver_status (self-referencing via parent_status_id)
```

---

## Common Design Patterns

### Multi-Tenancy

All tables include a `tenantid` column (VARCHAR(255), NOT NULL) to support multi-tenant deployments. This enables multiple organizations to share the same database infrastructure while maintaining data isolation. Queries should always include tenant filtering to ensure proper data separation.

### Audit Timestamps

All tables include automatic timestamp tracking:
- `create_timestamp` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP) - Automatically set on record creation
- `update_timestamp` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP) - Set on record creation; application responsible for updates

### Primary Keys

All tables use auto-incrementing integer primary keys named `id`. This provides:
- Simple, consistent key structure
- Efficient indexing and joins
- Database-agnostic compatibility

### Naming Conventions

- **Table Prefixes**: Tables use prefixes to indicate functional area
  - `sys_` - System management
  - `crt_` - Certification
  - `cls_` - Classes
  - `org_` - Organization
  - `tms_` - Teams
  - `wvr_` - Waivers

- **Column Names**: Foreign key columns typically include the referenced table name followed by `_id`

- **Constraint Naming**:
  - `pk_` - Primary keys
  - `ux_` - Unique constraints/indexes
  - `ix_` - Non-unique indexes
  - `fk_` - Foreign keys

### Hierarchical Structures

Two tables support hierarchical relationships through self-referencing foreign keys:
- `crt_certification.parent_certification_id` → `crt_certification.id`
- `wvr_waiver_status.parent_status_id` → `wvr_waiver_status.id`

### Referential Integrity

Foreign key constraints use different delete/update actions based on business logic:

- **NO ACTION**: Most relationships use NO ACTION, requiring explicit handling of dependent records
  - User-related tables
  - Certification and test relationships
  - Class management

- **CASCADE DELETE**: Used for hierarchical relationships where child records should be removed with parent
  - org_division → org_league
  - tms_team → org_division
  - tms_team → tms_pool
  - tms_team_contact → tms_team_role
  - wvr_waiver → tms_team
  - wvr_waiver → wvr_waiver_status

- **RESTRICT UPDATE**: Most CASCADE DELETE constraints pair with RESTRICT UPDATE to prevent accidental ID changes

### Default Values

Several tables include default values for configuration:
- **tms_pool**: length (25), length_unit ('Y'), lane_count (6), bidirectional (true)
- **crt_certification**: years_valid (2), background_check_required (false)
- **crt_test**: passing_score (90), number_questions (50), max_attempts (3)
- **sys_user**: locale_id ('en_US')
- **crt_test_attempt**: score (0)
- **Boolean flags**: Generally default to true or false based on expected common case

### Reference Data

Several tables are pre-populated with reference data:
- **tms_team_role**: Team Rep, President, Clerk of Course, Head Referee, Coach
- **tms_team_status**: ACTIVE, INACTIVE, SUSPENDED, PENDING

---

## Summary

This database schema provides a comprehensive solution for managing:

1. **Users and Security**: Role-based access control, user profiles, documents, and background checks
2. **Certifications**: Hierarchical certifications with configurable validity periods
3. **Testing**: Flexible test creation, question banks, attempt tracking, and detailed scoring
4. **Classes**: Class types, instances, enrollment, attendance, and location management
5. **Organizations**: League and division hierarchy
6. **Teams**: Team management with pools, contacts, roles, and status tracking
7. **Waivers**: Multi-level approval workflow for swimmer transfers between teams

The schema is designed for database portability (using standard data types), supports multi-tenancy, includes comprehensive audit tracking, and enforces referential integrity through foreign key constraints.

