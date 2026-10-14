# SACDCS — Task 2: Change Requests and Traceability Matrix

## 1. Version-controlled change log
| Version | CR | Req. ID | Status | Old text | New text | Reason |
|---------|----|---------|--------|----------|----------|--------|
| v1.0 | — | all | Baseline | — | Baseline approved (Task1_Requirements.md) | Initial release |
| v1.1 | CR-01 | FR-04 | **Modified** | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode within 3 seconds when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. | Medicine can spoil and the battery can overheat within seconds; the response needs a deadline and precise triggers. |
| v1.2 | CR-02 | NFR-04 | **Modified** | The system shall support communication with multiple robots simultaneously. | The system shall support at least 20 simultaneously connected robots. | The pilot serves 20 clinics; "multiple" cannot be tested. |
| v1.3 | CR-03 | NFR-02 | **Modified** | Only authenticated operators shall be permitted to issue robot commands. | The system shall require authenticated and role-authorised operators before accepting any robot command. | Identity alone is not enough; an operator must also be allowed to issue that specific command. |

Suggested git commits: "CR-01: FR-04 Safe Mode within 3 s (v1.1)", "CR-02: NFR-04 at least 20 robots (v1.2)", "CR-03: NFR-02 role-based authorisation (v1.3)".

### Requirements impacted (not rewritten, but must be re-verified)
| Changed by | Impacted requirement | Why |
|------------|----------------------|-----|
| CR-01 | FR-08 | Telemetry and sensor sampling must be fast enough to detect the condition within 3 s. |
| CR-01 | FR-06, SR-03 | The emergency level (below 10%) must be separated from the critical-low level (20%); thermal triggers overlap with chamber protection. |
| CR-02 | NFR-01, NFR-03, FR-10 | The 5 s limit, event buffering and logging must still hold with 20 robots. |
| CR-03 | FR-01, FR-09, FR-10 | A new reject reason (ROLE_NOT_AUTHORISED) is needed, and the role must be logged. Safe Mode recovery (T17, T18) also needs a role check. |

## 2. Why the new wording is more measurable or testable
- **CR-01:** The old text had no deadline and "critical condition" was undefined, so a late response could never fail a test. The new text has a start event (threshold exceeded or capacity below the emergency level), an end event (state SAFE_MODE) and a limit (3 s). A tester can inject a value, run a stopwatch and give a pass/fail result. The two triggers can also be tested separately.
- **CR-02:** "Multiple" is satisfied by 2 robots, so it can never fail. "At least 20 simultaneously connected" gives a number and a test (20 simulated robots connected at the same time). It matches the pilot size.
- **CR-03:** "Authenticated" tests only identity. The new text adds authorisation and an ordering rule (before accepting any command). This allows a test matrix of operator role versus command type, including negative cases (authenticated Observer sends MOVE, which must be rejected).

## 3. Test cases (traceable to: command received, authenticated, executed, status reported, logged)
| TC ID | Scenario step | Test input / action | Expected result | Covers |
|-------|---------------|---------------------|-----------------|--------|
| TC-01 | Command received | Authenticated operator sends a valid MOVE command, sequence ID 101 | Robot sends RECEIVED for ID 101; entry appears in log | FR-01, FR-09 |
| TC-02 | Authenticated | Send command with expired token, forged token and no token | All rejected (AUTH_FAILED); no state change; attempts logged | NFR-02 |
| TC-03 | Authenticated, then authorised | Observer sends MOVE; Commander sends the same MOVE | Observer rejected (ROLE_NOT_AUTHORISED); Commander accepted | NFR-02, FR-01 |
| TC-04 | Executed | Valid delivery request in IDLE; retransmit same sequence ID 3 times | Robot enters NAVIGATING once; duplicates acknowledged but not re-executed | FR-01, FR-02 |
| TC-05 | Status reported | Run command of TC-04 | Mission Control receives RECEIVED, ACCEPTED, EXECUTING, COMPLETED/FAILED in order, each with command ID | FR-09 |
| TC-06 | Logged | Inspect log after TC-03 and TC-04 | Every command and rejected attempt has timestamp, operator ID, role, robot ID, command ID | FR-10 |
| TC-07 | Executed (timing) | Send 100 valid commands at nominal link delay | At least 95 complete within 5 s of receipt | NFR-01 |
| TC-08 | Command received (invalid) | Send command with out-of-range parameter and with malformed field | Rejected with reason code; state unchanged; logged | FR-01 |
| TC-09 | Executed (obstacle) | Inject obstacle in NAVIGATING, then remove it | NAVIGATING to AVOIDING_OBSTACLE, then back to NAVIGATING toward same destination | FR-03 |
| TC-10 | Executed (interlock) | Force DELIVER from IDLE; force DELIVER from AVOIDING_OBSTACLE | Both rejected; state unchanged; violation logged | SR-04, FR-05 |
| TC-11 | Executed (battery) | Drop battery to 19% in NAVIGATING, then in AVOIDING_OBSTACLE, then in DELIVERING | NAVIGATING goes to RETURNING; in the other two the flag is latched and RETURNING follows after obstacle cleared / delivery done | FR-06 |
| TC-12 | Executed (Safe Mode) | Inject battery temp above critical; chamber temp above critical; capacity below 10%; measure time. Then recover as unauthorised and authorised operator | SAFE_MODE within 3 s in each case; alert and log; recovery only by authorised operator | FR-04, NFR-02 |
| TC-13 | Concurrency | Simulate 20 robots, each sending telemetry every 5 s and commands | All 20 stay connected, nothing dropped, NFR-01 still met | NFR-04 |
| TC-14 | Link | Cut link 40 s in NAVIGATING, restore; separately cut 130 s | LINK_LOST at 30 s; events buffered; on restore all delivered in order; at 130 s robot goes to RETURNING | NFR-03, SR-02 |
| TC-15 | Chamber | Push chamber temp out of range: (a) cooling works, (b) cooling disabled | (a) recovery declared only after sensor readings stay in range; (b) protection mode after 120 s, light reduced, alert within 5 s | FR-07, SR-03 |
| TC-16 | Railway | Simulate: barrier closed; train approaching; ARLCCS link lost; sensor failure; barrier open and no train | Robot holds in every unsafe case; crosses only when barrier open, no train and status fresh | SR-01 |
| TC-17 | Door interlock | Open chamber door while NAVIGATING; trigger DestinationReached with door open | Robot stops, conservation suspended, transition T4 blocked | SR-04, FR-05 |
| TC-18 | Status reported (telemetry) | Observe telemetry for 5 min and on state changes | All six data items present; interval at most 5 s; immediate report on state change | FR-08 |
| TC-19 | Executed (delivery) | Run complete delivery | Sequence NAVIGATING, DELIVERING, RETURNING, IDLE | FR-05 |
| TC-20 | Usability | 10 operators issue a standard command | At least 9 finish in at most 3 steps and 60 s | NFR-05 |

