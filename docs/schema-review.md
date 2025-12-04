# Database Schema Review and Recommendations

## Review Date
2025-12-03

## Overview
This document contains a comprehensive review of the database schema for consistency, best practices, and optimization opportunities.

---

## Critical Issues

### 1. ✅ RESOLVED: Data Type Inconsistency in wvr_waiver Table

**Issue:** The `wvr_waiver` table defines `teamrep_status` and `org_status` as VARCHAR(255), but the foreign key constraints reference `wvr_waiver_status.id` which is an INTEGER.

**Location:**
- [wvr-waiver-changelog-1.xml:137-142](../migrations/xml/wvr-waiver-changelog-1.xml)

**Current Schema:**
```xml
<column name="teamrep_status" type="VARCHAR(255)">
    <constraints nullable="true"/>
</column>
<column name="org_status" type="VARCHAR(255)">
    <constraints nullable="true"/>
</column>
```

**Foreign Key Constraints:**
```xml
<addForeignKeyConstraint baseColumnNames="teamrep_status"
    baseTableName="wvr_waiver"
    constraintName="fk_wvr_waiver_teamrep_status"
    referencedColumnNames="id"
    referencedTableName="wvr_waiver_status"
/>
```

**Impact:** HIGH - This will cause database creation to fail as you cannot create a foreign key from VARCHAR to INTEGER.

**Resolution:** Changed both columns to INTEGER type in [wvr-waiver-changelog-1.xml:137-142](../migrations/xml/wvr-waiver-changelog-1.xml)
```xml
<column name="teamrep_status" type="INT">
    <constraints nullable="true"/>
</column>
<column name="org_status" type="INT">
    <constraints nullable="true"/>
</column>
```

---

## Schema Design Issues

### 2. ✅ RESOLVED: Missing Foreign Key: wvr_waiver_approval.team_rep_id

**Issue:** The `wvr_waiver_approval.team_rep_id` column does not have a foreign key constraint to any table.

**Location:** [wvr-waiver-changelog-1.xml:244](../migrations/xml/wvr-waiver-changelog-1.xml)

**Impact:** MEDIUM - Data integrity issue; orphaned records possible.

**Resolution:** Added foreign key constraint to `sys_user(id)` in changeSet `wvr_waiver_approval-team_rep-fk`:
```xml
<changeSet author="${default_author}" id="wvr_waiver_approval-team_rep-fk">
    <comment>Link approval to team representative user</comment>
    <addForeignKeyConstraint baseColumnNames="team_rep_id"
        baseTableName="wvr_waiver_approval"
        constraintName="fk_wvr_waiver_approval_team_rep_id"
        onDelete="NO ACTION"
        onUpdate="NO ACTION"
        referencedColumnNames="id"
        referencedTableName="sys_user"
    />
</changeSet>
```

### 3. ✅ RESOLVED: Missing Relationship: wvr_waiver_approval Not Linked to wvr_waiver

**Issue:** The `wvr_waiver_approval` table has no foreign key to `wvr_waiver`, making it impossible to determine which waiver an approval belongs to.

**Impact:** HIGH - Critical data model issue; approvals cannot be associated with waivers.

**Resolution:** Added `waiver_id` column to the table and created foreign key constraint in changeSet `wvr_waiver_approval-waiver-fk`:
```xml
<changeSet author="${default_author}" id="wvr_waiver_approval-add-waiver_id">
    <comment>Add waiver reference to approval</comment>
    <addColumn tableName="wvr_waiver_approval">
        <column name="waiver_id" type="INT">
            <constraints nullable="false"/>
        </column>
    </addColumn>
</changeSet>

<changeSet author="${default_author}" id="wvr_waiver_approval-waiver-fk">
    <comment>Link approval to waiver</comment>
    <addForeignKeyConstraint baseColumnNames="waiver_id"
        baseTableName="wvr_waiver_approval"
        constraintName="fk_wvr_waiver_approval_waiver_id"
        onDelete="CASCADE"
        onUpdate="RESTRICT"
        referencedColumnNames="id"
        referencedTableName="wvr_waiver"
    />
</changeSet>
```

