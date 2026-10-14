# SACDCS — Task 1: Requirement Extraction (Baseline v1.0)

## Assumed parameters (configurable; used to make requirements measurable)
| Parameter | Assumed value |
|-----------|---------------|
| Chamber limits (vaccine batch profile) | 2–8 °C |
| Battery critical-low level | 20% capacity or below |
| Battery emergency level | below 10% capacity |
| Critical temperature (battery / chamber) | defined in configuration file |
| Link-loss detection | no valid heartbeat for 30 s |
| Autonomous return after link loss | link not restored within 120 s |
| Chamber recovery period | 120 s |
| Vibration stabilisation period | 30 s |
| ARLCCS status freshness | not older than 5 s |

## A. Functional Requirements (FR)
| Req. ID | Requirement | Type | Priority | Verification Method |
|---------|-------------|------|----------|---------------------|
| FR-01 | The robot shall validate every received command (format, parameter range, sequence ID) and reject an invalid command with a reject-reason code without changing its state. A valid, authorised command shall be executed exactly once even if it is retransmitted. | FR | High | Test |
| FR-02 | When a valid delivery request is received while the robot is IDLE (battery above critical-low level and chamber ready), the robot shall start navigation to the requested destination. | FR | High | Test |
| FR-03 | While navigating, the robot shall check for obstacles continuously (at least every 100 ms); on detection it shall suspend navigation and start obstacle avoidance, and when the obstacle is cleared it shall resume navigation to the same destination. | FR | High | Test |
| FR-04 | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | FR | Critical | Test |
| FR-05 | The robot shall start the delivery process only after reaching the destination while NAVIGATING; after delivery is confirmed it shall travel back to the warehouse and become IDLE on arrival. | FR | High | Test |
| FR-06 | If battery capacity falls to the critical-low level (20%) during navigation, the robot shall abandon the delivery and return to the warehouse. If it happens during obstacle avoidance or delivery, the robot shall finish that step and then return. | FR | High | Test |
| FR-07 | The chamber shall correct out-of-range temperature or humidity and shall declare the value recovered only after sensor readings confirm it is back within the batch limits (not merely after a correction command is issued). | FR | Critical | Test |
| FR-08 | The robot shall report position, battery level, chamber temperature, humidity, door status and communication status to Mission Control at least every 5 s and immediately on every state change. | FR | High | Test |
| FR-09 | Mission Control shall receive an execution status for every command (RECEIVED, ACCEPTED or REJECTED, EXECUTING, COMPLETED or FAILED), each referencing the command ID. | FR | Medium | Test |
| FR-10 | The system shall record every command and every critical robot event in a persistent log with timestamp, operator ID, robot ID and command/event ID. | FR | High | Test, Inspection |

## B. Non-Functional Requirements (NFR)
| Req. ID | Requirement | Type | Priority | Verification Method |
|---------|-------------|------|----------|---------------------|
| NFR-01 (Performance) | Command processing shall normally complete within 5 s after the command is received by the robot (at least 95% of commands). | NFR | Medium | Test (timing) |
| NFR-02 (Security) | Only authenticated operators shall be permitted to issue robot commands (0 commands from unauthenticated sessions may be executed). | NFR | Critical | Test (negative), Review |
| NFR-03 (Reliability) | The system shall continue operating during temporary link interruptions of up to 30 s, with 100% of buffered events and telemetry delivered in order after the link is restored. | NFR | High | Test (fault injection) |
| NFR-04 (Scalability) | The system shall support communication with multiple robots simultaneously. | NFR | High | Load test |
| NFR-05 (Usability) | A trained operator shall be able to issue a standard command and see its status in at most 3 UI steps and within 60 s (at least 90% of test operators). | NFR | Medium | Usability test |

## C. High-Risk (Safety) Requirements
| Req. ID | Requirement | Type | Priority | Verification Method |
|---------|-------------|------|----------|---------------------|
| SR-01 (Railway crossing) | The robot shall not enter a railway level-crossing unless the ARLCCS reports barrier open AND no train approaching, with status not older than 5 s; otherwise it shall stop at the holding line. | Safety | Critical | Simulation, Test |
| SR-02 (Link-dependent) | If the Mission Control link is lost for more than 30 s, the robot shall enter Link Lost state: stop at the nearest safe point (never on a railway crossing), keep chamber conservation running and buffer events. If the link is not restored within 120 s it shall return to the warehouse autonomously. | Safety | High | Fault injection |
| SR-03 (Real-time / cold chain) | If a chamber condition is not corrected within the recovery period (120 s), the chamber shall enter protection mode, reduce light exposure and alert the operator within 5 s. | Safety | Critical | Test |
| SR-04 (Interlock) | The robot shall never enter DELIVERING directly from IDLE, from AVOIDING_OBSTACLE, or while the chamber door is open. | Safety | Critical | Analysis (model review), Test |

## Review notes
- AMB-01 (ambiguity): "critical battery" in FR-04 and the critical-low battery in FR-06 can be read as the same event but require different behaviour (Safe Mode vs return to warehouse). CR-01 resolves this by defining a separate emergency level (below 10%).
- NFR-02 and NFR-04 are recorded in their baseline wording, which is not measurable or complete. CR-02 and CR-03 fix this.