## 4. Which test cases must be re-run after each CR
| TC | CR-01 | CR-02 | CR-03 |
|----|:-----:|:-----:|:-----:|
| TC-01 | | | ✔ |
| TC-02 | | | ✔ |
| TC-03 | | | ✔ |
| TC-04 | | | ✔ |
| TC-05 | | ✔ | |
| TC-06 | | ✔ | ✔ |
| TC-07 | | ✔ | ✔ |
| TC-08 | | | ✔ |
| TC-09 | | | |
| TC-10 | | | |
| TC-11 | ✔ | | |
| TC-12 | ✔ | | ✔ |
| TC-13 | | ✔ | |
| TC-14 | ✔ | ✔ | |
| TC-15 | ✔ | | |
| TC-16 | | | |
| TC-17 | | | |
| TC-18 | ✔ | ✔ | |
| TC-19 | | | |
| TC-20 | | | ✔ |

## 5. Multi-layer RTM (current baseline v1.3)
Requirement, then State/Transition (Task 3), Operation (Task 4), Constraint (Task 5), Test Case, Risk Level.

| Req. ID | State / Transition | Operation | Constraint | Test Case | Risk |
|---------|--------------------|-----------|------------|-----------|------|
| FR-01 | T1 | OP-C3, OP-C4, OP-C5 | C12 | TC-01, TC-04, TC-08 | High |
| FR-02 | T1 | OP-C4, OP-C10 | C11, C14 | TC-04, TC-10 | High |
| FR-03 | T2, T3, T8, T9, T24, T25 | OP-C11 | C11 | TC-09 | High |
| FR-04 (CR-01) | T11 to T18, state S6 | OP-C8 | C10 | TC-12 | Critical |
| FR-05 | T4, T5, T6 | OP-C13, OP-C14 | C8, C11 | TC-10, TC-19 | High |
| FR-06 | T7, T8, T9, T10 | OP-C14 | C14 | TC-11 | High |
| FR-07 | Chamber CH3 (Task 4) | OP-04, OP-05, OP-06 | C10 | TC-15 | Critical |
| FR-08 | All states | OP-C7 | C9, C10 | TC-18 | High |
| FR-09 | T1 | OP-C5 | C12 | TC-05 | Medium |
| FR-10 | All states | OP-C6 | C13 | TC-06 | High |
| NFR-01 | T1 | OP-C1 to OP-C4 | C12 | TC-07 | Medium |
| NFR-02 (CR-03) | T1, T17, T18 | OP-C1, OP-C2 | C12 | TC-02, TC-03 | Critical |
| NFR-03 | T19 to T23, S7 | OP-C6, OP-C9 | C5 | TC-14 | High |
| NFR-04 (CR-02) | All states | OP-C5, OP-C7 | none (capacity requirement) | TC-13 | High |
| NFR-05 | none | none | none | TC-20 | Low |
| SR-01 | NAVIGATING (S2) | OP-C12 | C1 to C7 | TC-16 | Critical |
| SR-02 | T19 to T26, S7 | OP-C9 | C5 | TC-14 | High |
| SR-03 | T12, T14, chamber CH3 to CH6 | OP-06, OP-07, OP-08, OP-09 | C10 | TC-15 | Critical |
| SR-04 | X1, X2 (blocked), T4 | OP-C13 | C8, C9, C11 | TC-10, TC-17 | Critical |
