# SACDCS — Task 3: State Modelling and Transition Verification

## 1. States
| State ID | State | Meaning | Entry condition | Exit condition |
|----------|-------|---------|-----------------|----------------|
| S1 | IDLE | Robot is at the warehouse, powered on, waiting for a request. | Power-on self-test passed; WarehouseReached (T6); recovery approved at warehouse (T17) | DeliveryRequest (T1) or EmergencyCondition (T11) |
| S2 | NAVIGATING | Robot is moving toward the destination. | T1, T3, T21 | ObstacleDetected, DestinationReached, CriticalBattery, EmergencyCondition, LinkLost |
| S3 | AVOIDING_OBSTACLE | Normal navigation is suspended while the robot handles an obstacle. | ObstacleDetected (T2) | ObstacleCleared (T3, T9, T25) or EmergencyCondition (T13) |
| S4 | DELIVERING | Robot is at the destination and performing the handover. | DestinationReached with door closed (T4) | DeliveryCompleted (T5) or EmergencyCondition (T14) |
| S5 | RETURNING | Robot is travelling back to the warehouse. | T5, T7, T9, T18, T22, T23 | WarehouseReached (T6), EmergencyCondition (T15), LinkLost (T20) |
| S6 | SAFE_MODE | Robot is stopped, non-essential functions off, chamber protected, waiting for operator. | EmergencyCondition (T11 to T16) | RecoveryApproved (T17, T18) |
| S7 | LINK_LOST | Robot has lost contact with Mission Control and holds a safe position, buffering events. | LinkLost (T19, T20, T25) | LinkRestored (T21, T22), LinkTimeout (T23), EmergencyCondition (T16) |

## 2. Events and conditions
| Event ID | Event | Meaning |
|----------|-------|---------|
| E1 | DeliveryRequest | A valid, authorised delivery request is received. |
| E2 | ObstacleDetected | Sensors detect an obstacle on the path. |
| E3 | ObstacleCleared | The obstacle has been avoided. |
| E4 | DestinationReached | Robot has arrived at the destination. |
| E5 | DeliveryCompleted | Handover confirmed. |
| E6 | WarehouseReached | Robot has arrived at the warehouse. |
| E7 | CriticalBattery | Battery capacity at or below 20% (critical-low). |
| E8 | EmergencyCondition | Battery capacity below 10%, OR battery temperature above critical, OR chamber temperature above critical, OR critical fault (CR-01). |
| E9 | LinkLost | No valid heartbeat for more than 30 s. |
| E10 | LinkRestored | Valid heartbeat received and status re-synchronised. |
| E11 | LinkTimeout | Link has been lost for more than 120 s. |
| E12 | RecoveryApproved | Authenticated, role-authorised operator approves recovery and conditions are normal. |

Event priority when several occur together: E8 (Emergency) > E7 (CriticalBattery) > E9 (LinkLost) > E2 (ObstacleDetected) > all others.

**State versus event:** A state is a situation that lasts over time (the robot *is* navigating). It has entry and exit conditions and can be observed at any moment. An event is an instantaneous occurrence (an obstacle *is detected*) that may trigger a change of state. A state answers "what is the robot doing now?", an event answers "what just happened?".

## 3. State Transition Table (Current State + Event [guard] → Next State)
| T-ID | Current state | Event [guard] | Next state | Req. ID | Valid? | Verification evidence |
|------|---------------|---------------|------------|---------|--------|-----------------------|
| T1 | IDLE | E1 DeliveryRequest [valid, authorised, no CriticalBattery, chamber ready] | NAVIGATING | FR-01, FR-02 | Valid | TC-04; IDLE has no other path to NAVIGATING |
| T2 | NAVIGATING | E2 ObstacleDetected | AVOIDING_OBSTACLE | FR-03 | Valid | TC-09 |
| T3 | AVOIDING_OBSTACLE | E3 ObstacleCleared [no latched battery or link flag] | NAVIGATING | FR-03 | Valid | TC-09 |
| T4 | NAVIGATING | E4 DestinationReached [door closed] | DELIVERING | FR-05, SR-04 | Valid | TC-19, TC-17 |
| T5 | DELIVERING | E5 DeliveryCompleted | RETURNING | FR-05 | Valid | TC-19 |
| T6 | RETURNING | E6 WarehouseReached | IDLE | FR-05 | Valid | TC-19 |
| T7 | NAVIGATING | E7 CriticalBattery | RETURNING | FR-06 | Valid | TC-11 |
| T8 | AVOIDING_OBSTACLE | E7 CriticalBattery | AVOIDING_OBSTACLE (flag latched) | FR-06 | Valid (added: was missing) | TC-11 |
| T9 | AVOIDING_OBSTACLE | E3 ObstacleCleared [CriticalBattery latched] | RETURNING | FR-06 | Valid (added) | TC-11 |
| T10 | DELIVERING | E7 CriticalBattery | DELIVERING (flag latched, then T5 leads to RETURNING) | FR-06 | Valid (added: was missing) | TC-11 |
| T11 | IDLE | E8 EmergencyCondition | SAFE_MODE | FR-04 | Valid | TC-12 |
| T12 | NAVIGATING | E8 EmergencyCondition | SAFE_MODE | FR-04 | Valid | TC-12 |
| T13 | AVOIDING_OBSTACLE | E8 EmergencyCondition | SAFE_MODE | FR-04 | Valid | TC-12 |
| T14 | DELIVERING | E8 EmergencyCondition | SAFE_MODE | FR-04 | Valid | TC-12 |
| T15 | RETURNING | E8 EmergencyCondition | SAFE_MODE | FR-04 | Valid | TC-12 |
| T16 | LINK_LOST | E8 EmergencyCondition | SAFE_MODE | FR-04 | Valid | TC-12 |
| T17 | SAFE_MODE | E12 RecoveryApproved [at warehouse] | IDLE | FR-04, NFR-02 | Valid | TC-12 |
| T18 | SAFE_MODE | E12 RecoveryApproved [not at warehouse] | RETURNING | FR-04, NFR-02 | Valid | TC-12 |
| T19 | NAVIGATING | E9 LinkLost | LINK_LOST | SR-02 | Valid | TC-14 |
| T20 | RETURNING | E9 LinkLost | LINK_LOST | SR-02 | Valid | TC-14 |
| T21 | LINK_LOST | E10 LinkRestored [came from NAVIGATING, no CriticalBattery] | NAVIGATING | SR-02 | Valid | TC-14 |
| T22 | LINK_LOST | E10 LinkRestored [came from RETURNING, or CriticalBattery] | RETURNING | SR-02 | Valid | TC-14 |
| T23 | LINK_LOST | E11 LinkTimeout | RETURNING | SR-02 | Valid | TC-14 |
| T24 | AVOIDING_OBSTACLE | E9 LinkLost | AVOIDING_OBSTACLE (flag latched) | SR-02 | Valid (added) | TC-14 |
| T25 | AVOIDING_OBSTACLE | E3 ObstacleCleared [LinkLost latched, no CriticalBattery] | LINK_LOST | SR-02 | Valid (added) | TC-14 |
| T26 | DELIVERING | E9 LinkLost | DELIVERING (flag latched; after T5 the link rule T20 applies) | SR-02 | Valid (added) | TC-14 |

