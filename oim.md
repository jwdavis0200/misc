# OIM 12c — UAT child insert-trigger task not firing: action runbook

Case state (as established):
- Role request -> Evaluate User Policies -> AP provisions NEW account with child data.
- Child row IS inserted; NO child insert-trigger task instance ever appears
  (task history: System Validation + Create User only).
- Manual child-add on an EXISTING account DOES fire the task.
- Restart, cache, lookup dedupe, recon, BP/form/RO export comparison, new child
  table, new policy, latest version active: all tried, issue persists.
- Other apps with Entitlement=true and NO capture jobs work fine.

Remaining candidates: (1) child value representation written by policy child
data vs the lookup's encoded value; (2) policy composition/linkage (which policy
creates the account vs which supplies the child delta).

Column names below are from OIM data dictionaries (community references) and
Oracle doc patterns. Verify with the appendix query if any errors.

---

## STEP 1 — Snapshot the account and what ran

```sql
-- 1a. account row (SELECT * avoids column guessing)
SELECT * FROM oiu WHERE orc_key = &orc;

-- 1b. tasks actually inserted for this account
SELECT m.mil_name, s.sch_status
FROM osi o
JOIN sch s ON s.sch_key = o.sch_key
JOIN mil m ON m.mil_key = o.mil_key
WHERE o.orc_key = &orc
ORDER BY m.mil_name;
```

Expect: System Validation + Create User only.

---

## STEP 2 — Decisive: policy-written value vs lookup encoded value (read-only)

```sql
-- 2a. child data configured on policies attached to the role
SELECT p.pol_key, p.pol_name, c.poc_field_value
FROM pol p
JOIN pog g ON g.pol_key = p.pol_key
JOIN ugp u ON u.ugp_key = g.ugp_key
LEFT JOIN poc c ON c.pol_key = p.pol_key
WHERE u.ugp_name = '&role_name';

-- 2b. single-row comparison: written vs configured vs lookup-encoded
SELECT (SELECT &child_col FROM &child_table WHERE orc_key = &orc) AS written_value,
       (SELECT c.poc_field_value FROM poc c
         WHERE c.pol_key = &pol_key AND ROWNUM = 1)                AS policy_value,
       (SELECT v.lkv_encoded FROM lkv v JOIN lku k ON k.lku_key = v.lku_key
         WHERE k.lku_code = '&lookup_name'
           AND v.lkv_encoded = (SELECT &child_col FROM &child_table
                                 WHERE orc_key = &orc))             AS lookup_encoded
FROM dual;

-- 2c. byte-level check of the child row value
SELECT &child_col, LENGTH(&child_col) AS len, DUMP(&child_col, 1016) AS bytes
FROM &child_table WHERE orc_key = &orc;
```

Reading:
- written = policy = lookup_encoded (same bytes/length) -> representation OK, go Step 3.
- written = policy, lookup_encoded NULL or bytes differ -> CAUSE: policy child
  data literal is not the lookup's encoded value. Fix policy child data to the
  encoded value; repro.
- policy != written -> engine transforms value; capture and escalate.

Note: ENT_LIST may be empty (capture jobs not run). Expected, not the issue.
This test uses LKV (the lookup), not ENT_LIST.

---

## STEP 3 — Is your policy the one creating the account?

```sql
SELECT p.pol_key, p.pol_name, COUNT(c.poc_field_value) AS child_rows
FROM pol p
JOIN pog g ON g.pol_key = p.pol_key
JOIN ugp u ON u.ugp_key = g.ugp_key
LEFT JOIN poc c ON c.pol_key = p.pol_key
WHERE u.ugp_name = '&role_name'
GROUP BY p.pol_key, p.pol_name;
```

- Exactly one policy (yours) -> linkage clean.
- Multiple policies, or another policy -> account may be created by one policy
  and child data applied as a secondary delta. Test: remove the app instance
  from the other policies in UAT, repro with a fresh user, re-run Step 1.

```sql
SELECT * FROM oiu WHERE orc_key = &orc;
-- compare POL_KEY (if present) with the pol_key above.
-- If POL_KEY is absent in your schema, ENT_ASSIGN carries a provisioning
-- mechanism column to the same effect.
```

---

## STEP 4 — Only if 2 & 3 are clean: instrument and compare

Enable on UAT OIM server (EM -> Logs -> Log Configuration, or logging.xml):
  oracle.iam.platform.kernel            = FINE
  oracle.iam.accesspolicy               = TRACE:16
  oracle.iam.transUI                    = TRACE:16
  oracle.iam.platform.entitymgr         = TRACE:16
  oracle.iam.identity.scheduledtasks    = TRACE:16

Restart managed server. One fresh-user repro. Capture orchestration id from logs,
then RequestDiagnostic/OrchestrationEngine via EM System MBean Browser. Diff
against the working instance's creation in the same environment. That diff is the
SR artifact.

---

## Appendix — column verification (one query, only if something errors)

```sql
SELECT table_name, column_name, data_type
FROM all_tab_columns
WHERE table_name IN ('OIU','POL','POC','POF','ENT_LIST','ENT_ASSIGN',
                     'LKV','LKU','OSI','SCH','MIL','ORC','PKG','TOS','SDK')
ORDER BY table_name, column_id;
```

