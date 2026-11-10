# Siebel Datafix & Data Loader Framework - Technical Documentation & KN-221 Implementation

## 1. Introduction and Architecture Overview
This framework provides a modular Unix shell and Oracle SQL environment designed to automate data corrections (Datafixes), bulk data CSV loading via SQL*Loader, and Siebel Workflow/EIM executions in Siebel CRM.

### Core Architectural Principles:
* **Separation of Concerns:** Generic execution engines (`wp_generic_*.sh`) are cleanly separated from task-specific configuration folders (deployment packages).
* **Compliance:** Direct DML modifications to Siebel base tables are avoided where possible by utilizing standard **Siebel EIM (Enterprise Integration Manager)** or proper **Role Refresh (TRA)** mechanisms to ensure data integrity and federation to downstream systems like **AMS**.
* **Auditability & Error Handling:** Strict exit codes (`exit 10` for manual check required, `exit 0` for clean termination) and automated log management with timestamps.

---

## 2. Directory Structure on Target Server (Unix/Linux)
* `/u01/app/oracle/product/siebel/scripts/bin/` (`$RUNDIR`) – Core generic execution scripts (`call.sh`, `wp_generic_datafix_param.sh`, etc.).
* `../conf/` – Encrypted credentials (`*_pwd.par.gpg`).
* `../sql/` – SQL procedure scripts.
* `../ifb/` – Siebel EIM control files (`.ifb`).
* `../log/` & `../log/history/` – Operation logs and archives.

---

## 3. KN-221 Task Specific Implementation (PDS Flag & Entitlement Expiry)

To fulfill the requirements of **KN-221**, the original monolithic script has been cleanly split into two dedicated packages (`RENAME EVERYTHING` approach):

1. **Package I (`wp_kn221_update_pds_flag`):** Sets `X_WP_PRI_PDS_FLG = 'Y'` on Group Merchants and all child POs across all hierarchy levels, skipping already updated records, and triggers a Role Refresh (`TRA`) to AMS.
2. **Package II (`wp_kn221_expire_entitlement`):** Expires active Pricing Entitlements by setting `ValidTo` to the previous calendar day (`TRUNC(SYSDATE) - 1`), skipping already expired records, and triggers a Role Refresh to AMS.

---

## 4. Complete Source Code (KN-221 Implementation)

### 4.1. Package I: PDS Flag Update across Hierarchy

#### Wrapper Script: `wp_kn221_update_pds_flag.sh`
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

#### SQL Procedure: `wp_kn221_update_pds_flag.sql`
```sql
SET SERVEROUTPUT ON
SET VERIFY OFF
WHENEVER SQLERROR EXIT FAILURE

DECLARE
    counter   NUMBER(9) := 0;
    CreatedBy VARCHAR2(15) := '&1';

    CURSOR curr_rec IS
        SELECT 
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
          AND NVL(org.X_WP_PRI_PDS_FLG, 'N') <> 'Y'
          AND (
              org.ROW_ID IN (
                  SELECT ROW_ID FROM SIEBEL.S_ORG_EXT
                  START WITH ROW_ID IN (
                      SELECT ROW_ID FROM SIEBEL.S_ORG_EXT 
                      WHERE NAME IN (SELECT PARTY_ID FROM PDS_MIGRATED_PARTY@EDS_AMS_DBLINK WHERE CREATED_BY = CreatedBy)
                  )
                  CONNECT BY NOCYCLE PRIOR ROW_ID = PAR_OU_ID
              )
              OR org.MASTER_OU_ID IN (
                  SELECT ROW_ID FROM SIEBEL.S_ORG_EXT 
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
                WHERE PARTY_ID = reg_rec.NAME;
                CONTINUE;
        END;
    END LOOP;
    COMMIT;
END;
/
EXIT;
```

#### EIM Control File: `wp_kn221_update_pds_flag.ifb`
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

---

### 4.2. Package II: Pricing Entitlement Expiry (`SYSDATE - 1`)

#### Wrapper Script: `wp_kn221_expire_entitlement.sh`
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

#### SQL Procedure: `wp_kn221_expire_entitlement.sql`
```sql
SET SERVEROUTPUT ON
SET VERIFY OFF
WHENEVER SQLERROR EXIT FAILURE

DECLARE
    counter      NUMBER(9) := 0;
    CreatedBy    VARCHAR2(15) := '&1';
    sAgreementId VARCHAR2(15);
    sPartyId     VARCHAR2(15);
    sDate        DATE := TRUNC(SYSDATE);
    sNewDate     DATE := TRUNC(SYSDATE) - 1;
    v_rows_ins   NUMBER := 0;

    CURSOR curr_rec IS
        SELECT 
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
                  AND (ENT_ORG.EFF_END_DT IS NULL OR ENT_ORG.EFF_END_DT >= sDate)
                  AND ENT.ENTL_TYPE_CD = 'PRICING'
                  AND AGREEMENT.ROW_ID = sAgreementId;

                v_rows_ins := SQL%ROWCOUNT;

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
                WHERE PARTY_ID = reg_rec.NAME;
                CONTINUE;
        END;
    END LOOP;
    COMMIT;
END;
/
EXIT;
```

#### EIM Control File: `wp_kn221_expire_entitlement.ifb`
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