### Invalid or unsafe transitions (must be impossible)
| X-ID | Attempted transition | Valid? | Requirement violated | Verification evidence |
|------|----------------------|--------|----------------------|-----------------------|
| X1 | IDLE to DELIVERING | **Invalid** | FR-02, FR-05, SR-04 | No such row in table; TC-10 shows DELIVER command in IDLE is rejected |
| X2 | AVOIDING_OBSTACLE to DELIVERING | **Invalid** | FR-03, SR-04 | No such row; TC-10 |
| X3 | SAFE_MODE to NAVIGATING | **Invalid** | FR-04, NFR-02 (skips recovery approval) | No such row; TC-12 |
| X4 | LINK_LOST to DELIVERING | **Invalid** | SR-02, SR-04 | No such row; TC-14 |
| X5 | IDLE + DestinationReached (spurious) | Ignored, no state change | FR-05 | Event logged as anomaly; TC-10 |

## 4. Extension: Safe Mode and Link Lost
| Existing state | May enter SAFE_MODE? | May enter LINK_LOST? | Decision and reason |
|----------------|:--------------------:|:--------------------:|---------------------|
| IDLE | Yes (T11) | No | Emergency applies everywhere. In IDLE no commands arrive during a link loss, so only a flag is set. |
| NAVIGATING | Yes (T12) | Yes (T19) | Robot is moving, so it must stop safely if contact is lost. |
| AVOIDING_OBSTACLE | Yes (T13) | Deferred (T24, T25) | Stopping mid-manoeuvre could be unsafe. The flag is latched and applied after ObstacleCleared. |
| DELIVERING | Yes (T14) | Deferred (T26) | Handover is local and should finish. The flag is applied after delivery. |
| RETURNING | Yes (T15) | Yes (T20) | Same as NAVIGATING. |
| LINK_LOST | Yes (T16) | n/a | Emergency overrides link loss. |

## 5. Verifier investigation
**(a) Can IDLE to DELIVERING happen?** It must not. It violates FR-02 (IDLE may only start navigation), FR-05 (delivery starts only after reaching the destination from NAVIGATING) and SR-04. The robot would start a handover with no destination reached, so the package could be released at the warehouse or at the wrong place.

**(b) What if AVOIDING_OBSTACLE has no way back?** The model has a **deadlock**. The robot stays in AVOIDING_OBSTACLE forever and can never deliver. This violates FR-03. The chamber keeps running with a stationary robot until the battery fails, so SR-03 (cold-chain protection) is also at risk. Fix: T3 (and T9, T25).

**(c) Can AVOIDING_OBSTACLE go directly to DELIVERING?** No. It violates SR-04 and FR-03. ObstacleCleared does not mean the destination was reached. The only legal path is T3 (back to NAVIGATING) and then T4.

**(d) CriticalBattery in AVOIDING_OBSTACLE or DELIVERING.** In the baseline model there is **no transition**, so the event is ignored. The robot continues, drains the battery and is stranded with the medicine, which breaks the cold chain and violates FR-06. Fix: T8 and T10 latch the flag, and T9 or T5 send the robot to RETURNING at the first safe moment.

### Other defects found by the verifier
| # | Defect | Fix |
|---|--------|-----|
| D1 | Safe Mode had no exit | T17 and T18 with operator approval; X3 forbids a direct jump back to NAVIGATING |
| D2 | CriticalBattery and LinkLost can occur together | Priority order and guards on T3, T9, T25 |
| D3 | Emergency was undefined in some states | T11 to T16 cover every state |
