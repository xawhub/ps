# Siebel Datafix & Data Loader Framework - Complete Technical Documentation & KN-221 Implementation

## 1. Introduction and Architecture Overview

This framework provides a modular Unix shell (`Bash` and `KornShell`) and Oracle SQL/PLSQL environment designed to automate data corrections (Datafixes), bulk data CSV loading via `SQL*Loader`, and Siebel Workflow/EIM executions in Siebel CRM.

### Core Architectural Principles:
* **Separation of Concerns:** Generic execution engines (`wp_generic_*.sh` and `call.sh`) are cleanly separated from task-specific configuration folders (deployment packages).
* **Compliance:** Direct DML modifications to Siebel base tables are avoided where possible by utilizing standard **Siebel EIM (Enterprise Integration Manager)** and **Role Refresh (TRA)** mechanisms to ensure data integrity and federation to downstream systems like **AMS**.
* **Auditability & Error Handling:** Strict exit codes (`exit 0` for clean termination or empty batch, `exit 1` for execution error, `exit 10` for data errors requiring manual verification) and automated log management with timestamps.

---

## 2. Directory Structure & Execution Flow

### 2.1. Target Server Directory Structure (Unix/Linux)
* `/u01/app/oracle/product/siebel/scripts/bin/` (`$RUNDIR`) – Main execution directory containing reusable generic scripts: `call.sh`, `wp_generic_datafix.sh`, `wp_generic_datafix_param.sh`, `wp_generic_data_loader.sh`, `wp_generic_workflow.sh`, `wp_generic_sql.sh`.
* `../conf/` – Configuration resources and GPG-encrypted credential files (`${SIE_REL_FIL}_pwd.par.gpg`).
* `../sql/` – SQL scripts responsible for populating EIM tables, verifying record counts (`wp_eim_count.sql`), gathering database statistics (`wp_run_stats.sql`), and post-EIM verification (`wp_count_*.sql`).
* `../ifb/` – `.ifb` configuration files controlling Siebel EIM tasks.
* `../data/` – Source CSV batch files used in bulk data loading processes.
* `../ctl/` – `.ctl` control files dedicated to Oracle `SQL*Loader`.
* `../log/` & `../log/history/` – Operational log directories and their archive.

### 2.2. Generic Control Scripts Summary

| Control Script | Operational Use Case | Input Arguments |
| :--- | :--- | :--- |
| **`wp_generic_datafix.sh`** | Standard datafix based on SQL EIM population and EIM job execution. | **6 parameters:** `sqlfilename`, `ifbfilename`, `countfilename`, `batch`, `processname`, `eimtablename` |
| **`wp_generic_datafix_param.sh`** | Extended datafix supporting up to 5 dynamic parameters passed to SQL scripts. | **11 parameters:** 6 standard parameters + `param1`, `param2`, `param3`, `param4`, `param5` |
| **`wp_generic_data_loader.sh`** | Bulk data loading from external CSV files using `sqlldr` and Post-SQL scripts. | **8 parameters:** `datafilename`, `ctlfilename`, `postfilename`, `ifbfilename`, `countfilename`, `batch`, `processname`, `eimtablename` |
| **`wp_generic_workflow.sh`** | Data correction executed via Siebel Workflow instead of EIM. | **5 parameters:** `processname`, `countfilename`, `searchspec`, `rowid`, `Functional process name` |

### 2.3. Detailed EIM Process Execution Flow
```
[Environment Initialization & GPG Password Decryption]
                       │
                       ▼
          [Populate EIM Table]
    (via SQL Insert or SQL*Loader + Post-SQL)
                       │
                       ▼
        [Batch Verification: wp_eim_count.sql]
                       │
             ┌─────────┴─────────┐
      (nCnt > 0)             (nCnt = 0)
             │                   │
             ▼                   ▼
[Gather DBMS Stats]      [Stop: Exit 0 (No Data)]
             │
             ▼
   [Execute Siebel EIM]
             │
             ▼
  [Verify EIM Results (count.sql)]
             │
             ├───────────────────┐
     (No Data Errors)    (Data Errors Found)
             │                   │
             ▼                   ▼
       [Exit 0: Success]   [Exit 10: Manual Check]
```

---

## 3. Identified Framework Risks & Technical Recommendations

1. **Inconsistent Database Credentials Across Generic Scripts:**
   * `wp_generic_datafix.sh` executes main and count SQL scripts using `Invoke_Sqlplus` (`$SAD_USR`), whereas `wp_generic_datafix_param.sh` uses `Invoke_Sqlplus_Siebel` (`SIEBEL`).
   * *Recommendation:* Always use `wp_generic_datafix_param.sh` (which runs as schema owner `SIEBEL`) or standardize `wp_generic_datafix.sh` to avoid privilege issues (`ORA-00942` / `ORA-01031`).
2. **Single-Batch Check Limitation (`wp_eim_count.sql`):**
   * The expression `export ibatch=$(echo $batch | cut -d'-' -f 1)` checks only the first batch number in a range (e.g., `2901900`). Ensure that the SQL population script always starts populating from the exact first batch number (`2901900 + FLOOR(ROWNUM / 5000)`).
3. **Direct Base Table Updates in Legacy Count Scripts:**
   * Legacy `wp_count_pds_flag.sql` contained direct `UPDATE` statements on `S_ORG_EXT_XM` and `S_ORG_EXT`. All base table modifications should be handled strictly via EIM and `wp_update_tra_status.sh`.
4. **Commented-Out Log Archiving:**
   * `#mv $logFilePath/*.log $logFilePath/history` is commented out across all generic scripts.

---

## 4. KN-221 Task Specific Implementation (PDS Flag & Entitlement Expiry)

### Business Problem & Refactoring Design
In the legacy `wp_update_pds_flag.sql` script, PDS Flag updates (`EIM_ACCOUNT`), Pricing Entitlement expirations (`EIM_ENTLMNT`), and Role Refresh staging (`EIM_DEFECT`) were tightly coupled inside a single PL/SQL loop. If a merchant did not have an active agreement in `SIEBEL.CX_AGREE_ORG`, `SELECT AGREE_ID INTO sAgreementId` threw a `NO_DATA_FOUND` exception, aborting the iteration and skipping the PDS Flag update and Role Refresh entirely.

To fulfill all Acceptance Criteria of **KN-221**, the solution is split into **two independent, renamed deployment packages**:

* **Package I (`wp_kn221_update_pds_flag`):**
  * Sets `X_WP_PRI_PDS_FLG = 'Y'` on every Group Merchant in the list and on every child PO in that Group Merchant's hierarchy at all levels (`START WITH ... CONNECT BY NOCYCLE PRIOR ROW_ID = PAR_OU_ID` and `MASTER_OU_ID`).
  * Merchants already set to `'Y'` are left unchanged and are **not** re-refreshed.
  * Populates `EIM_DEFECT` and calls `./wp_update_tra_status.sh` to perform Role Refresh (`TRA`) so changes federate to AMS.
  * Produces a detailed report (`wp_count_kn221_pds_flag.sql`) listing each party `UPDATED`, `SKIPPED`, and `FAILED` with the reason (including `PO NUMBER DOES NOT EXIST IN SIEBEL`).

* **Package II (`wp_kn221_expire_entitlement`):**
  * Evaluates `sNewDate := TRUNC(SYSDATE) - 1` once at run start.
  * Expires active Pricing Entitlements (`ENT.ENTL_TYPE_CD = 'PRICING'`) by setting `ORG_EFF_END_DT` to `SYSDATE - 1`.
  * Entitlements with `ValidTo` (`EFF_END_DT`) already in the past are left unchanged; merchants with no active Pricing Entitlement are gracefully skipped (`NO_DATA_FOUND` caught locally).
  * Populates `EIM_DEFECT` and calls `./wp_update_tra_status.sh` only for updated parties to federate changes to AMS.
  * Produces a detailed report (`wp_count_kn221_expire_entitlement.sql`) listing each party `UPDATED`, `SKIPPED`, and `FAILED` with the reason.

---

