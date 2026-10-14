# SACDCS — Task 5: Constraints, Formalisation and Violations (Railway Crossing)

## Variables used (all observable by a test or a runtime monitor)
| Variable | Meaning |
|----------|---------|
| Train_Present | Train detected inside the crossing zone |
| Train_Approaching | Train detected in the approach zone |
| Barrier_Open | Barrier position sensor shows fully raised |
| Warning_Active, Alarm_Active | Warning lights on, audible alarm on |
| Crossing_Clear_Confirmed | Flag set by the safety-monitoring unit after all sensors agree the crossing is empty |
| Conflicting_Readings | Two train-detection sensors disagree |
| Sensor_Failure | A train-detection sensor missed its heartbeat or reports an invalid value |
| ARLCCS_Link_Lost | Robot has no ARLCCS status newer than 5 s |
| Barrier_Failure | Barrier did not reach the commanded position in 15 s |
| Robot_Enter_Crossing | Robot passes the holding line |
| Robot_Stopped, Operator_Alert | Robot speed is 0; alert sent to operator |
| Door_Open | Chamber door sensor shows open |
| Enter_DELIVERING | Robot is changing into state DELIVERING |
| Previous_State | State before the current one |
| Emergency_Condition | Event E8 from Task 3 |
| Battery_Critical | Battery at or below 20% |
| Command_Accepted, Authenticated, Role_Authorised | Command was accepted; session valid; role allowed for this command |
| Critical_Event, Event_Logged | A critical event occurred; it was logged with timestamp and operator ID |

## 1. Constraints
| ID | Constraint (simple English) | Why it is necessary |
|----|-----------------------------|---------------------|
| C1 | The barrier must not be open while a train is in the crossing. | Road traffic or the robot could enter while the train passes. |
| C2 | Warning lights and alarm must be on whenever a train is approaching or present. | Gives road users and the robot time to stop. |
| C3 | The barrier may be open only if the safety monitor has confirmed the crossing is clear. | The rear of the train or a second train may still be there. |
| C4 | The robot may enter the crossing only if the barrier is open and no train is approaching. | A robot on the track is hit by the train. |
| C5 | If a sensor has failed or the ARLCCS link is lost, the robot must not enter the crossing. | Without reliable data safety cannot be proven, so the system must fail safe. |
| C6 | If the barrier fails while a train is approaching, the alarm must stay on, the operator must be alerted and the robot must stop. | A failed barrier is an immediate collision hazard needing human action. |
| C7 | If sensor readings conflict, the crossing must not be confirmed clear. | A faulty sensor must not be allowed to open the barrier. |
| C8 | The robot must not enter DELIVERING while the chamber door is open. (robot and chamber) | An open door during handover entry exposes the batch and makes the state unverifiable. |
| C9 | The chamber door must be closed whenever the robot is NAVIGATING, AVOIDING_OBSTACLE or RETURNING. (robot and chamber) | Vibration and movement with an open door damage the batch and break the cold chain. |
| C10 | An emergency condition (battery or chamber temperature above critical, or battery below emergency level) must put the robot in SAFE_MODE within 3 s. | CR-01: medicine and battery are at risk within seconds. |
| C11 | The robot must enter DELIVERING only from NAVIGATING (never from IDLE or AVOIDING_OBSTACLE). | Delivery without a confirmed arrival, or during obstacle handling, is unsafe. |
| C12 | A command may be accepted only if the operator is authenticated and role-authorised. | CR-03: prevents unauthorised commands. |
| C13 | Every critical event must be logged with timestamp and operator ID. | Needed for investigation. |
| C14 | A robot in NAVIGATING with critical-low battery must move to RETURNING. | Prevents the robot being stranded with medicine. |

## 2. Formalisation
| F-ID | Constraint | Formal expression |
|------|------------|-------------------|
| F1 | C1 | Train_Present → ¬Barrier_Open |
| F2 | C2 | (Train_Approaching ∨ Train_Present) → (Warning_Active ∧ Alarm_Active) |
| F3 | C3 | Barrier_Open → Crossing_Clear_Confirmed |
| F4 | C4 | Robot_Enter_Crossing → (Barrier_Open ∧ ¬Train_Approaching) |
| F5 | C5 | (Sensor_Failure ∨ ARLCCS_Link_Lost) → ¬Robot_Enter_Crossing |
| F6 | C6 | (Barrier_Failure ∧ Train_Approaching) → (Alarm_Active ∧ Operator_Alert ∧ Robot_Stopped) |
| F7 | C7 | Conflicting_Readings → ¬Crossing_Clear_Confirmed |
| F8 | C8 | Enter_DELIVERING → ¬Door_Open |
| F9 | C9 | (Robot_State = NAVIGATING ∨ Robot_State = AVOIDING_OBSTACLE ∨ Robot_State = RETURNING) → ¬Door_Open |
| F10 | C10 | (Emergency_Condition ∧ elapsed_time ≥ 3 s) → Robot_State = SAFE_MODE |
| F11 | C11 | Enter_DELIVERING → (Previous_State = NAVIGATING) |
| F12 | C12 | Command_Accepted → (Authenticated ∧ Role_Authorised) |
| F13 | C13 | Critical_Event → Event_Logged |
| F14 | C14 | (Battery_Critical ∧ Robot_State = NAVIGATING) → Next_State = RETURNING |

