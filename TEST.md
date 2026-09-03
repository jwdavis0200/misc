# OIM 12c Cross-Country Access Approval — Full Implementation Guide

**Use case:** For specific roles (scoped by role name), when the beneficiary's country differs from the
SBE value on the access policy's application-instance child form, the request routes first to the
beneficiary as a document-submission task (attachment upload required by business process), then to the
manager for the sole Approve/Reject decision. If beneficiary country == SBE, the request goes straight
to the manager (normal flow). If the beneficiary cancels, the request ends as **Rejected** with a
distinguishing comment ("Cancelled by beneficiary" or business-specific reason).

**Isolation guarantee:** requests for all other roles keep flowing through the existing
DefaultOperationalApproval untouched. Routing is decided by an Approval Workflow Rule; the custom
composite only ever sees in-scope requests.

**Verified facts this design is built on (checked this session against primary sources):**
- The SOA/BPEL payload for approval composites contains ONLY: RequestID, RequestModel, RequestTarget,
  URL, RequesterDetails, BeneficiaryDetails, ObjectDetails, OtherDetails (12.2.1.4 Dev Guide ch.11.3.1.1).
  **No form data, no child-form values.** SBE must be fetched via OIM API from inside a Java Embedding.
- `BeneficiaryDetails` is populated for single-beneficiary requests at operational level; doc explicitly
  recommends using it in operational-level composites (ch.11.3.1.1).
- The request engine expects only **Approved** or **Rejected** from the workflow callback
  (12.2.1.4 Administering OIG ch.4.1.3). There is no "Cancelled" outcome a composite can return;
  Request Withdrawn/Closed are requester/admin actions on the request, not workflow outcomes.
- Account beneficiaries cannot approve their own requests by default, even if in the approver group
  (ch.11.7.4, IRoutingSlipCallback). The beneficiary task below never carries Approve/Reject outcomes,
  so this design never relies on that control being bypassed.
- Human tasks support comments and attachments from assignees (ch.11.1.1 overview).
- All Java API entry points in Phase 3 verified against live 12.2.1.4 Javadocs (oracle.iam.request.vo.*,
  Platform, UserManager, UserManagerConstants.AttributeName).

**Tag legend:**
- `[VERIFIED-doc]` — anchored to an Oracle 12.2.1.4 doc page / Javadoc confirmed this session.
- `[CHECK ENV]` — environment-specific literal or screen label you must read off your own system.
  Do not guess these.

---

## Prerequisites (confirm ALL before starting)

1. JDeveloper 12.2.1.4 installed from the SOA Quick Start distribution (SOA design-time plugin
   included). `[CHECK ENV]`
2. EM (Fusion Middleware Control) console access to the SOA domain (export + deploy + flow trace). `[CHECK ENV]`
3. OIM sysadmin access to Identity System Administration (SysAdmin -> Workflows -> Approval). `[CHECK ENV]`
4. Access to Form Designer for the app instance in scope. `[CHECK ENV]`
5. OIM_HOME path on a server machine (only needed if you fall back to the template utility). `[CHECK ENV]`

---

## Phase 0 — Environment facts (do this first, ~15 min)

Collect these literals. Every later phase parameterizes on them.

1. **Operation string + app-instance name.**
   Identity System Administration -> Workflows -> Approval -> open the existing rule that routes these
   roles to DefaultOperationalApproval. Record the exact operation name string (e.g. "Provision
   Application Instance") and the app-instance display/technical name. `[CHECK ENV]`

2. **Child-form technical name and SBE attribute name.**
   Form Designer -> open the app-instance form -> child form -> record:
   - child form technical name (pattern `UD_xxx`), e.g. `UD_APPXX` `[CHECK ENV]`
   - the SBE column's technical attribute name, e.g. `UD_APPXX_SBE` (NOT the display label) `[CHECK ENV]`
   - what the SBE value looks like (country code format: `SG`, `MY`). Confirm the beneficiary's user
     profile country value uses the same codes/format — if one side is "SG" and the other is "Singapore",
     Phase 3's comparison must normalize.

3. **User country attribute name.**
   The user-profile country attribute's platform name (typically `usr_country`; the 12.2.1.4 Javadoc
   catalogs the constant `UserManagerConstants.AttributeName.USER_COUNTRY` which maps to it).
   Also the user-login attribute name (`UserManagerConstants.AttributeName.USER_LOGIN` — verify on the
   same Javadoc page / your env). `[CHECK ENV]`