### 4.1. Package I: PDS Flag Update across Hierarchy

#### 1. Wrapper Script: `wp_kn221_update_pds_flag.sh`
```bash
#!/bin/bash
#######################################################################################
# SCRIPT NAME: wp_kn221_update_pds_flag.sh
# DESCRIPTION: KN-221 Part I - Update PDS Flag to Y on Group Merchants and all child POs
#              and trigger Role Refresh to AMS
#######################################################################################

cd $RUNDIR
chmod 750 wp_generic_datafix_param.sh
./wp_generic_datafix_param.sh "wp_kn221_update_pds_flag.sql" "wp_kn221_update_pds_flag.ifb" "wp_count_kn221_pds_flag.sql" 2901900-2901999 "wp_kn221_update_pds_flag" "EIM_ACCOUNT" $1
./wp_update_tra_status.sh EIM_DEFECT DEFECT_NUM 2901900 2901999
```

#### 2. SQL Procedure: `wp_kn221_update_pds_flag.sql`
```sql
SET SERVEROUTPUT ON
SET VERIFY OFF
WHENEVER SQLERROR EXIT FAILURE

DECLARE
    counter   NUMBER(9) := 0;
    CreatedBy VARCHAR2(15) := '&1';

    CURSOR curr_rec IS
        SELECT /*+ opt_param('hash_join_enabled','true') opt_param('optimizer_mode','all_rows') */
            ROWNUM                              AS ROW_ID,
            'FOR_IMPORT'                        AS IF_ROW_STAT,
            2901900 + FLOOR(ROWNUM / 5000)      AS IF_ROW_BATCH_NUM,
            orgparty.PARTY_UID                  AS PARTY_UID,
            orgparty.PARTY_TYPE_CD              AS PARTY_TYPE_CD,
            org.NAME                            AS NAME,
            org.LOC                             AS LOC,
            orgbu.ROW_ID                        AS ACCNT_BI,
            orgbu.NAME                          AS ACCNT_BU,
            org.X_WP_PRI_PDS_FLG                AS X_WP_PRI_PDS_FLG,
            org.ROW_ID                          AS PARTY_ID
        FROM SIEBEL.S_PARTY orgparty
        INNER JOIN SIEBEL.S_ORG_EXT org ON orgparty.ROW_ID = org.PAR_ROW_ID
        INNER JOIN SIEBEL.S_BU orgbu    ON org.BU_ID = orgbu.ROW_ID
        WHERE org.OU_TYPE_CD = 'Customer'
          -- Acceptance Criterion: Merchants already set to Y are left unchanged and not re-refreshed
          AND NVL(org.X_WP_PRI_PDS_FLG, 'N') <> 'Y'
          AND (
              -- Hierarchical traversal: Group Merchant + all child POs at all levels
              org.ROW_ID IN (
                  SELECT ROW_ID FROM SIEBEL.S_ORG_EXT
                  START WITH ROW_ID IN (
                      SELECT ROW_ID FROM SIEBEL.S_ORG_EXT 
                      WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = CreatedBy)
                      UNION
                      SELECT MASTER_OU_ID FROM SIEBEL.S_ORG_EXT 
                      WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = CreatedBy)
                  )
                  CONNECT BY NOCYCLE PRIOR ROW_ID = PAR_OU_ID
              )
              OR org.MASTER_OU_ID IN (
                  SELECT ROW_ID FROM SIEBEL.S_ORG_EXT 
                  WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = CreatedBy)
                  UNION
                  SELECT MASTER_OU_ID FROM SIEBEL.S_ORG_EXT 
                  WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = CreatedBy)
              )
          );

    reg_rec curr_rec%ROWTYPE;

BEGIN
    DELETE FROM SIEBEL.EIM_ACCOUNT WHERE IF_ROW_BATCH_NUM BETWEEN 2901900 AND 2901999;
    DELETE FROM SIEBEL.EIM_DEFECT  WHERE IF_ROW_BATCH_NUM BETWEEN 2901900 AND 2901999;
    COMMIT;

    FOR reg_rec IN curr_rec LOOP
        BEGIN
            -- 1. Stage PDS Flag = 'Y' update in EIM_ACCOUNT
            INSERT INTO SIEBEL.EIM_ACCOUNT (
                ROW_ID,
                IF_ROW_STAT,
                IF_ROW_BATCH_NUM,
                PARTY_UID,
                PARTY_TYPE_CD,
                NAME,
                LOC,
                ACCNT_BI,
                ACCNT_BU,
                X_WP_PRI_PDS_FLG
            ) VALUES (
                reg_rec.PARTY_ID,
                reg_rec.IF_ROW_STAT,
                reg_rec.IF_ROW_BATCH_NUM,
                reg_rec.PARTY_UID,
                reg_rec.PARTY_TYPE_CD,
                reg_rec.NAME,
                reg_rec.LOC,
                reg_rec.ACCNT_BI,
                reg_rec.ACCNT_BU,
                'Y'
            );

            -- 2. Stage updated Party in EIM_DEFECT for Role Refresh (wp_update_tra_status.sh)
            INSERT INTO SIEBEL.EIM_DEFECT (
                IF_ROW_BATCH_NUM,
                IF_ROW_STAT,
                ROW_ID,
                DEFECT_NUM
            ) VALUES (
                reg_rec.IF_ROW_BATCH_NUM,
                'IMPORTED',
                reg_rec.PARTY_ID,
                reg_rec.NAME
            );

            counter := counter + 1;
            IF counter = 5000 THEN
                COMMIT;
                counter := 0;
            END IF;

        EXCEPTION
            WHEN OTHERS THEN
                UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
                SET SDM_MIGRATED_STATUS = 'FAILURE',
                    SDM_ERROR_MSG = 'SQL_PROCESSING_ERROR',
                    LAST_UPD = SYSTIMESTAMP
                WHERE PARTY_ID = reg_rec.NAME
                  AND CREATED_BY = CreatedBy;
                CONTINUE;
        END;
    END LOOP;
    COMMIT;

    -- Mark any input records that do not exist in Siebel S_ORG_EXT as FAILURE
    UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
    SET SDM_MIGRATED_STATUS = 'FAILURE',
        SDM_ERROR_MSG = 'PO NUMBER DOES NOT EXIST IN SIEBEL',
        LAST_UPD = SYSTIMESTAMP
    WHERE PARTY_ID NOT IN (
        SELECT A.PARTY_ID 
        FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK A, SIEBEL.S_ORG_EXT B 
        WHERE A.PARTY_ID = B.NAME
    )
    AND CREATED_BY = CreatedBy;
    COMMIT;
END;
/
EXIT;
```

#### 3. EIM Control File: `wp_kn221_update_pds_flag.ifb`
```ini
[Siebel Interface Manager]
    LOG TRANSACTIONS TO FILE = FALSE
    USING SYNONYMS = FALSE
    USE INDEX HINTS = TRUE
    USE ESSENTIAL INDEX HINTS = TRUE
    PROCESS = IMPORT_PDS_FLAG

[IMPORT_PDS_FLAG]
    TYPE = IMPORT
    BATCH = $BatchRange
    TABLE = EIM_ACCOUNT
    INSERT ROWS = S_PARTY, FALSE
    UPDATE ROWS = S_PARTY, FALSE
    INSERT ROWS = S_ORG_EXT, FALSE
    UPDATE ROWS = S_ORG_EXT, TRUE
    ONLY BASE TABLES = S_PARTY, S_ORG_EXT
    ONLY BASE COLUMNS = S_PARTY.PARTY_UID, \
                        S_PARTY.PARTY_TYPE_CD, \
                        S_ORG_EXT.NAME, \
                        S_ORG_EXT.LOC, \
                        S_ORG_EXT.BU_ID, \
                        S_ORG_EXT.X_WP_PRI_PDS_FLG
```