---

## Optimization Opportunities

### 4. Missing Indexes on Foreign Keys

**Issue:** Several foreign key columns lack indexes, which can significantly impact query performance.

**Impact:** MEDIUM - Performance degradation on joins and lookups.

**Affected Tables:**

1. **sys_user.certification_id** - No index on foreign key
2. **sys_user_role.user_id** - No index
3. **sys_user_role.role_id** - No index
4. **sys_user_bg_check.user_id** - No index
5. **sys_user_document.user_id** - No index
6. **crt_test.crt_certification_id** - No index
7. **crt_test_question.crt_test_id** - No index
8. **crt_test_attempt.crt_test_id** - No index
9. **crt_test_attempt.taker_id** - No index
10. **crt_test_attempt_answer.crt_test_attempt_id** - No index
11. **crt_test_attempt_answer.crt_test_question_id** - No index
12. **cls_class_roster.class_id** - No index
13. **cls_class_roster.student_id** - No index

**Recommendation:** Add indexes to all foreign key columns for optimal join performance:
```xml
<changeSet author="${default_author}" id="add-foreign-key-indexes">
    <comment>Add indexes on foreign key columns for performance</comment>
    <createIndex indexName="ix_sys_user_certification_id" tableName="sys_user">
        <column name="certification_id"/>
    </createIndex>
    <createIndex indexName="ix_sys_user_role_user_id" tableName="sys_user_role">
        <column name="user_id"/>
    </createIndex>
    <createIndex indexName="ix_sys_user_role_role_id" tableName="sys_user_role">
        <column name="role_id"/>
    </createIndex>
    <!-- Add remaining indexes -->
</changeSet>
```

### 5. Missing Composite Indexes for Common Query Patterns

**Issue:** Several tables would benefit from composite indexes based on expected query patterns.

**Recommendations:**

**a) sys_user_bg_check:**
```xml
<createIndex indexName="ix_sys_user_bg_check_user_expires" tableName="sys_user_bg_check">
    <column name="user_id"/>
    <column name="expires"/>
</createIndex>
```
*Rationale:* Queries checking for expired background checks for specific users.

**b) crt_test:**
```xml
<createIndex indexName="ix_crt_test_cert_active" tableName="crt_test">
    <column name="crt_certification_id"/>
    <column name="active_from"/>
    <column name="expires_after"/>
</createIndex>
```
*Rationale:* Finding active tests for a certification within date range.

**c) crt_test_attempt:**
```xml
<createIndex indexName="ix_crt_test_attempt_user_test" tableName="crt_test_attempt">
    <column name="taker_id"/>
    <column name="crt_test_id"/>
    <column name="submitted"/>
</createIndex>
```
*Rationale:* Finding test attempts for a user, ordered by submission date.

**d) cls_class_roster:**
```xml
<createIndex indexName="ix_cls_class_roster_student_attended" tableName="cls_class_roster">
    <column name="student_id"/>
    <column name="attended_class"/>
</createIndex>
```
*Rationale:* Finding classes attended by a specific student.

### 6. Missing Index on tenantid for Multi-Tenant Queries

**Issue:** Most tables lack indexes on `tenantid`, which will be included in virtually all queries.

**Impact:** MEDIUM-HIGH - All queries will be slower without tenant filtering optimization.

**Recommendation:** Add tenantid index to all tables (except sys_property which already has it):
```xml
<changeSet author="${default_author}" id="add-tenantid-indexes">
    <comment>Add indexes on tenantid for multi-tenant query optimization</comment>
    <createIndex indexName="ix_sys_role_tenantid" tableName="sys_role">
        <column name="tenantid"/>
    </createIndex>
    <createIndex indexName="ix_sys_user_tenantid" tableName="sys_user">
        <column name="tenantid"/>
    </createIndex>
    <!-- Add to all remaining tables -->
</changeSet>
```