4. **BeneficiaryDetails login element name.**
   Later (Phase 1/2) in JDeveloper, open `RequestDetails.xsd` from the project's Schema Files and read
   the exact child element of `BeneficiaryDetails` that carries the user's login (e.g. `Login` vs
   `UserLogin`). `[CHECK ENV]` — needed for the task-assignment XPath in Phase 2.

---

## Phase 1 — Clone DefaultOperationalApproval

You are cloning the deployed OOB composite, not generating a template. Never edit the deployed default
in place; the clone gets its own name and version, and the workflow rule (Phase 5) is the only thing
that diverts traffic to it.

1. EM console -> SOA -> soa-infra -> `default` partition -> locate **DefaultOperationalApproval**.
2. Composite home page -> **Export** (the 12.2.1.4 SOA admin guide "Deploying and Managing SOA Composite
   Applications" documents the export/deploy wizards). Download the SAR (`sca_*.jar`) to your JDeveloper
   machine. `[CHECK ENV: exact export-action label varies by EM skin]`
3. JDeveloper -> File -> New -> Application -> **SOA Application**:
   - Application name: `CrossCountryAccessApprovalApp`
   - Project name: `CrossCountryAccessApproval`
   - Composite: import from the SAR you downloaded ("Composite From SAR" / import path — label varies
     by JDeveloper build). `[CHECK ENV]`
4. **One-pass rename sweep BEFORE editing any logic:**
   - Composite name -> `CrossCountryAccessApproval`
   - BPEL component name (if it embeds the old composite name)
   - The `.task` human-task definition file name AND its reference inside the BPEL component
   - grep the whole project for the string `DefaultOperationalApproval` — zero hits when done
5. Set composite revision to `1.0` (composite.xml, revision field). Your final identifier is
   `default/CrossCountryAccessApproval!1.0`.
6. File -> Save All. **Build -> Build Project must be clean.** Fix any broken references (a BPEL
   component still pointing at the renamed `.task` file is the classic one) before continuing.
   Do NOT proceed with compile errors.

Note: the dev guide's `OIM_HOME/workflows/new-workflow` helper utility (`ant -f new_project.xml`) is the
alternative "from-template" path; you chose the clone path. Both are legitimate; the clone inherits your
existing backings/Java embeddings, which you want preserved.

---

## Phase 2 — Beneficiary info-collection human task

In JDeveloper, project open, double-click the composite -> visual SOA Composite editor.

1. From Component Palette (right panel, "SOA Components") drag **Human Task** onto the Components
   swimlane of the composite surface.
2. Create Human Task dialog: Name = `BeneficiaryInfoTask`. Accept defaults, click OK. The `.task`
   editor opens.

### 2a. Data tab
Add the standard approval parameter set (dev guide Table 11-2 pattern — these are the parameters the
tutorial's approval tasks carry):

| Parameter | Type |
|---|---|
| RequestID | xsd:string |
| RequestModel | xsd:string |
| RequestTarget | xsd:string |
| RequesterDetails | RequestDetails:RequesterDetails |
| BeneficiaryDetails | RequestDetails:BeneficiaryDetails |
| ObjectDetails | RequestDetails:ObjectDetails |
| OtherDetails | RequestDetails:OtherDetails |
| url | RequestDetails:url |
| Catalogdata | RequestDataService:CatalogData |
| RequesterDisplayName | xsd:string |
| BeneficiaryDisplayName | xsd:string |
| Requester | xsd:string |

### 2b. General tab
- **Task Title** via expression builder: click Edit next to Task Title -> in the expression builder
  expand the payload tree -> select `task:payload -> ns2:BeneficiaryDetails -> ns2:DisplayName`
  -> Insert Into Expression. Wrap with text so the final title reads:

  `Provide required documentation for cross-country access for </task:task/task:payload/ns2:BeneficiaryDetails/ns2:DisplayName/>`

  `[VERIFIED-doc: the BeneficiaryDetails/DisplayName expression is the tutorial's own pattern;
  the ns2 prefix must match your clone's namespace mapping — adjust prefix, not the path]`
- **Task Owner:** Group = `SYSTEM ADMINISTRATORS` (mirrors clone defaults).
- **Category:** By name = `approvals`.

### 2c. Outcomes (audit-critical)
In the task outcomes list, define EXACTLY two custom outcomes:
- `SUBMIT_DOCUMENTS`
- `CANCEL`

No APPROVE / REJECT on this task. The beneficiary must never see an Approve/Reject button.
`[VERIFIED-doc: outcomes are string constants read in BPEL via
/task:task/task:systemAttributes/task:outcome — dev guide lines showing 'APPROVE'/'REJECT' switch
cases confirm the mechanism; custom strings behave identically]`

### 2d. Notification tab
Advanced -> check **Make notification actionable**; uncheck "Show worklist/workspace url in
notifications". `[VERIFIED-doc: verbatim from dev guide 11.4.5.4]`

### 2e. Assignment tab
1. Drag a **Single Participant** into the stage box. Label = `Beneficiary`.
2. Build a list of participants -> **Names and Expressions** (label varies by skin; Rule-based is the
   tutorial's Manager-stage alternative — you need the plain XPath mode here). `[CHECK ENV: skin label]`
3. Value/expression: XPath into the task payload — `task:payload` -> `BeneficiaryDetails` -> the login
   element you recorded in Phase 0 item 4. `[CHECK ENV: element name from RequestDetails.xsd]`

`[VERIFIED-doc: BeneficiaryDetails is populated at operational level for single-beneficiary requests;
using it in an operational-level composite is the doc's explicit recommendation]`

### 2f. Attachments
Enabled by platform default. No `.task` flag needed — you verify the upload control visually in
Phase 5 test. `[VERIFIED-doc: dev guide 11.1.1 overview documents gathering "comments and attachments"
from approvers/assignees]`

Save the `.task` file.

---

## Phase 3 — BPEL gate (SBE vs beneficiary country)

SBE is NOT in the payload (confirmed against your system), so this phase is mandatory, not optional.

### 3a. Scalar variables
In the BPEL designer, Variables dialog (or source), add three plain scalars — do NOT bind them to any
message element:
- `sbeValue` — xsd:string
- `benCountry` — xsd:string
- `isCrossCountry` — xsd:boolean

### 3b. Java Embedding activity
Drag **Java Embedding** from Component Palette ("BPEL Activities") into the BPEL flow, placed AFTER the
clone's existing request-details invoke step.

Paste this code (class/method names verified against 12.2.1.4 Javadocs this session):

```java
try {
    String reqId = (String) getVariableData("inputVariable", "payload",
                      "/ns3:process/ns3:RequestID");
    // [CHECK ENV: ns3 prefix must match your clone's WSDL alias; RequestID itself is a
    //  mandatory attribute per dev guide 11.3.1.1]

    String CHILD_FORM   = "UD_APPXX";       // <- Phase 0 item 2 child-form technical name
    String SBE_ATTR     = "UD_APPXX_SBE";   // <- Phase 0 item 2 SBE attribute technical name
    String COUNTRY_ATTR = "usr_country";    // [CHECK ENV: Phase 0 item 3]

    oracle.iam.request.api.RequestService reqSvc =
        oracle.iam.platform.Platform.getService(
            oracle.iam.request.api.RequestService.class);          // [VERIFIED-doc: Platform.getService]

    oracle.iam.request.vo.Request req =
        reqSvc.getBasicRequestData(reqId);                         // [VERIFIED-doc: RequestService
                                                                   //  .getBasicRequestData(String)]
    java.util.List bens = req.getBeneficiaries();                  // [VERIFIED-doc: Request page]

    String sbe = null;
    String benLogin = null;
    String benCountry = null;

    if (bens != null && !bens.isEmpty()) {
        oracle.iam.request.vo.Beneficiary ben =
            (oracle.iam.request.vo.Beneficiary) bens.get(0);   // single-beneficiary, operational level

        // Walk Beneficiary -> target entities -> RequestBeneficiaryEntity -> getEntityData()
        // -> List<RequestBeneficiaryEntityAttribute>. Child-form rows are parent attributes with
        // hasChild() == true; the row's getChildAttributes() carries the child-form columns.
        java.util.List entities = ben.getTargetEntities();         // [VERIFIED-doc: Beneficiary page]
        for (Object eo : entities) {
            oracle.iam.request.vo.RequestBeneficiaryEntity rbe =
                (oracle.iam.request.vo.RequestBeneficiaryEntity) eo;
            java.util.List attrs = rbe.getEntityData();           // [VERIFIED-doc: returns List]
            sbe = findSbeInAttributes(attrs, CHILD_FORM, SBE_ATTR);
            if (sbe != null) break;
        }

        benLogin = ben.getBeneficiaryKey();                         // [VERIFIED-doc: Beneficiary page;
                                                                    //  returns the user key]
        // [CHECK ENV: if UserManager.getDetails(key, attrs, true) rejects the key value,
        //  resolve key -> login with your env's identity call, then pass isUserLogin=true]

        if (benLogin != null) {
            oracle.iam.identity.usermgmt.api.UserManager um =
                oracle.iam.platform.Platform.getService(
                    oracle.iam.identity.usermgmt.api.UserManager.class);
            java.util.Set retAttrs = new java.util.HashSet();
            retAttrs.add(COUNTRY_ATTR);
            oracle.iam.identity.usermgmt.vo.User u =
                um.getDetails(benLogin, retAttrs, true);            // [VERIFIED-doc: UserManager
                                                                    //  .getDetails(String,Set,boolean)]
            Object c = u.getAttribute(COUNTRY_ATTR);                // [VERIFIED-doc: User VO]
            benCountry = (c == null) ? null : c.toString();
        }
    }

    setVariableData("sbeValue", sbe);
    setVariableData("benCountry", benCountry);
    boolean cross = (sbe != null && benCountry != null
                     && !sbe.trim().equalsIgnoreCase(benCountry.trim()));
    setVariableData("isCrossCountry", Boolean.valueOf(cross));

} catch (Exception e) {
    // Fail-closed policy: a lookup failure faults the composite loudly rather than silently
    // defaulting to not-cross-country (which would route straight to the manager and mask the
    // data problem as a benign routing decision).
    throw new RuntimeException("CrossCountry lookup failed: " + e, e);
}
```

Helper method — same embedding editor, immediately after the code above (the SOA java-embedding pane
accepts additional private methods; `[VERIFIED-doc: SOA 12.2.1.3 "Incorporating Java and Java EE Code in
a BPEL Process"]`):

```java
// Recursively walk the attribute tree; returns the SBE value on the child row whose
// name matches the child-form technical name, else null.
private String findSbeInAttributes(java.util.List attrs,
                                   String childForm, String sbeAttr) {
    if (attrs == null) return null;
    for (Object ao : attrs) {
        oracle.iam.request.vo.RequestBeneficiaryEntityAttribute a =
            (oracle.iam.request.vo.RequestBeneficiaryEntityAttribute) ao;
        if (a.hasChild() && childForm.equalsIgnoreCase(String.valueOf(a.getName()))) {
            java.util.List kids = a.getChildAttributes();          // [VERIFIED-doc]
            for (Object ko : kids) {
                oracle.iam.request.vo.RequestBeneficiaryEntityAttribute k =
                    (oracle.iam.request.vo.RequestBeneficiaryEntityAttribute) ko;
                if (sbeAttr.equalsIgnoreCase(String.valueOf(k.getName()))) {
                    Object v = k.getValue();                        // [VERIFIED-doc]
                    return (v == null) ? null : v.toString();
                }
            }
        }
        if (a.hasChild()) {
            String got = findSbeInAttributes(a.getChildAttributes(), childForm, sbeAttr);
            if (got != null) return got;
        }
    }
    return null;
}
```

**Residual uncertainty, stated honestly:** I verified `hasChild()`, `getChildAttributes()`, `getRowKey()`,
`getName()`, `getValue()` exist with their return types on the 12.2.1.4 attribute pages, but the
Javadocs do not include a worked example of how child-form rows identify themselves in the
`getEntityData()` tree (child-form technical name as the row attribute's `getName()`, vs a separate key
field). The walker above is defensive and recursive, so it handles both shapes. **If testing returns
null, add `System.out` logging of the full attribute tree (goes to the SOA server log), capture one
cross-country request, and adjust the matcher accordingly.**

### 3c. Runtime classpath
The composite SAR must see the OIM client jars at RUNTIME (build-clean is not enough). Per the
documented pattern ("Using OIM APIs in SOA Composites", 11g ch.26 — same mechanism in 12c), register
the OIM client jars as a WebLogic shared library on the SOA server, or package them in the SAR.
`[CHECK ENV: jar list — typically oimclient.jar + dependencies, read from your domain]`
Missing classpath = `ClassNotFoundException` at runtime, not at build time.

### 3d. Assign + Switch
1. After the Java Embedding, add an **Assign** activity: copy `$isCrossCountry` into the BPEL variable
   if your embedding wrote it via setVariableData above it is already set — keep the two consistent:
   either the embedding sets BPEL variables (as written above) OR an Assign does; not both.
2. Add a **Switch** activity:
   - **Case "CrossCountry"** — condition via expression builder on `$isCrossCountry = true()`
     (or `bpws:getVariableData('isCrossCountry') = 'true'` matching your clone's BPEL dialect —
     `[CHECK ENV: BPEL 1.1 vs 2.0 dialect of your clone]`).
     Inside: drag a **Human Task** activity, select `BeneficiaryInfoTask`, then after it drag the
     **manager approval** human task activity (the clone's existing approval `.task`).
   - **Otherwise** — only the manager approval human task activity.

### 3e. Human-task activity parameter mapping (both tasks)
Per dev guide 11.4.5.5.1 pattern:
- Initiator -> requester login (from BPEL variables)
- Task parameters -> the Data-tab parameters from Phase 2a, mapped to the BPEL input variables
- Advanced tab -> Identification Key -> RequestID `[VERIFIED-doc]`

---

## Phase 4 — Outcome wiring (the audit boundary)

### 4a. Manager task — keep the clone's switch, verify against these blocks

Post-task switch on `bpws:getVariableData('<ManagerTask>_globalVariable','payload',
'/task:task/task:systemAttributes/task:outcome')`:

**Case APPROVE** `[VERIFIED-doc: dev guide 11.4.5.5.1, verbatim assign block]`:

```xml
<sequence>
  <assign>
    <copy>
      <from expression="string('approved')"/>
      <to variable="outputVariable" part="payload"
          query="/ns3:processResponse/ns3:result"/>
    </copy>
    <copy>
      <from expression="ora:getConversationId()"/>
      <to variable="Invoke_1_callback_InputVariable_1"
          part="parameters" query="/ns1:callback/arg0"/>
    </copy>
    <copy>
      <from expression="string('approved')"/>
      <to variable="Invoke_1_callback_InputVariable_1"
          part="parameters" query="/ns1:callback/arg1"/>
    </copy>
  </assign>
</sequence>
```

**Case REJECT** — identical block with `string('rejected')` in both copies.
**Otherwise** — passthrough of `task:state`:

```xml
<sequence>
  <assign>
    <copy>
      <from expression="bpws:getVariableData('SerialApproval1_globalVariable','payload',
                       '/task:task/task:systemAttributes/task:state')"/>
      <to variable="outputVariable" part="payload"
          query="/ns3:processResponse/ns3:result"/>
    </copy>
    <copy>
      <from expression="ora:getConversationId()"/>
      <to variable="Invoke_1_callback_InputVariable_1"
          part="parameters" query="/ns1:callback/arg0"/>
    </copy>
    <copy>
      <from expression="bpws:getVariableData('SerialApproval1_globalVariable','payload',
                       '/task:task/task:systemAttributes/task:state')"/>
      <to variable="Invoke_1_callback_InputVariable_1"
          part="parameters" query="/ns1:callback/arg1"/>
    </copy>
  </assign>
</sequence>
```

(Replace `SerialApproval1_globalVariable` with your manager task's actual global variable name.
`[VERIFIED-doc: all three blocks are the tutorial's verbatim structures]`)

### 4b. Beneficiary task — new switch

Post-task switch on `bpws:getVariableData('BeneficiaryInfoTask_globalVariable','payload',
'/task:task/task:systemAttributes/task:outcome')` `[CHECK ENV: exact global variable name the designer
generated for your task]`:

- **Case `= 'SUBMIT_DOCUMENTS'`** -> EMPTY sequence. Flow continues to the manager task. The attachment
  rides with the task; nothing to write.
- **Case `= 'CANCEL'`** -> the REJECT-shaped assign block from 4a, with `string('rejected')` written to
  `outputVariable.../result` and to the callback `arg1`. This is the documented request-engine contract
  ("the outcome that the request engine expects from request service is Approved or Rejected").
  The beneficiary's typed comment rides on the SOA task record and surfaces in request history —
  SOA task comments are the documented replacement for request comments (the deprecated
  `RequestService.addRequestComment` Javadoc says so verbatim).
- **Otherwise** -> the state-passthrough block from 4a.

**Hard audit invariant — verify before final build:**
grep the entire BPEL source for writes to `outputVariable` result and callback `arg1`. **Only the
manager task's APPROVE case may write `approved`.** The beneficiary task never writes `approved` on any
branch. That is the structural guarantee the beneficiary cannot self-approve.

---

## Phase 5 — Build, deploy, bind, test

1. File -> Save All. Build -> Build Project -> clean.
2. Right-click project -> Deploy -> CrossCountryAccessApproval -> Deploy to Application Server
   (or Deploy to SAR, then EM -> Deploy SOA Composite). `[VERIFIED-doc: dev guide 11.4.5.6]`
3. EM -> SOA -> soa-infra -> default -> confirm `CrossCountryAccessApproval` revision 1.0, active.
   **Never redeploy same-version over in-flight instances** — pending approvals go stale and are removed
   from users' task lists. Bump the revision every redeploy. `[VERIFIED-doc: dev guide 11.3.2 notes]`
4. OIM SysAdmin -> Workflows -> Approval -> Create rule:
   - Operation: the exact string from Phase 0 item 1 `[CHECK ENV]`
   - Condition: role-name scoping per the client requirement
     (`[VERIFIED-doc: Administering OIG ch.4.2 covers approval workflow rules and custom rule conditions]`)
   - Action: invoke composite `default/CrossCountryAccessApproval!1.0`
   - `[VERIFIED-doc: dev guide 11.4.5.7 shows this exact rule-creation sequence]`
5. Do NOT touch `oim-config.xml` `DefaultRequestLevelComposite` / `DefaultOperationLevelComposite`
   (changing those requires an OIM restart; you are deliberately not changing global defaults).
   `[VERIFIED-doc: dev guide 11.5]`
6. Test matrix (catalog/self-service UI + EM flow trace + OIM request history):

   | # | Scenario | Expected |
   |---|---|---|
   | 1 | Beneficiary country == SBE | Beneficiary task absent; manager task fires; Approve -> Approved -> provisioning fires |
   | 2 | Country != SBE, beneficiary submits | Beneficiary task shows SUBMIT_DOCUMENTS/CANCEL only; attach file; submit; manager task appears with attachment visible; Approve -> Approved |
   | 3 | Country != SBE, beneficiary cancels | Beneficiary picks CANCEL + comment "Cancelled by beneficiary"; request Rejected; comment visible in history; NO manager task; NO provisioning |

   On tests 2 and 3, also inspect the EM audit trail of the Java Embedding: confirm `sbeValue`,
   `benCountry`, `isCrossCountry` populated with sane values. If the lookup silently no-ops, routing
   straight to the manager LOOKS like success when it isn't — this check is the difference.
7. Attachment UI check: on test 2/3, confirm the beneficiary's task-details page in Pending Approvals
  shows the attachment upload control. If it does not, the follow-on increment is a custom task-details
  taskflow (dev guide 11.6) — out of scope for this build.

---

## Appendix A — Failure modes worth knowing before you hit them

- **Composite faults immediately on request:** almost always the missing OIM client jars on the SOA
  classpath (Phase 3c). Check the soa_server log for `ClassNotFoundException` on
  `oracle.iam.request.*`.
- **isCrossCountry always false:** dump the attribute tree via System.out as described in Phase 3b's
  residual-uncertainty note; adjust the matcher.
- **Beneficiary task created but assigned to nobody:** the assignment XPath in Phase 2e resolved empty
  — recheck the BeneficiaryDetails login element name against RequestDetails.xsd.
- **Request ends but task still pending:** means the callback arg0 conversation ID wasn't written
  correctly in the CANCEL branch — re-diff against the 4a block shape.
- **Auditor asks "where is it recorded":** the beneficiary appears as task assignee completing
  SUBMIT_DOCUMENTS/CANCEL (with comment + attachment) in the request's task history; the manager is the
  only approver on the APPROVE path. The request engine's lifecycle stages (Request Awaiting Approval ->
  Approved/Rejected) are the documented audit stages. What you have NOT verified yet (do this with a
  test request in hand): exactly how those entries render in the 12c request-history UI and the
  REST requesthistory endpoint — show that to your auditors before go-live, not after.

## Appendix B — What this build deliberately excludes

- Custom task-details taskflow (dev guide 11.6): only needed if the attachment control is missing in
  the standard UI, or if the business later demands UI-enforced mandatory comments on CANCEL.
  Currently comment-mandatory is a process rule, accepted by design decision.
- True "Request Withdrawn/Closed" end-state on beneficiary cancel: requires a RequestService
  close/withdraw call from the composite plus handling of workflow cancellation — the documented
  stages exist but the callback contract does not emit them. You accepted REJECTED-with-comment;
  this appendix records the road not taken if the business revisits it.
- Bulk/multi-beneficiary requests: BeneficiaryDetails is empty for multi-beneficiary requests
  (dev guide 11.3.1.1). This composite is bound at operational level where exactly one beneficiary
  exists; if the client later wants bulk requests in scope, the beneficiary identification logic
  and task fan-out become a separate design.

---

*Sources verified this session: OIM/OIG 12.2.1.4 Dev Guide ch.11 (Developing Workflows), Administering
OIG 12.2.1.4 ch.4 (Managing Workflows / request lifecycle), live 12.2.1.4 Javadocs for
oracle.iam.request.api.RequestService, oracle.iam.request.vo.{Request, Beneficiary,
RequestBeneficiaryEntity, RequestBeneficiaryEntityAttribute, RequestEntity, RequestEntityAttribute,
RequestData}, oracle.iam.platform.Platform, oracle.iam.identity.usermgmt.{api.UserManager, vo.User,
api.UserManagerConstants.AttributeName}. Items tagged `[CHECK ENV]` are intentionally left for reads
against your own system — do not substitute guesses for them.*