#### 4. Verification & Count SQL: `wp_count_kn221_pds_flag.sql`
```sql
SET VERIFY OFF
SET HEADING ON
SET PAGESIZE 50000
SET FEEDBACK ON
SET LINESIZE 200
WHENEVER SQLERROR EXIT FAILURE

DEFINE CreatedBy='&1'

-- 1. Summary of EIM statuses
SELECT IF_ROW_STAT, COUNT(1) AS TOTAL_RECORDS
FROM SIEBEL.EIM_ACCOUNT
WHERE IF_ROW_BATCH_NUM BETWEEN 2901900 AND 2901999
GROUP BY IF_ROW_STAT;

-- 2. Detailed Audit Report required by Jira KN-221: UPDATED / FAILED / SKIPPED
SELECT 
    NAME AS PARTY_ID,
    PARTY_UID,
    CASE 
        WHEN IF_ROW_STAT = 'IMPORTED' THEN 'UPDATED'
        ELSE 'FAILED'
    END AS ACTION_STATUS,
    IF_ROW_STAT AS REASON
FROM SIEBEL.EIM_ACCOUNT
WHERE IF_ROW_BATCH_NUM BETWEEN 2901900 AND 2901999
UNION ALL
SELECT 
    org.NAME AS PARTY_ID,
    orgparty.PARTY_UID,
    'SKIPPED' AS ACTION_STATUS,
    'PDS Flag already set to Y' AS REASON
FROM SIEBEL.S_ORG_EXT org
INNER JOIN SIEBEL.S_PARTY orgparty ON orgparty.ROW_ID = org.PAR_ROW_ID
WHERE org.OU_TYPE_CD = 'Customer'
  AND org.X_WP_PRI_PDS_FLG = 'Y'
  AND (
      org.ROW_ID IN (
          SELECT ROW_ID FROM SIEBEL.S_ORG_EXT
          START WITH ROW_ID IN (
              SELECT ROW_ID FROM SIEBEL.S_ORG_EXT 
              WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = '&CreatedBy')
              UNION
              SELECT MASTER_OU_ID FROM SIEBEL.S_ORG_EXT 
              WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = '&CreatedBy')
          )
          CONNECT BY NOCYCLE PRIOR ROW_ID = PAR_OU_ID
      )
      OR org.MASTER_OU_ID IN (
          SELECT ROW_ID FROM SIEBEL.S_ORG_EXT 
          WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = '&CreatedBy')
          UNION
          SELECT MASTER_OU_ID FROM SIEBEL.S_ORG_EXT 
          WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = '&CreatedBy')
      )
  )
  AND org.ROW_ID NOT IN (
      SELECT ROW_ID FROM SIEBEL.EIM_ACCOUNT WHERE IF_ROW_BATCH_NUM BETWEEN 2901900 AND 2901999
  )
UNION ALL
SELECT 
    PARTY_ID,
    'N/A' AS PARTY_UID,
    'FAILED' AS ACTION_STATUS,
    SDM_ERROR_MSG AS REASON
FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
WHERE CREATED_BY = '&CreatedBy'
  AND SDM_MIGRATED_STATUS = 'FAILURE';

-- 3. Update status in migration tracking table for successful imports
UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
SET SDM_MIGRATED_STATUS = 'COMPLETED', LAST_UPD = SYSTIMESTAMP
WHERE CREATED_BY = '&CreatedBy'
  AND PARTY_ID IN (
      SELECT NAME FROM SIEBEL.EIM_ACCOUNT 
      WHERE IF_ROW_BATCH_NUM BETWEEN 2901900 AND 2901999 
        AND IF_ROW_STAT = 'IMPORTED'
  );
COMMIT;

-- 4. Control query for wp_generic_datafix_param.sh
-- (Must return "no rows selected" when all EIM rows succeeded as IMPORTED)
SELECT ROW_ID, NAME, IF_ROW_STAT 
FROM SIEBEL.EIM_ACCOUNT 
WHERE IF_ROW_BATCH_NUM BETWEEN 2901900 AND 2901999 
  AND IF_ROW_STAT <> 'IMPORTED';

EXIT;
```

---

### 4.2. Package II: Pricing Entitlement Expiry (`SYSDATE - 1`)

#### 1. Wrapper Script: `wp_kn221_expire_entitlement.sh`
```bash
#!/bin/bash
#######################################################################################
# SCRIPT NAME: wp_kn221_expire_entitlement.sh
# DESCRIPTION: KN-221 Part II - Expire Pricing Entitlement with SYSDATE - 1
#              and trigger Role Refresh to AMS
#######################################################################################

cd $RUNDIR
chmod 750 wp_generic_datafix_param.sh
./wp_generic_datafix_param.sh "wp_kn221_expire_entitlement.sql" "wp_kn221_expire_entitlement.ifb" "wp_count_kn221_expire_entitlement.sql" 2902000-2902999 "wp_kn221_expire_entitlement" "EIM_ENTLMNT" $1
./wp_update_tra_status.sh EIM_DEFECT DEFECT_NUM 2902000 2902999
```