---

## Best Practices Issues

### 7. Inconsistent Unique Index Naming

**Issue:** Some unique indexes use `ux_` prefix while others use `ix_` prefix, even though they are unique.

**Examples:**
- `ix_crt_certification_name` (unique) - should be `ux_`
- `ix_crt_test_name` (unique) - should be `ux_`
- `ix_cls_class_type_name` (unique) - should be `ux_`
- `ix_cls_class_name` (unique) - should be `ux_`

**Impact:** LOW - Documentation clarity issue.

**Recommendation:** Use consistent naming convention:
- `ux_` for unique indexes
- `ix_` for non-unique indexes

### 8. VARCHAR(255) Size May Be Excessive

**Issue:** Many columns use VARCHAR(255) which may be unnecessarily large for certain fields.

**Examples:**
- `tms_pool.length_unit` - VARCHAR(1) is correct, but could be CHAR(1)
- `sys_user.locale_id` - VARCHAR(50) is appropriate
- Question text and answers - VARCHAR(255) may be too small for complex questions

**Recommendation:** Consider more appropriate sizes:
- **Short codes** (2-10 chars): team codes, status codes
- **Names** (50-100 chars): person names, short descriptions
- **Long text** (500-1000+ chars): question text, descriptions, comments
- **Email addresses** (254 chars): RFC 5321 maximum
- **Phone numbers** (20 chars): International format

**Specific Suggestions:**
```xml
<!-- Instead of VARCHAR(255) for questions -->
<column name="question" type="VARCHAR(1000)">
    <constraints nullable="false"/>
</column>

<!-- For team codes -->
<column name="code" type="VARCHAR(10)">
    <constraints nullable="false"/>
</column>

<!-- For status names -->
<column name="name" type="VARCHAR(50)">
    <constraints nullable="false"/>
</column>
```

### 9. Missing NOT NULL Constraints on Some Foreign Keys

**Issue:** Some foreign keys that logically should always have a value are nullable.

**Examples:**
- `tms_team_contact.tms_team_role_id` - Currently nullable, but a contact should have a role
- `wvr_waiver.teamrep_status` - Should track status from creation

**Impact:** LOW-MEDIUM - Data quality issue.

**Recommendation:** Review business logic and add NOT NULL constraints where appropriate.

### 10. SMALLINT vs INTEGER for Scores and Counts

**Issue:** Using SMALLINT for scores limits range to -32,768 to 32,767 (or 0-65,535 unsigned).

**Current Usage:**
- `crt_test_attempt.score` - SMALLINT (good for 0-100 scores)
- `crt_question.correct_answer` - SMALLINT (good for 1-6)
- `crt_test_attempt_answer.answer` - INTEGER (could be SMALLINT)

**Recommendation:** Current usage is generally appropriate, but ensure consistency:
- Use SMALLINT for: scores (0-100), small counts (0-999), answer indexes
- Use INTEGER for: IDs, large counts, unconstrained numbers

---

## Data Model Improvements

### 11. Consider Soft Deletes for Historical Data

**Issue:** CASCADE DELETE on several relationships may result in loss of historical data.

**Examples:**
- Deleting a team cascades to delete all waivers
- Deleting a test cascades to delete all attempts and answers

**Impact:** MEDIUM - Potential data loss for reporting/auditing.

**Recommendation:** Consider adding soft delete pattern:
```xml
<column name="deleted" type="BOOLEAN" defaultValue="false">
    <constraints nullable="false"/>
</column>
<column name="deleted_timestamp" type="TIMESTAMP">
    <constraints nullable="true"/>
</column>
```

Then change CASCADE to NO ACTION and handle soft deletes in application logic.

