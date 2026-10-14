# SACDCS — Task 4: Operation Identification and Operation Schema (Conservation Chamber)

## 1. Chamber operations (17)
| OP_Id | Operation | Purpose |
|-------|-----------|---------|
| OP-01 | PerformSelfCheck | Verify that all essential sensors and control devices work before monitoring starts. |
| OP-02 | LoadBatchProfile | Record the batch ID and its required temperature and humidity limits. |
| OP-03 | StartConservation | Begin active conservation only when the door is closed and the profile is loaded. |
| OP-04 | CorrectTemperature | Drive temperature back into the batch limits. |
| OP-05 | CorrectHumidity | Drive humidity back into the batch limits. |
| OP-06 | VerifyRecovery | Confirm from sensor readings that the value is really back in range. |
| OP-07 | EnterProtectionMode | Prioritise batch protection when correction fails within the recovery period. |
| OP-08 | ReduceLightExposure | Lower light on the batch during protection. |
| OP-09 | RaiseOperatorAlert | Notify the operator of a protection event. |
| OP-10 | SuspendRiskyActivities | Stop activities that could harm the batch when significant vibration is detected. |
| OP-11 | VerifyVibrationStability | Confirm vibration stayed below threshold for the full stabilisation period. |
| OP-12 | SuspendConservationOnDoorOpen | Stop normal conservation as soon as the door opens. |
| OP-13 | VerifyConditionsBeforeResume | Check conditions and sensors before resuming after the door is closed again. |
| OP-14 | SwitchToEmergencyPower | Move the chamber to emergency power after main power loss. |
| OP-15 | RecordPowerIncident | Log the incident when no emergency power is available. |
| OP-16 | PerformSafeShutdown | Shut the chamber down safely when no power source is available. |
| OP-17 | AuthoriseBatchRemoval | Allow removal only when the chamber is safe and no protection response is active. |

## 2. Operation Schema (8 operations)

### OP-04 CorrectTemperature
- **Description:** Uses heating or cooling to bring temperature back into the batch limits.
- **Inputs:** temperature reading T, batch limits [Tmin, Tmax], recovery period (120 s)
- **Outputs:** actuator command (heat or cool level), correction-started flag
- **Reads:** temperature sensors, batch profile, door status, power status
- **Changes:** thermal actuator setpoint, recovery timer started, chamber status = CORRECTING
- **Sends/Alerts:** log event TEMP_OUT_OF_RANGE; telemetry update to Mission Control
- **Assumes (precondition):** chamber in CONSERVATION_ACTIVE, profile loaded, door closed, actuator healthy, T outside limits
- **Result (postcondition):** correction command issued and timer running. Temperature is **not** assumed to be in range; OP-06 must confirm it.

### OP-06 VerifyRecovery
- **Description:** Confirms from sensors that the corrected value stays in range for the hold time (10 s).
- **Inputs:** sensor readings, batch limits, hold time, remaining recovery time
- **Outputs:** RECOVERED, NOT_RECOVERED or TIMEOUT
- **Reads:** temperature and humidity sensors, recovery timer
- **Changes:** chamber status goes from CORRECTING back to CONSERVATION_ACTIVE if recovered; timer cleared
- **Sends/Alerts:** log RECOVERY_CONFIRMED; on TIMEOUT it triggers OP-07
- **Assumes:** OP-04 or OP-05 has been issued, timer running, sensors valid
- **Result:** chamber returns to normal only if readings stayed within limits for the hold time; otherwise TIMEOUT leads to protection mode.

### OP-07 EnterProtectionMode
- **Description:** Puts protection of the batch above normal conservation.
- **Inputs:** TIMEOUT from OP-06, current readings
- **Outputs:** protection flag, light-reduction command, additional-control commands
- **Reads:** sensors, batch profile, power status
- **Changes:** chamber state CONSERVATION_ACTIVE to PROTECTION_MODE; light level reduced; additional controls on; normal conservation routines disabled
- **Sends/Alerts:** operator alert within 5 s; critical event logged; telemetry to Mission Control; raises EmergencyCondition (E8) to the robot if critical temperature is exceeded
- **Assumes:** recovery period expired without verified recovery
- **Result:** chamber in PROTECTION_MODE, light reduced, extra controls active, alert sent and logged.

### OP-11 VerifyVibrationStability
- **Description:** Checks that vibration stays under the threshold for the whole stabilisation period (30 s).
- **Inputs:** vibration readings, threshold, stabilisation period
- **Outputs:** STABLE or NOT_STABLE
- **Reads:** vibration sensor, stabilisation timer
- **Changes:** timer (reset if threshold exceeded); chamber state VIBRATION_RESPONSE to CONSERVATION_ACTIVE once stable
- **Sends/Alerts:** log VIBRATION_STABLE; telemetry update
- **Assumes:** chamber in VIBRATION_RESPONSE, vibration sensor healthy
- **Result:** normal conservation resumes only after 30 s continuously below threshold. Vibration merely stopping is not enough.

### OP-12 SuspendConservationOnDoorOpen
- **Description:** Immediately halts normal environmental control when the door opens.
- **Inputs:** door status = OPEN, chamber state
- **Outputs:** suspend command to actuators, door-open flag
- **Reads:** door sensor, chamber state
- **Changes:** chamber state CONSERVATION_ACTIVE to DOOR_OPEN_SUSPENDED; actuators set to hold; conservation timers paused
- **Sends/Alerts:** log with timestamp; telemetry door status; alert to robot logic if the robot is moving (constraint C9)
- **Assumes:** chamber in CONSERVATION_ACTIVE
- **Result:** within 1 s no normal environmental control runs while the door is open.