#### 2. SQL Procedure: `wp_kn221_expire_entitlement.sql`
```sql
SET SERVEROUTPUT ON
SET VERIFY OFF
WHENEVER SQLERROR EXIT FAILURE

DECLARE
    counter      NUMBER(9) := 0;
    CreatedBy    VARCHAR2(15) := '&1';
    sAgreementId VARCHAR2(15);
    sPartyId     VARCHAR2(15);
    -- Acceptance Criterion: Evaluated once at run start (previous calendar day SYSDATE - 1)
    sDate        DATE := TRUNC(SYSDATE);
    sNewDate     DATE := TRUNC(SYSDATE) - 1;
    v_rows_ins   NUMBER := 0;

    CURSOR curr_rec IS
        SELECT /*+ opt_param('hash_join_enabled','true') opt_param('optimizer_mode','all_rows') */
            ROWNUM                              AS ROW_ID,
            'FOR_IMPORT'                        AS IF_ROW_STAT,
            2902000 + FLOOR(ROWNUM / 5000)      AS IF_ROW_BATCH_NUM,
            orgparty.PARTY_UID                  AS PARTY_UID,
            orgparty.PARTY_TYPE_CD              AS PARTY_TYPE_CD,
            org.NAME                            AS NAME,
            org.LOC                             AS LOC,
            orgbu.ROW_ID                        AS ACCNT_BI,
            orgbu.NAME                          AS ACCNT_BU,
            org.ROW_ID                          AS PARTY_ID
        FROM SIEBEL.S_PARTY orgparty
        INNER JOIN SIEBEL.S_ORG_EXT org ON orgparty.ROW_ID = org.PAR_ROW_ID
        INNER JOIN SIEBEL.S_BU orgbu    ON org.BU_ID = orgbu.ROW_ID
        WHERE org.OU_TYPE_CD = 'Customer'
          AND org.NAME IN (
              SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = CreatedBy
          );

    reg_rec curr_rec%ROWTYPE;

BEGIN
    DELETE FROM SIEBEL.EIM_ENTLMNT WHERE IF_ROW_BATCH_NUM BETWEEN 2902000 AND 2902999;
    DELETE FROM SIEBEL.EIM_DEFECT  WHERE IF_ROW_BATCH_NUM BETWEEN 2902000 AND 2902999;
    COMMIT;

    FOR reg_rec IN curr_rec LOOP
        BEGIN
            sPartyId := reg_rec.PARTY_ID;
            sAgreementId := NULL;

            -- Acceptance Criterion: Merchants with no active Pricing Entitlement are skipped
            BEGIN
                SELECT AGREE_ID INTO sAgreementId 
                FROM SIEBEL.CX_AGREE_ORG 
                WHERE ACCNT_ID = sPartyId 
                  AND (END_DT IS NULL OR END_DT >= sDate);
            EXCEPTION
                WHEN NO_DATA_FOUND THEN
                    sAgreementId := NULL;
            END;

            IF sAgreementId IS NOT NULL THEN
                INSERT INTO SIEBEL.EIM_ENTLMNT (
                    IF_ROW_BATCH_NUM,
                    IF_ROW_STAT,
                    ROW_ID,
                    ORG_ACCNT_BI,
                    ORG_ACCNT_BU,
                    ORG_ACCNT_LOC,
                    ORG_ACCNT_NAME,
                    PAR_AGREE_ACC_BI,
                    PAR_AGREE_ACC_BU,
                    PAR_AGREE_ACC_LOC,
                    PAR_AGREE_ACC_NAME,
                    PAR_AGREE_BI,
                    PAR_AGREE_BU,
                    PAR_AGREE_NAME,
                    PAR_AGREE_REV_NUM,
                    ENTL_NAME,
                    ORG_EFF_END_DT
                )
                SELECT
                    reg_rec.IF_ROW_BATCH_NUM,
                    reg_rec.IF_ROW_STAT,
                    ENT_ORG.ROW_ID,
                    '0-R9NH',
                    'Default Organization',
                    PARTY.LOC,
                    PARTY.NAME,
                    '0-R9NH',
                    'Default Organization',
                    PARTY.LOC,
                    PARTY.NAME,
                    '0-R9NH',
                    'Default Organization',
                    AGREEMENT.NAME,
                    AGREEMENT.REV_NUM,
                    ENT.NAME,
                    CASE
                        WHEN (ENT_ORG.EFF_START_DT >= sDate) THEN ENT_ORG.EFF_START_DT
                        WHEN (ENT_ORG.EFF_START_DT < sDate)  THEN sNewDate
                    END
                FROM SIEBEL.S_DOC_AGREE AGREEMENT,
                     SIEBEL.S_ENTLMNT_ORG ENT_ORG,
                     SIEBEL.S_ENTLMNT ENT,
                     SIEBEL.S_ORG_EXT PARTY
                WHERE AGREEMENT.ROW_ID = ENT.PAR_AGREE_ID
                  AND ENT.ROW_ID = ENT_ORG.AGREE_ID
                  AND ENT_ORG.OU_ID = PARTY.ROW_ID
                  AND (AGREEMENT.EFF_END_DT IS NULL OR AGREEMENT.EFF_END_DT >= sDate)
                  -- Acceptance Criterion: Entitlements with ValidTo already in the past are left unchanged
                  AND (ENT_ORG.EFF_END_DT IS NULL OR ENT_ORG.EFF_END_DT >= sDate)
                  AND ENT.ENTL_TYPE_CD = 'PRICING'
                  AND AGREEMENT.ROW_ID = sAgreementId;

                v_rows_ins := SQL%ROWCOUNT;

                -- Perform Role Refresh only on targeted parties where an active Pricing Entitlement was updated
                IF v_rows_ins > 0 THEN
                    INSERT INTO SIEBEL.EIM_DEFECT (
                        IF_ROW_BATCH_NUM,
                        IF_ROW_STAT,
                        ROW_ID,
                        DEFECT_NUM
                    ) VALUES (
                        reg_rec.IF_ROW_BATCH_NUM,
                        'IMPORTED',
                        reg_rec.PARTY_ID,
                        reg_rec.NAME
                    );
                END IF;
            END IF;

            counter := counter + 1;
            IF counter = 5000 THEN
                COMMIT;
                counter := 0;
            END IF;

        EXCEPTION
            WHEN OTHERS THEN
                UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
                SET SDM_MIGRATED_STATUS = 'FAILURE',
                    SDM_ERROR_MSG = 'SQL_PROCESSING_ERROR',
                    LAST_UPD = SYSTIMESTAMP
                WHERE PARTY_ID = reg_rec.NAME
                  AND CREATED_BY = CreatedBy;
                CONTINUE;
        END;
    END LOOP;
    COMMIT;

    -- Mark any input records that do not exist in Siebel S_ORG_EXT as FAILURE
    UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
    SET SDM_MIGRATED_STATUS = 'FAILURE',
        SDM_ERROR_MSG = 'PO NUMBER DOES NOT EXIST IN SIEBEL',
        LAST_UPD = SYSTIMESTAMP
    WHERE PARTY_ID NOT IN (
        SELECT A.PARTY_ID 
        FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK A, SIEBEL.S_ORG_EXT B 
        WHERE A.PARTY_ID = B.NAME
    )
    AND CREATED_BY = CreatedBy;
    COMMIT;
END;
/
EXIT;
```

#### 3. EIM Control File: `wp_kn221_expire_entitlement.ifb`
```ini
[Siebel Interface Manager]
    LOG TRANSACTIONS TO FILE = FALSE
    USING SYNONYMS = FALSE
    USE INDEX HINTS = TRUE
    USE ESSENTIAL INDEX HINTS = TRUE
    PROCESS = IMPORT_ENTL

[IMPORT_ENTL]
    TYPE = IMPORT
    BATCH = $BatchRange
    TABLE = EIM_ENTLMNT
    INSERT ROWS = S_PARTY, FALSE
    UPDATE ROWS = S_PARTY, FALSE
    INSERT ROWS = S_ENTLMNT_ORG, FALSE
    UPDATE ROWS = S_ENTLMNT_ORG, TRUE
    ONLY BASE TABLES = S_PARTY, S_ENTLMNT_ORG
    ONLY BASE COLUMNS = S_PARTY.PARTY_UID, \
                        S_PARTY.PARTY_TYPE_CD, \
                        S_ENTLMNT_ORG.ORG_EFF_END_DT
```

#### 4. Verification & Count SQL: `wp_count_kn221_expire_entitlement.sql`
```sql
SET VERIFY OFF
SET HEADING ON
SET PAGESIZE 50000
SET FEEDBACK ON
SET LINESIZE 200
WHENEVER SQLERROR EXIT FAILURE

DEFINE CreatedBy='&1'

-- 1. Summary of EIM statuses
SELECT IF_ROW_STAT, COUNT(1) AS TOTAL_RECORDS
FROM SIEBEL.EIM_ENTLMNT
WHERE IF_ROW_BATCH_NUM BETWEEN 2902000 AND 2902999
GROUP BY IF_ROW_STAT;

-- 2. Detailed Audit Report required by Jira KN-221: UPDATED / FAILED / SKIPPED
SELECT 
    ORG_ACCNT_NAME AS PARTY_ID,
    ENTL_NAME,
    TO_CHAR(ORG_EFF_END_DT, 'DD/MM/YYYY') AS NEW_VALID_TO,
    CASE 
        WHEN IF_ROW_STAT = 'IMPORTED' THEN 'UPDATED'
        ELSE 'FAILED'
    END AS ACTION_STATUS,
    IF_ROW_STAT AS REASON
FROM SIEBEL.EIM_ENTLMNT
WHERE IF_ROW_BATCH_NUM BETWEEN 2902000 AND 2902999
UNION ALL
SELECT 
    pmp.PARTY_ID,
    'N/A' AS ENTL_NAME,
    'N/A' AS NEW_VALID_TO,
    'SKIPPED' AS ACTION_STATUS,
    'No active Pricing Entitlement or ValidTo already in the past' AS REASON
FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK pmp
WHERE pmp.CREATED_BY = '&CreatedBy'
  AND NVL(pmp.SDM_MIGRATED_STATUS, 'NEW') <> 'FAILURE'
  AND pmp.PARTY_ID NOT IN (
      SELECT ORG_ACCNT_NAME FROM SIEBEL.EIM_ENTLMNT 
      WHERE IF_ROW_BATCH_NUM BETWEEN 2902000 AND 2902999
  )
UNION ALL
SELECT 
    pmp.PARTY_ID,
    'N/A' AS ENTL_NAME,
    'N/A' AS NEW_VALID_TO,
    'FAILED' AS ACTION_STATUS,
    pmp.SDM_ERROR_MSG AS REASON
FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK pmp
WHERE pmp.CREATED_BY = '&CreatedBy'
  AND pmp.SDM_MIGRATED_STATUS = 'FAILURE';

-- 3. Update status in migration tracking table
UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
SET SDM_MIGRATED_STATUS = 'COMPLETED', LAST_UPD = SYSTIMESTAMP
WHERE CREATED_BY = '&CreatedBy'
  AND PARTY_ID IN (
      SELECT ORG_ACCNT_NAME FROM SIEBEL.EIM_ENTLMNT 
      WHERE IF_ROW_BATCH_NUM BETWEEN 2902000 AND 2902999 
        AND IF_ROW_STAT = 'IMPORTED'
  );
COMMIT;

-- 4. Control query for wp_generic_datafix_param.sh
-- (Must return "no rows selected" when no EIM errors occurred)
SELECT ROW_ID, ORG_ACCNT_NAME, IF_ROW_STAT 
FROM SIEBEL.EIM_ENTLMNT 
WHERE IF_ROW_BATCH_NUM BETWEEN 2902000 AND 2902999 
  AND IF_ROW_STAT <> 'IMPORTED';

EXIT;
```