### 12. Missing Audit Fields for Compliance

**Issue:** Tables have create/update timestamps but not user tracking.

**Recommendation:** Consider adding audit columns for compliance:
```xml
<column name="created_by" type="INT">
    <constraints nullable="true"/>
</column>
<column name="updated_by" type="INT">
    <constraints nullable="true"/>
</column>
```

---

## Security Considerations

### 13. Sensitive Data Storage

**Issue:** The schema stores sensitive information (background checks, documents, personal addresses) without encryption indicators.

**Affected Tables:**
- `sys_user_bg_check` - Background check information
- `sys_user_document` - Document BLOBs
- `wvr_waiver` - Personal addresses, phone numbers, emails
- `cls_class_roster` - Contact information

**Recommendation:**
1. Document which columns should be encrypted at rest
2. Consider using database-level encryption for sensitive columns
3. Add notes in schema about PII (Personally Identifiable Information)
4. Consider data retention policies and GDPR compliance

---

## Documentation Issues

### 14. Inconsistent Comment Quality in Migration Files

**Issue:** Some foreign key constraints have placeholder or incorrect comments.

**Examples:**
- [wvr-waiver-changelog-1.xml:220](../migrations/xml/wvr-waiver-changelog-1.xml): "Divisions haveTeams" (copy-paste error)
- [wvr-waiver-changelog-1.xml:171,183](../migrations/xml/wvr-waiver-changelog-1.xml): "parent status" (too generic)

**Recommendation:** Update comments to be more descriptive:
```xml
<comment>Links waiver to the team submitting the request</comment>
```

---

## Priority Summary

### Critical (Fix Immediately)
1. ✅ **wvr_waiver status columns data type** - Will prevent database creation
2. ✅ **wvr_waiver_approval missing waiver_id** - Critical data model issue

### High Priority (Fix Before Production)
3. ✅ **RESOLVED**: Add foreign key for wvr_waiver_approval.team_rep_id
4. ✅ **RESOLVED**: Add indexes on all foreign key columns
5. ✅ **RESOLVED**: Add tenantid indexes for all tables

### Medium Priority (Optimize Performance)
6. ✅ **RESOLVED**: Add composite indexes for common query patterns
7. Review VARCHAR sizes for optimization (deferred - application-specific)
8. Consider soft delete pattern for historical data (deferred - business decision)

### Low Priority (Improve Maintainability)
9. ✅ **RESOLVED**: Standardize unique index naming (ux_ vs ix_)
10. ✅ **RESOLVED**: Improve migration file comments
11. Add audit trail fields (created_by, updated_by) (deferred - future enhancement)
12. Document PII fields for compliance (deferred - separate documentation)

---

## Conclusion

The database schema is well-structured with good use of:

- Multi-tenancy support
- Audit timestamps
- Consistent naming conventions
- Hierarchical relationships
- Appropriate use of foreign keys

### Status Update

**All Critical and High Priority Issues Have Been Resolved:**

**Critical Issues:**

1. ✅ Data type inconsistencies in the waiver status columns - Fixed
2. ✅ Missing relationship between waiver approvals and waivers - Fixed
3. ✅ Missing foreign key for team_rep_id - Fixed

**High Priority Issues:**

4. ✅ Indexes on all foreign key columns - Added via performance-indexes-changelog-1.xml
5. ✅ Tenantid indexes for all tables - Added for multi-tenant query optimization

**Medium Priority Issues:**

6. ✅ Composite indexes for common query patterns - Added (background checks, test attempts, class rosters)

**Low Priority Issues:**

7. ✅ Unique index naming standardized - Changed all unique indexes to use ux_ prefix
8. ✅ Migration comments improved - Fixed typos and clarified descriptions

The schema is now **production-ready** with comprehensive performance optimizations. Remaining deferred items are application-specific or require business decisions (soft deletes, audit fields, VARCHAR sizing).