### OP-13 VerifyConditionsBeforeResume
- **Description:** After the door is closed again, checks conditions and sensors before resuming.
- **Inputs:** door status = CLOSED and latched, temperature, humidity, vibration readings, sensor self-test result
- **Outputs:** RESUME, REMAIN_SUSPENDED or CORRECT
- **Reads:** door sensor, all environmental sensors, batch limits
- **Changes:** state DOOR_OPEN_SUSPENDED to CONSERVATION_ACTIVE if all pass; to correction (OP-04, OP-05) if a value is out of range
- **Sends/Alerts:** log result; alert if a sensor fails the self-test
- **Assumes:** chamber in DOOR_OPEN_SUSPENDED and door closed
- **Result:** conservation resumes only if every check passes. It never resumes automatically on door close.

### OP-14 SwitchToEmergencyPower
- **Description:** Moves the chamber load to the emergency source after main power is lost.
- **Inputs:** main power status = LOST, emergency source status
- **Outputs:** transfer command, power-source status
- **Reads:** main power sensor, emergency source health, bus voltage
- **Changes:** active power source set to EMERGENCY; non-essential loads shed
- **Sends/Alerts:** log POWER_SWITCH; operator alert; telemetry power status
- **Assumes:** main power lost and emergency source reported healthy
- **Result:** chamber is on emergency power, **confirmed by bus-voltage reading within tolerance**. If not confirmed, the source is treated as unavailable and OP-15 and OP-16 follow.

### OP-17 AuthoriseBatchRemoval
- **Description:** Grants batch removal only in a safe chamber condition.
- **Inputs:** removal request, operator ID and role, chamber state, protection flag
- **Outputs:** GRANTED or DENIED, door-unlock command
- **Reads:** chamber state, protection flag, active alerts, operator role
- **Changes:** door lock (unlocked if granted); batch record status = REMOVED
- **Sends/Alerts:** log with timestamp and operator ID; alert if denied
- **Assumes:** operator authenticated and authorised, batch present
- **Result:** removal permitted only if the chamber is in a normal safe state and no protection response is active.

## 3. Operations that change the chamber state (linked to the Task 3 robot model)
Chamber states: CH1 SELF_CHECK, CH2 MONITORING, CH3 CONSERVATION_ACTIVE, CH4 DOOR_OPEN_SUSPENDED, CH5 VIBRATION_RESPONSE, CH6 PROTECTION_MODE, CH7 SAFE_SHUTDOWN.

| Operation | Chamber state change | Link to robot state model (Task 3) |
|-----------|----------------------|------------------------------------|
| OP-12 SuspendConservationOnDoorOpen | CH3 to CH4 when the door opens during conservation | If the robot is in NAVIGATING, AVOIDING_OBSTACLE or RETURNING, this is a violation of C9, and the robot must stop and alert. Transition T4 (to DELIVERING) is blocked while CH4 is active (C8, SR-04). |
| OP-07 EnterProtectionMode | CH3 to CH6 when correction fails within 120 s | If the chamber temperature is above the critical threshold it raises E8 EmergencyCondition, so the robot takes T12 or T14 (or T11, T13, T15, T16 depending on its state) into SAFE_MODE within 3 s (CR-01). |
| OP-13 VerifyConditionsBeforeResume | CH4 to CH3 only if checks pass | The robot may continue its journey only after the chamber is back in CH3. |

## 4. Operations whose postcondition must be verified by sensors
1. **OP-04 CorrectTemperature (and OP-05).** Issuing a cooling command does not prove the temperature came back. The actuator may be faulty, the door may be ajar, or the load may be too large. If the system assumed success, it would report a safe batch while vaccines warm up. Therefore OP-06 reads the sensors and requires the value to stay in range for the hold time.
2. **OP-14 SwitchToEmergencyPower.** A relay command only shows the system *tried* to switch. The emergency source could be empty or the relay stuck. If the system assumed power was restored, the chamber could be dead while the system believes it is protecting the batch. Therefore the bus voltage is read and must be within tolerance.

## 5. Supplementary robot and Mission Control operations (used in the RTM)
| OP_Id | Operation | Purpose |
|-------|-----------|---------|
| OP-C1 | AuthenticateOperator | Verify the operator identity and session token. |
| OP-C2 | AuthoriseRole | Check that the operator role may issue this command. |
| OP-C3 | ValidateCommand | Check format, parameter range and sequence ID. |
| OP-C4 | ExecuteCommand | Execute a valid, authorised command exactly once. |
| OP-C5 | ReportCommandStatus | Send RECEIVED, ACCEPTED/REJECTED, EXECUTING, COMPLETED/FAILED to Mission Control. |
| OP-C6 | LogEvent | Write command and critical events with timestamp, operator ID and robot ID. |
| OP-C7 | SendTelemetry | Report position, battery, chamber data and link status. |
| OP-C8 | EnterSafeMode | Stop the robot and protect the chamber within 3 s of an emergency condition. |
| OP-C9 | HandleLinkLoss | Hold position, buffer events, return autonomously after timeout. |
| OP-C10 | NavigateToDestination | Move along the planned route. |
| OP-C11 | AvoidObstacle | Handle a detected obstacle and resume navigation. |
| OP-C12 | NegotiateRailCrossing | Query ARLCCS and enter the crossing only when safe. |
| OP-C13 | PerformDelivery | Carry out the handover at the destination. |
| OP-C14 | ReturnToWarehouse | Travel back to the warehouse and become IDLE. |