---

## 5. Complete Core Framework & Legacy Reference Scripts (All Screenshots)

### 5.1. `README.txt`
```text
1. Move the Unix scripts in the root folder (wp_generic_datafix.sh, call.sh) to /u01/app/oracle/product/siebel/scripts/bin

-- Other data fixes rely on the following generic shell scripts:
wp_generic_datafix.sh
call.sh
```

### 5.2. `call.sh` (Complete Core Function Library)
```bash
#!/usr/bin/ksh
######################################################################################
# SCRIPT NAME: call
# AUTHOR:      ARC
# DATE:        10/10/17
#
# DESCRIPTION: General. It contains all function that need to be used to call EIM process, Sql plus and Sql ldr.
#              It can be improved with functions for calling WF, Batch process...
# PARAMETERS:  None
#
# HISTORY:
# Date       Description
# 03/08/17   Initial version
# 25/01/17   Added workflow
# 13/03/26   [Vinay Chopra] - Updated Workflow_Process to submit kafka workflow in async
######################################################################################

# Decrypts the siebel and sadmin password from the encrypted .gpg files
decrypt_password_file () {
  echo "Decrypting password file ..."
  [ -f ~/gpg_passphrase.txt ] || { echo "Failed to locate gpg_passphrase.txt file"; exit 1; }

  export SAD_PWD=`cat ~/gpg_passphrase.txt | gpg --batch --yes --passphrase-fd 0 -d ${SCRIPTS_HOME}/conf/${SIE_REL_FIL}_pwd.par.gpg | grep SAD_PWD | tail -1 | tr -d '()\/' | cut -d= -f2 | cut -d" " -f1`
  [ -z $SAD_PWD ] && { echo "SADMIN password not set... Exiting script..."; exit 1; } || echo "SADMIN password set"

  export SIEB_PWD=`cat ~/gpg_passphrase.txt | gpg --batch --yes --passphrase-fd 0 -d ${SCRIPTS_HOME}/conf/${SIE_REL_FIL}_pwd.par.gpg | grep SIEB_PWD | tail -1 | tr -d '()\/' | cut -d= -f2 | cut -d" " -f1`
  [ -z $SIEB_PWD ] && { echo "SIEBEL password not set... Exiting script..."; exit 1; } || echo "SIEBEL password set"
}

# Load Siebel tables from EIM tables
EIM_Process ()
{
  ifbFileName=$1
  externalLogFilePath=$2
  externalLogFileName=$3
  InitialBatch=$4
  timeStamp3=$5

  internalLogFileName=$1_$timeStamp3.log

  Log $externalLogFilePath $externalLogFileName "\t\tRunning EIM task"
  Log $externalLogFilePath $externalLogFileName "\t\t\tIFB: $ifbFileName"
  Log $externalLogFilePath $externalLogFileName "\t\t\tBatch: $InitialBatch"
  Log $externalLogFilePath $externalLogFileName "\t\t\tLOG: $internalLogFileName"

  ${SIEBEL_ROOT}/bin/srvrmgr /g $SIE_GATE /e $SIE_ENT_SRVR /s $WP_SIEBEL_SERVER /u $SAD_USR /p $SAD_PWD /c "run task for comp EIM with config=$ifbFileName, ExtendedParams=\"batchRange=$InitialBatch\", errorflags=1, traceflags=1, sqlflags=8" >> $externalLogFilePath/$internalLogFileName

  result=`grep "completed successfully" $externalLogFilePath/$internalLogFileName`
  if [ "$result" != "" ]
  then
    intResult=0
  else
    intResult=1
  fi

  return $intResult
}

Invoke_Sqlplus ()
{
  sqlFilePath=$1
  sqlFileName=$2
  externalLogFilePath=$3
  externalLogFileName=$4
  timeStamp3=$5
  param1=$6
  param2=$7
  param3=$8
  param4=$9
  param5=${10}

  internalLogFileName=$2_$timeStamp3.log

  Log $externalLogFilePath $externalLogFileName "\t\tInvoking SQLPlus"
  Log $externalLogFilePath $externalLogFileName "\t\t\tSQL: $sqlFileName"
  Log $externalLogFilePath $externalLogFileName "\t\t\tLOG: $internalLogFileName"

  sqlplus $SAD_USR/$SAD_PWD@$SIE_CONN_DB @$sqlFilePath/$sqlFileName $param1 $param2 $param3 $param4 $param5 >> $externalLogFilePath/$internalLogFileName

  error=`cat $externalLogFilePath/$internalLogFileName | grep -i "ERROR" | wc -l`

  if [ $error -ne 0 ]
  then
    intResult=1
    Log $externalLogFilePath $externalLogFileName "\tExecuted with errors. See $internalLogFileName for details"
  else
    intResult=0
  fi

  return $intResult
}

Invoke_Sqlplus_Siebel ()
{
  sqlFilePath=$1
  sqlFileName=$2
  externalLogFilePath=$3
  externalLogFileName=$4
  timeStamp3=$5
  param1=$6
  param2=$7
  param3=$8
  param4=$9
  param5=${10}

  internalLogFileName=$2_$timeStamp3.log

  Log $externalLogFilePath $externalLogFileName "\t\tInvoking SQLPlus"
  Log $externalLogFilePath $externalLogFileName "\t\t\tSQL: $sqlFileName"
  Log $externalLogFilePath $externalLogFileName "\t\t\tLOG: $internalLogFileName"

  sqlplus SIEBEL/$SIEB_PWD@$SIE_CONN_DB @$sqlFilePath/$sqlFileName $param1 $param2 $param3 $param4 $param5 >> $externalLogFilePath/$internalLogFileName

  error=`cat $externalLogFilePath/$internalLogFileName | grep -i "ERROR" | wc -l`

  if [ $error -ne 0 ]
  then
    intResult=1
    Log $externalLogFilePath $externalLogFileName "\tExecuted with errors. See $internalLogFileName for details"
  else
    intResult=0
  fi

  return $intResult
}

Log ()
{
  externalLogFilePath="$1"
  externalLogFileName="$2"
  message="$3"

  timeStamp=`date "+%d/%m/%Y %H:%M:%S"`
  echo "${timeStamp} ${message}" | tee -a "$externalLogFilePath/$externalLogFileName"
}

Invoke_SqlLoader ()
{
  dataFilePath=$1
  dataFileName=$2
  ctlFilePath=$3
  ctlFileName=$4
  entityName=$(cut -d '.' -f 1 <<< $ctlFileName)

  externalLogFilePath=$5
  externalLogFileName=$6
  timeStamp3=$7

  internalLogFileName=$2_$timeStamp3.log
  sqlLoaderLogFileName=$2_$timeStamp3"_sqlldr.log"

  Log $externalLogFilePath $externalLogFileName "\t\tInvoking SQLloader"
  Log $externalLogFilePath $externalLogFileName "\t\t\tData: $dataFileName"
  Log $externalLogFilePath $externalLogFileName "\t\t\tCTL: $ctlFileName"
  Log $externalLogFilePath $externalLogFileName "\t\t\tLOG: $internalLogFileName"

  sqlldr SIEBEL/$SIEB_PWD@$SIE_CONN_DB control=$ctlFilePath/$ctlFileName data=$dataFilePath/$dataFileName discard=$dataFilePath/$entityName.dsc bad=$dataFilePath/$entityName.bad log=$externalLogFilePath/$sqlLoaderLogFileName errors=0 >> $externalLogFilePath/$internalLogFileName

  error=`cat $externalLogFilePath/$internalLogFileName | grep -i "ERROR" | wc -l`
  error2=`cat $ctlFilePath/$entityName.bad | grep -i "ERROR" | wc -l`

  if [ $? -ne 0 ]
  then
    intResult=1
    Log $externalLogFilePath $externalLogFileName "\tExecuted with errors. See $externalLogFilePath/$sqlLoaderLogFileName for details"
  else
    intResult=0
  fi

  return $intResult
}

Workflow_Process ()
{
  processname=$1
  searchspec=$2
  rowid=$3
}
```