Verified names used above: POC.POC_FIELD_VALUE, POC.POL_KEY, POL.POL_KEY/POL_NAME,
POG.POL_KEY/POG.UGP_KEY, UGP.UGP_KEY/UGP_NAME, LKU.LKU_KEY/LKU_CODE,
LKV.LKU_KEY/LKV_ENCODED/LKV_DECODED, OSI.ORC_KEY/OSI.SCH_KEY/OSI.MIL_KEY,
SCH.SCH_KEY/SCH_STATUS, MIL.MIL_KEY/MIL_NAME, OIU.ORC_KEY.
Community-documented but not Oracle-doc-verified: OIU.POL_KEY (Step 1a uses
SELECT * so it does not matter).


Rebuild the process definition for the same RO

Design Console route (clean, slow):
1. Design Console → Process Management → Process Definition.
2. Create the new definition: Actions → Create New (or the new-record icon). Fill:
   - Name: TRDTH User V2
   - Type: Provisioning
   - Object Name: TRDTH User (same RO — the app instance keeps pointing at the same resource object)
   - Table Name: UD_TRDTH_U
   - Default Process: leave unchecked for now
   - Auto Save Form / Auto Pre-Populate: checked (matches the current definition: TOS_AUTOSAVE=1, TOS_AUTO_PREPOP=1)
3. Save, then recreate the tasks. Your export has 17 in TRDTH User; the ones that matter for the bug are:
   - TRDTH Group Insert — Insert trigger, child table UD_TRTH_GR1, Allow Multiple Instances, Conditional, Required for Completion, Allow Cancellation while Pending, No Effect, Offline unset, adapter adpAS400OPENCHILDTABLEINSERT with the same 9 variable mappings (childTableName=UD_TRTH_GR1, itResourceFieldName=UD_TRDTH_U_IT, processTaskName literal, applicationName via ITResourceDef, etc.)
   - TRDTH Group Delete — same but Delete trigger
   - TRDTH Entity Insert / Delete — same, child table UD_TRTH_EN1
   - Create User, Delete User, Enable User, Disable User, Password Updated, the three Reconciliation … Received tasks   - TRDTH Entity Insert / Delete — same, child table UD_TRTH_EN1
   - Create User, Delete User, Enable User, Disable User, Password Updated, the three Reconciliation … Received tasks, the attestation tasks, Service Account …/System Validation                                                            For each, carry over: General tab values, Integration tab adapter + variable mappings, Responses tab (UNKNOWN → R, SUCCESS → P), Task-to-Object Status mapping, Task Assignment rules. There are no task dependencies in the current definition, so nothing to recreate there.
4. Switch the default: open the old "TRDTH User" and uncheck Default Process; open "TRDTH User V2" and check Default Process; save both. Only one default per RO.
5. PurgeCache + restart, then fresh-user repro. Verify the instance used the new definition:
   SELECT o.orc_key, p.pkg_name FROM orc o JOIN pkg p ON p.pkg_key = o.pkg_key WHERE o.orc_key = &orc;                                                                                                                                    Faster variant (test env, if you accept a small risk): Deployment Manager export of the process, edit only the Process element's name attribute ("TRDTH User" → "TRDTH User V2"), keep task names as-is (they're unique within a process, not globally), then import. Oracle discourages hand-editing DM XML, so do this in UAT on a copy first, and compare task counts before/after.

STEP 3 — Re-save the application instance so the wizard rewrites its associations                                    
Verified path: Identity System Administration → left pane, Provisioning Configuration → Application Instances → search TRDTH → Actions → Open → Attributes tab → change nothing meaningful (e.g., re-enter the Display Name) → Save.                                                                                                                           Then force the form association to be re-applied: on the Application Instance details page, click Refresh next to the Form field, select the form from the list, click Apply. That's the documented lever for re-pointing an instance's form, and it rewrites the association rows the import rebuilt for you.                                                 
After saving, run the Catalog Synchronization Job (the docs call for it after modifying application instance attributes), then PurgeCache, restart, fresh-user repro.

STEP 4 — Re-import the same XML under the same names                                                                 
Verified path: Identity System Administration → left pane, System Configuration → Import → Browse to your export → Next.

Import Options — the choices that matter:
- User Reference: Keep Original                                                                                      - Role Reference: Keep Original
- If Object Exists: Override (this replaces the stored definitions with the XML's — which is the point of the repair; Skip does nothing for you)

Next/Summary → Import.                                                                                                                                                                                                                    Guardrails, all from the docs: back up the database first (Oracle recommends it before any import); form/field imports add entries that cannot be rolled back; only System Administrators see the Deployment Manager menu; and re-imports merge rather than replace, so verify counts before and after:                                                        
  SELECT COUNT(*) FROM mil WHERE tos_key = (SELECT tos_key FROM tos t JOIN pkg p ON p.pkg_key=t.pkg_key WHERE p.pkg_name='TRDTH User');
  SELECT sdk_name, sdk_active_version, sdk_latest_version FROM sdk WHERE sdk_name LIKE 'UD_TRDTH%' OR sdk_name LIKE 'UD_TRTH%';

If task or form counts grow, the import duplicated rather than repaired — stop and restore.                                                                                                                                               Recommended order for you: 3 first (cheapest, no duplication risk, and it directly rewrites what the import rebuilt), then 1/2 if 3 doesn't take, then 4 only as a controlled test with a backup. And keep the working re-imported instance in place as your fallback throughout — you've already proven it works, so you're not risking downtime on this investigation.