## 3 and 4. Violations, detection and expected response
| V-ID | Violates | Variable values | What went wrong | How the violation is detected | Expected response |
|------|----------|-----------------|-----------------|-------------------------------|-------------------|
| V1 | F1 | Train_Present = TRUE, Barrier_Open = TRUE | Barrier raised while a train occupies the crossing (actuator or control fault). | ARLCCS safety monitor evaluates F1 every 100 ms from the occupancy sensors and barrier position sensor. | Barrier closed, alarm on, robot stop (HOLD sent), operator alert, event logged. |
| V2 | F2 | Train_Approaching = TRUE, Warning_Active = FALSE, Alarm_Active = TRUE | Warning lights failed to light. | Lamp-current feedback shows no light while the approach flag is TRUE. | Re-trigger warning, barrier closed, operator alert, event logged. |
| V3 | F3 | Barrier_Open = TRUE, Crossing_Clear_Confirmed = FALSE (rear-of-train sensor still occupied) | Barrier opened before the whole train left. | Monitor sees open barrier with the clear flag FALSE. | Barrier closed, alarm on, robot stop, operator alert, event logged. |
| V4 | F4 | Robot_Enter_Crossing = TRUE, Barrier_Open = TRUE, Train_Approaching = TRUE (robot used a 12 s old status) | Robot entered on stale data while a train approached. | Robot checks status age against 5 s; ARLCCS sees robot on the holding line plus train approaching. | Robot stop (or exits at safe speed if already inside), barrier closed, alarm, Safe Mode, operator alert, event logged. |
| V5 | F5 | ARLCCS_Link_Lost = TRUE (status age 12 s), Robot_Enter_Crossing = TRUE. Variant: Sensor_Failure = TRUE | Robot crossed without live safety data. | Robot heartbeat check on ARLCCS status; sensor heartbeat monitor for the sensor variant. | Robot stop at holding line, barrier forced closed, operator alert, event logged. |
| V6 | F6 | Barrier_Failure = TRUE, Train_Approaching = TRUE, Alarm_Active = TRUE, Operator_Alert = FALSE, Robot_Stopped = FALSE | Barrier stuck and nobody told, robot still moving. | Barrier travel timer (15 s) expires; monitor finds alert flag FALSE. | Operator alert (escalated), alarm kept on, robot stop, event logged. |
| V7 | F7 | Conflicting_Readings = TRUE (sensor A occupied, sensor B clear), Crossing_Clear_Confirmed = TRUE | System picked the "clear" reading. | Cross-check of the two sensors against the clear flag. | Clearance revoked, barrier kept or returned to closed, alarm, operator alert, event logged. |
| V8 | F8 | Enter_DELIVERING = TRUE, Door_Open = TRUE | Robot began delivery with the door open. | Guard on transition T4 reads the door sensor. | Transition blocked, robot stays NAVIGATING, door re-latch attempt, operator alert, event logged. If chamber temperature rises, protection mode (OP-07). |
| V9 | F9 | Robot_State = NAVIGATING, Door_Open = TRUE | Door came open while the robot was moving. | Chamber door sensor combined with robot state in telemetry (OP-12). | Robot stop, conservation suspended, operator alert, event logged. |
| V10 | F10 | Battery temperature = 62 °C (critical 55 °C), elapsed_time = 4.5 s, Robot_State = NAVIGATING | Safe Mode entered too late. | Timer from threshold crossing to SAFE_MODE exceeds 3 s. | Force Safe Mode at once, operator alert, event logged, timing defect raised. |
| V11 | F11 | Enter_DELIVERING = TRUE, Previous_State = AVOIDING_OBSTACLE (or IDLE) | Delivery started from a forbidden state. | State-machine monitor compares Previous_State at entry. | Transition rejected, robot stays in current state, operator alert, event logged. |
| V12 | F12 | Command_Accepted = TRUE, Authenticated = TRUE, Role_Authorised = FALSE (Observer sent MOVE) | Command accepted without role check. | Runtime audit compares role against the permission table on every accepted command. | Command cancelled, robot holds (robot stop), security alert to operator, session revoked, event logged. |
| V13 | F13 | Critical_Event = Safe Mode entry, Event_Logged = FALSE | Event missing from log. | Log sequence number gap against the event counter. | Event rewritten from robot buffer, operator alert for audit gap, event logged. |
| V14 | F14 | Battery_Critical = TRUE (battery 14%), Robot_State = NAVIGATING for 10 s | Robot ignored low battery. | Monitor sees battery below 20% with state unchanged after 1 s. | Force RETURNING, operator alert, event logged. |

## 5. Coverage check
- Two constraints linking robot and chamber: C8 and C9.
- Constraint for sensor failure and communication loss: C5 (also C6 for barrier failure and C7 for incorrect readings).
- Every constraint is stated over observed values, so a test or a monitor can decide pass or fail.