### 5.3. `wp_generic_datafix.sh` (Standard 6-Parameter Runner)
```bash
#!/bin/bash
typeset RUNDIR=$(dirname ${BASH_SOURCE})
RUNDIR=$(cd $RUNDIR ; pwd)

. /home/orafmw/env.par
. ./call.sh

decrypt_password_file
rm -f $RUNDIR/../conf/${SIE_REL_FIL}_pwd.par
. ${SIEBEL_ROOT}/bin/../siebenv.sh

logFilePath=$RUNDIR/../log
sqlFilePath=$RUNDIR/../sql
timeStamp2=`date "+%d%m%Y%H%M%S"`
sqlfilename=$1
ifbfilename=$2
countfilename=$3
batch=$4
processname=$5
eimtablename=$6

logFileName="$processname"_"$timeStamp2".log
echo $logFileName

Log $logFilePath $logFileName "Script Started"
Log $logFilePath $logFileName "Log directory: '$logFilePath'"

Invoke_Sqlplus $sqlFilePath $sqlfilename $logFilePath $logFileName $timeStamp2
if [ $? -ne 0 ]
then
  Log $logFilePath $logFileName "Error running Inserting EIM tables for \"$processname\". Please check the log"
else
  export ibatch=`echo $batch | cut -d'-' -f 1`
  Invoke_Sqlplus_Siebel $sqlFilePath "wp_eim_count.sql" $logFilePath $logFileName $timeStamp2 $eimtablename $ibatch
  if [ $? -ne 0 ]
  then
    Log $logFilePath $logFileName "No EIM rows to Process - exiting. Please check the log."
    exit 0
  else
    Log $logFilePath $logFileName "DB Stats starting.."
    Invoke_Sqlplus_Siebel $sqlFilePath "wp_run_stats.sql" $logFilePath $logFileName $timeStamp2 SIEBEL $eimtablename

    Log $logFilePath $logFileName "EIM job starting.."
    EIM_Process $ifbfilename $logFilePath $logFileName $batch $timeStamp2
    if [ $? -ne 0 ]
    then
      Log $logFilePath $logFileName "Error running the EIM job for \"$processname\". Please check the log"
    else
      Invoke_Sqlplus $sqlFilePath $countfilename $logFilePath $logFileName $timeStamp2
    fi
  fi
fi

if [ $? -ne 0 ]
then
  Log $logFilePath $logFileName "Error running \"$processname\". Please check the logs"
  exit 1
else
  countLog="$countfilename"_"$timeStamp2".log
  if [ -f $logFilePath/$countLog ]
  then
    passString=`grep "no rows selected" $logFilePath/$countLog`
    if [ "$passString" != "" ]
    then
      failCount=0
    else
      failCount=1
    fi
  else
    Log $logFilePath $logFileName "\tExecuted with errors. Please check the logs"
    exit 1
  fi

  if [ $failCount -ne 0 ]
  then
    Log $logFilePath $logFileName "\tProcess completed with data errors, manual verification required. Please check ${countLog} for details"
    exit 10
  else
    Log $logFilePath $logFileName "Script completed"
  fi
fi
```

### 5.4. `wp_generic_datafix_param.sh` (Parameterized 11-Parameter Runner)
```bash
#!/bin/bash
typeset RUNDIR=$(dirname ${BASH_SOURCE})
RUNDIR=$(cd $RUNDIR ; pwd)

. /home/orafmw/env.par
. ./call.sh

decrypt_password_file
rm -f $RUNDIR/../conf/${SIE_REL_FIL}_pwd.par
. ${SIEBEL_ROOT}/bin/../siebenv.sh

logFilePath=$RUNDIR/../log
sqlFilePath=$RUNDIR/../sql
timeStamp2=`date "+%d%m%Y%H%M%S"`
sqlfilename=$1
ifbfilename=$2
countfilename=$3
batch=$4
processname=$5
eimtablename=$6
param1=$7; param2=$8; param3=$9; param4=${10}; param5=${11}

logFileName="$processname"_"$timeStamp2".log
Log $logFilePath $logFileName "Script Started"
Log $logFilePath $logFileName "Log directory: '$logFilePath'"

Invoke_Sqlplus_Siebel $sqlFilePath $sqlfilename $logFilePath $logFileName $timeStamp2 $param1 $param2 $param3 $param4 $param5
if [ $? -ne 0 ]; then
  Log $logFilePath $logFileName "Error running Inserting EIM tables for \"$processname\". Please check the log"
else
  export ibatch=`echo $batch | cut -d'-' -f 1`
  Invoke_Sqlplus_Siebel $sqlFilePath "wp_eim_count.sql" $logFilePath $logFileName $timeStamp2 $eimtablename $ibatch
  if [ $? -ne 0 ]; then
    Log $logFilePath $logFileName "No EIM rows to Process - exiting. Please check the log."
    exit 0
  else
    Log $logFilePath $logFileName "DB Stats starting.."
    Invoke_Sqlplus_Siebel $sqlFilePath "wp_run_stats.sql" $logFilePath $logFileName $timeStamp2 SIEBEL $eimtablename

    Log $logFilePath $logFileName "EIM job starting.."
    EIM_Process $ifbfilename $logFilePath $logFileName $batch $timeStamp2
    if [ $? -ne 0 ]; then
      Log $logFilePath $logFileName "Error running the EIM job for \"$processname\". Please check the log"
    else
      Invoke_Sqlplus_Siebel $sqlFilePath $countfilename $logFilePath $logFileName $timeStamp2 $param1 $param2 $param3 $param4 $param5
    fi
  fi
fi

if [ $? -ne 0 ]; then
  Log $logFilePath $logFileName "Error running \"$processname\". Please check the logs"
  exit 1
else
  countLog="$countfilename"_"$timeStamp2".log
  if [ -f $logFilePath/$countLog ]; then
    passString=`grep "no rows selected" $logFilePath/$countLog`
    if [ "$passString" != "" ]; then
      failCount=0
    else
      failCount=1
    fi
  else
    Log $logFilePath $logFileName "\tExecuted with errors. Please check the logs"
    exit 1
  fi

  if [ $failCount -ne 0 ]; then
    Log $logFilePath $logFileName "\tProcess completed with data errors, manual verification required. Please check ${countLog} for details"
    exit 10
  else
    Log $logFilePath $logFileName "Script completed"
  fi
fi
```

### 5.5. `wp_generic_data_loader.sh` (CSV SQL*Loader Runner)
```bash
#!/bin/bash
typeset RUNDIR=$(dirname ${BASH_SOURCE})
RUNDIR=$(cd $RUNDIR ; pwd)

. /home/orafmw/env.par
. ./call.sh

decrypt_password_file
rm -f $RUNDIR/../conf/${SIE_REL_FIL}_pwd.par
. ${SIEBEL_ROOT}/bin/../siebenv.sh

logFilePath=$RUNDIR/../log
dataFilePath=$RUNDIR/../data
ctlFilePath=$RUNDIR/../ctl
sqlFilePath=$RUNDIR/../sql
timeStamp2=`date "+%d%m%Y%H%M%S"`
datafilename=$1
ctlfilename=$2
postfilename=$3
ifbfilename=$4
countfilename=$5
batch=$6
processname=$7
eimtablename=$8

logFileName="$processname"_"$timeStamp2".log
echo $logFileName

Log $logFilePath $logFileName "Script Started"
Log $logFilePath $logFileName "Log directory: '$logFilePath'"

Invoke_SqlLoader $dataFilePath $datafilename $ctlFilePath $ctlfilename $logFilePath $logFileName $timeStamp2
if [ $? -ne 0 ]
then
  Log $logFilePath $logFileName "Error running SQL Loader process to load EIM table. Please check the log"
  exit 1
else
  Log $logFilePath $logFileName "SQL Loader Completed"
  Invoke_Sqlplus_Siebel $sqlFilePath $postfilename $logFilePath $logFileName $timeStamp2 SIEBEL $eimtablename
  if [ $? -ne 0 ]
  then
    Log $logFilePath $logFileName "Error running Post SQL Loader SQL. Please check the log"
    exit 1
  else
    export ibatch=`echo $batch | cut -d'-' -f 1`
    Invoke_Sqlplus_Siebel $sqlFilePath "wp_eim_count.sql" $logFilePath $logFileName $timeStamp2 $eimtablename $ibatch
    if [ $? -ne 0 ]
    then
      Log $logFilePath $logFileName "No EIM rows to Process - exiting. Please check the log."
      exit 0
    else
      Log $logFilePath $logFileName "DB Stats starting.."
      Invoke_Sqlplus_Siebel $sqlFilePath "wp_run_stats.sql" $logFilePath $logFileName $timeStamp2 SIEBEL $eimtablename

      Log $logFilePath $logFileName "EIM job starting.."
      EIM_Process $ifbfilename $logFilePath $logFileName $batch $timeStamp2
      if [ $? -ne 0 ]
      then
        Log $logFilePath $logFileName "Error running the EIM job for \"$processname\". Please check the log"
      else
        Invoke_Sqlplus $sqlFilePath $countfilename $logFilePath $logFileName $timeStamp2
      fi
    fi
  fi
fi

if [ $? -ne 0 ]
then
  Log $logFilePath $logFileName "Error running \"$processname\". Please check the logs"
  exit 1
else
  countLog="$countfilename"_"$timeStamp2".log
  if [ -f $logFilePath/$countLog ]
  then
    passString=`grep "no rows selected" $logFilePath/$countLog`
    if [ "$passString" != "" ]
    then
      failCount=0
    else
      failCount=1
    fi
  else
    Log $logFilePath $logFileName "\tExecuted with errors. Please check the logs"
    exit 1
  fi

  if [ $failCount -ne 0 ]
  then
    Log $logFilePath $logFileName "\tProcess completed with data errors, manual verification required. Please check ${countLog} for details"
    exit 10
  else
    Log $logFilePath $logFileName "Script completed"
  fi
fi
```

### 5.6. `wp_generic_workflow.sh` (Workflow Process Runner)
```bash
#!/bin/bash
typeset RUNDIR=$(dirname ${BASH_SOURCE})
RUNDIR=$(cd $RUNDIR ; pwd)

. /home/orafmw/env.par
. ./call.sh

decrypt_password_file
rm -f $RUNDIR/../conf/${SIE_REL_FIL}_pwd.par
. ${SIEBEL_ROOT}/bin/../siebenv.sh

logFilePath="$RUNDIR/../log"
sqlFilePath="$RUNDIR/../sql"
timeStamp2=`date "+%d%m%Y%H%M%S"`
processname="$1"
countfilename="$2"
searchspec="$3"
rowid="$4"

logFileName="$5"_"$timeStamp2".log

Log "$logFilePath" "$logFileName" "Script Started"
Log "$logFilePath" "$logFileName" "Log directory: '$logFilePath'"

Workflow_Process "$processname" "$searchspec" "$rowid" "$logFilePath" "$logFileName" "$timeStamp2"
if [ $? -ne 0 ]
then
  Log "$logFilePath" "$logFileName" "Error executing workflow for \"$processname\". Please check the log"
  exit 1
fi

Invoke_Sqlplus $sqlFilePath $countfilename $logFilePath $logFileName $timeStamp2
if [ $? -ne 0 ]
then
  Log $logFilePath $logFileName "Error running the count file for \"$processname\". Please check the logs"
  exit 1
else
  countLog="$countfilename"_"$timeStamp2".log
  if [ -f $logFilePath/$countLog ]
  then
    passString=`grep "no rows selected" $logFilePath/$countLog`
    if [ "$passString" != "" ]
    then
      failCount=0
    else
      failCount=1
    fi
  else
    Log $logFilePath $logFileName "\tExecuted with errors. Please check the logs"
    exit 1
  fi

  if [ $failCount -ne 0 ]
  then
    Log $logFilePath $logFileName "\tProcess completed with data errors, manual verification required. Please check ${countLog} for details"
    exit 10
  else
    Log $logFilePath $logFileName "Script completed"
  fi
fi
```

### 5.7. `update_wp_tra_status.sql` (Role Refresh to AMS via TRA Status)
```sql
SET SERVEROUTPUT ON
SET VERIFY OFF
DEFINE EIM_TABLE='&1'
DEFINE PO_NAME='&2'
DEFINE SBATCH='&3'
DEFINE EBATCH='&4'
WHENEVER SQLERROR EXIT FAILURE

DELETE SIEBEL.EIM_ORG_EXT_XM WHERE IF_ROW_BATCH_NUM BETWEEN &SBATCH AND &EBATCH;

DECLARE
  stable     VARCHAR2(100);
  sPOName    VARCHAR2(100);
  sql_insert VARCHAR2(32760);
BEGIN
  stable  := '&EIM_TABLE';
  sPOName := '&PO_NAME';

  sql_insert := 'INSERT INTO SIEBEL.EIM_ORG_EXT_XM (ROW_ID, IF_ROW_BATCH_NUM, IF_ROW_STAT, PARTY_UID, PARTY_TYPE_CD, PAR_ACCNT_NAME, PAR_ACCNT_LOC, PAR_ACCNT_BU, XM_NAME, XM_TYPE, ATTRIB_36)
  SELECT ROWNUM, ''200'' || 15 + floor(rownum/5000), ''FOR_IMPORT'', org.PARTY_UID, ''Organization'', org.NAME, org.LOC, org.BU_ID, org_xm.NAME, org_xm.TYPE, SYSDATE
  FROM (SELECT ROW_ID, NAME, LOC, BU_ID, PARTY_UID FROM SIEBEL.S_ORG_EXT) org
  INNER JOIN SIEBEL.S_ORG_EXT_XM org_xm ON org.ROW_ID = org_xm.PAR_ROW_ID
  INNER JOIN (
      SELECT PAR_ROW_ID, NAME, TYPE, X_WP_ROLE_TYPE, X_WP_ROLE_STATUS_TYPE, X_WP_ROLE_STATUS
      FROM SIEBEL.S_ORG_EXT_XM
      WHERE TYPE = ''BPO_ROLE_STATUS''
        AND X_WP_ROLE_TYPE = ''MERCHANT''
        AND X_WP_ROLE_STATUS_TYPE = ''TRA''
        AND X_WP_ROLE_STATUS IN (''LIVE'', ''SUSPENDED'')
  ) org_status ON org.ROW_ID = org_status.PAR_ROW_ID
  WHERE org.NAME IN (
      SELECT ' || sPOName || ' FROM SIEBEL.' || stable || '
      WHERE IF_ROW_BATCH_NUM BETWEEN :1 AND :2
        AND IF_ROW_STAT IN (''IMPORTED'')
  )';

  EXECUTE IMMEDIATE sql_insert USING &SBATCH, &EBATCH;
  COMMIT;
END;
/

EXEC DBMS_STATS.GATHER_TABLE_STATS(OWNNAME => 'SIEBEL', TABNAME => 'EIM_ORG_EXT_XM', no_invalidate => TRUE);

QUIT;
```

### 5.8. `wp_eim_count.sql` (Empty Batch Check)
```sql
SET SERVEROUTPUT ON
SET VERIFY OFF
DEFINE EIM_TABLE='&1'
DEFINE IBATCH='&2'
WHENEVER SQLERROR EXIT FAILURE

DECLARE
  nCnt      VARCHAR2(100);
  sql_query VARCHAR2(300);
BEGIN
  sql_query := 'select count(1) from siebel.' || '&EIM_TABLE' || ' where if_row_batch_num = :1 ';
  EXECUTE IMMEDIATE sql_query INTO nCnt USING &IBATCH;

  IF nCnt > 0 THEN
    DBMS_OUTPUT.PUT_LINE(nCnt || ' rows : EIM data ready');
  ELSE
    DBMS_OUTPUT.PUT_LINE(nCnt || ' rows : ERROR in COUNT');
  END IF;
END;
/
QUIT;
```

### 5.9. Legacy `wp_update_pds_flag.sql` (Original Monolithic Script from Screenshots 21-23)
```sql
SET SERVEROUTPUT ON

DECLARE
    counter      NUMBER(9) := 0;
    CreatedBy    VARCHAR2(15) := '&1';
    sAgreementId VARCHAR2(15);
    sPartyId     VARCHAR2(15);
    sDate        DATE := trunc(SYSDATE);
    sNewDate     DATE := trunc(SYSDATE) - 1;

    CURSOR curr_rec IS
        SELECT /*+ opt_param('hash_join_enabled','true') opt_param('optimizer_mode','all_rows') */
            ROWNUM                              AS ROW_ID,
            'FOR_IMPORT'                        AS IF_ROW_STAT,
            2901900 + FLOOR(ROWNUM / 5000)      AS IF_ROW_BATCH_NUM,
            orgparty.PARTY_UID                  AS PARTY_UID,
            orgparty.PARTY_TYPE_CD              AS PARTY_TYPE_CD,
            org.NAME                            AS NAME,
            org.LOC                             AS LOC,
            orgbu.ROW_ID                        AS ACCNT_BI,
            orgbu.NAME                          AS ACCNT_BU,
            org.X_WP_PRI_PDS_FLG                AS X_WP_PRI_PDS_FLG,
            org.ROW_ID                          AS PARTY_ID
        FROM SIEBEL.S_PARTY orgparty
        INNER JOIN SIEBEL.S_ORG_EXT org ON orgparty.ROW_ID = org.PAR_ROW_ID
        INNER JOIN SIEBEL.S_BU orgbu    ON org.BU_ID = orgbu.ROW_ID
        WHERE org.OU_TYPE_CD = 'Customer'
          AND org.NAME IN (
              SELECT NAME FROM SIEBEL.S_ORG_EXT 
              WHERE ROW_ID IN (
                  SELECT MASTER_OU_ID FROM SIEBEL.S_ORG_EXT 
                  WHERE NAME IN (
                      SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK PMP 
                      WHERE CREATED_BY = CreatedBy
                  )
              )
          );

    reg_rec curr_rec%ROWTYPE;

BEGIN
    DELETE FROM siebel.EIM_ACCOUNT WHERE if_row_batch_num BETWEEN 2901900 AND 2901999;
    DELETE FROM siebel.EIM_ENTLMNT WHERE if_row_batch_num BETWEEN 2901900 AND 2901999;
    DELETE FROM siebel.EIM_DEFECT  WHERE if_row_batch_num BETWEEN 2901900 AND 2901999;
    COMMIT;

    FOR reg_rec IN curr_rec LOOP
        BEGIN
            sPartyId := reg_rec.PARTY_ID;
            SELECT AGREE_ID INTO sAgreementId FROM SIEBEL.CX_AGREE_ORG 
            WHERE ACCNT_ID = sPartyId AND (END_DT IS NULL OR END_DT >= sDate);

            INSERT INTO SIEBEL.EIM_ACCOUNT (
                ROW_ID, IF_ROW_STAT, IF_ROW_BATCH_NUM, PARTY_UID, PARTY_TYPE_CD,
                NAME, LOC, ACCNT_BI, ACCNT_BU, X_WP_PRI_PDS_FLG
            ) VALUES (
                reg_rec.PARTY_ID, reg_rec.IF_ROW_STAT, reg_rec.IF_ROW_BATCH_NUM,
                reg_rec.PARTY_UID, reg_rec.PARTY_TYPE_CD, reg_rec.NAME, reg_rec.LOC,
                reg_rec.ACCNT_BI, reg_rec.ACCNT_BU, 'Y'
            );

            INSERT INTO SIEBEL.EIM_ENTLMNT (
                IF_ROW_BATCH_NUM, IF_ROW_STAT, ROW_ID, ORG_ACCNT_BI, ORG_ACCNT_BU,
                ORG_ACCNT_LOC, ORG_ACCNT_NAME, PAR_AGREE_ACC_BI, PAR_AGREE_ACC_BU,
                PAR_AGREE_ACC_LOC, PAR_AGREE_ACC_NAME, PAR_AGREE_BI, PAR_AGREE_BU,
                PAR_AGREE_NAME, PAR_AGREE_REV_NUM, ENTL_NAME, ORG_EFF_END_DT
            )
            SELECT
                reg_rec.IF_ROW_BATCH_NUM, reg_rec.IF_ROW_STAT, ENT_ORG.ROW_ID,
                '0-R9NH', 'Default Organization', PARTY.LOC, PARTY.NAME,
                '0-R9NH', 'Default Organization', PARTY.LOC, PARTY.NAME,
                '0-R9NH', 'Default Organization', AGREEMENT.NAME, AGREEMENT.REV_NUM,
                ENT.NAME,
                CASE
                    WHEN (ENT_ORG.EFF_START_DT >= sDate) THEN ENT_ORG.EFF_START_DT
                    WHEN (ENT_ORG.EFF_START_DT < sDate)  THEN sNewDate
                END
            FROM SIEBEL.S_DOC_AGREE AGREEMENT,
                 SIEBEL.S_ENTLMNT_ORG ENT_ORG,
                 SIEBEL.S_ENTLMNT ENT,
                 SIEBEL.S_ORG_EXT PARTY
            WHERE AGREEMENT.ROW_ID = ENT.PAR_AGREE_ID
              AND ENT.ROW_ID = ENT_ORG.AGREE_ID
              AND ENT_ORG.OU_ID = PARTY.ROW_ID
              AND (AGREEMENT.EFF_END_DT IS NULL OR AGREEMENT.EFF_END_DT >= sDate)
              AND (ENT_ORG.EFF_END_DT IS NULL OR ENT_ORG.EFF_END_DT >= sDate)
              AND ENT.ENTL_TYPE_CD = 'PRICING'
              AND AGREEMENT.ROW_ID = sAgreementId;

            INSERT INTO SIEBEL.EIM_DEFECT (
                IF_ROW_BATCH_NUM, IF_ROW_STAT, ROW_ID, DEFECT_NUM
            )
            SELECT reg_rec.IF_ROW_BATCH_NUM, 'IMPORTED', ROW_ID, NAME
            FROM SIEBEL.S_ORG_EXT WHERE MASTER_OU_ID = sPartyId;

            counter := counter + 1;
            IF counter = 5000 THEN
                COMMIT;
                counter := 0;
            END IF;

        EXCEPTION
            WHEN OTHERS THEN
                UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
                SET SDM_MIGRATED_STATUS = 'FAILURE', SDM_ERROR_MSG = 'SQL_PROCESSING_ERROR', LAST_UPD = SYSTIMESTAMP
                WHERE PARTY_ID IN (SELECT NAME FROM SIEBEL.S_ORG_EXT WHERE MASTER_OU_ID = sPartyId);
                CONTINUE;
        END;
    END LOOP;
    COMMIT;

    UPDATE PDS_MIGRATED_PARTY@EDS_AMS_DBLINK
    SET SDM_MIGRATED_STATUS = 'FAILURE', SDM_ERROR_MSG = 'PO NUMBER DOES NOT EXIST IN SIEBEL', LAST_UPD = SYSTIMESTAMP
    WHERE PARTY_ID NOT IN (
        SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK A, SIEBEL.S_ORG_EXT B WHERE A.PARTY_ID = B.NAME
    ) AND CREATED_BY = CreatedBy;
    COMMIT;
END;
/
EXIT;
```
