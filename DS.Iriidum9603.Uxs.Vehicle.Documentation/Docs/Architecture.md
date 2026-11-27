Uxs.Vehicle Architecture
========================

Overview
--------

This document describes the architecture of the Uxs.Vehicle subsystem.

The Uxs.Vehicle subsystem provides bidirectional communication between
an autopilot and the Uxs.Control subsystem over an Iridium Short Burst
Data (SBD) datalink.

The subsystem manages MAVLink message processing, Iridium Short Burst
Data (SBD) datalink establishment, message retransmission, and telemetry
logging.

-------------------------------------------------------------------------------

High-Level Architecture
-----------------------

                    +-----------+
                    | Autopilot |
                    +-----+-----+
                          |
                          v
                    +------------+
                    | SerialPort |
                    +-----+------+
                          |
                          v
                    +------------+
                    |  Vehicle   |
                    +-----+------+
                          |
          +---------------+----------------+
          |               |                |
          v               v                v
    +-----------+   +-----------+   +---------------+
    | Autopilot |   | Satellite |   | Telemetry Log |
    |  Workers  |   |  Workers  |   |    Worker     |
    +-----------+   +-----+-----+   +-------+-------+
                          |                 |
                          v                 v
                   +-------------+   +---------------+
                   | Iridium9603 |   | Telemetry Log |
                   +------+------+   |     File      |
                          |          +---------------+
                          v
                   +-------------+
                   |   Iridium   |
                   |   Network   |
                   +------+------+
                          |
                          v
                   +-------------+
                   | Uxs.Control |
                   +-------------+

-------------------------------------------------------------------------------

Responsibilities
----------------

Vehicle

The Vehicle class implements the subsystem.

Responsibilities include:

- Processing inbound and outbound MAVLink messages.
- Establishing Iridium Short Burst Data (SBD) datalinks.
- Processing Iridium Short Burst Data (SBD) Ring Alerts.
- Retransmitting outbound MAVLink messages.
- Logging telemetry.

SerialPort

The SerialPort class provides access to the serial interface connected
to the autopilot.

Responsibilities include:

- Sending MAVLink messages to the autopilot.
- Receiving MAVLink messages from the autopilot.

Iridium9603

The Iridium9603 subsystem provides an interface for communicating with
an Iridium 9603 satellite transceiver.

Responsibilities include:

- Managing lifecycle state.
- Processing commands.
- Processing unsolicited result codes (URCs).
- Coordinating access to the transceiver.

-------------------------------------------------------------------------------

Worker Architecture
-------------------

The subsystem uses dedicated asynchronous workers to perform internal
operations.

The following workers are defined:

`TelemetryLogWorkerAsync`

Responsible for writing MAVLink messages to the telemetry log file.

`AutopilotInboundMessageWorkerAsync`

Responsible for sending MAVLink messages to the autopilot.

`AutopilotOutboundMessageWorkerAsync`

Responsible for receiving MAVLink messages from the autopilot and
queuing them for transmission.

`AutopilotRetryOutboundMessageWorkerAsync`

Responsible for queuing outbound MAVLink messages for retransmission
following a specified delay.

`SatelliteUnsolicitedResultCodeWorkerAsync`

Responsible for processing unsolicited result codes (URCs) received
from the Iridium9603 subsystem.

`SatelliteDatalinkWorkerAsync`

Responsible for establishing Iridium Short Burst Data (SBD) datalinks
and transferring inbound and outbound MAVLink messages.

-------------------------------------------------------------------------------

Autopilot Outbound Message Worker
---------------------------------

A dedicated worker continuously monitors the serial interface
connected to the autopilot for MAVLink messages.

The outbound message processing sequence is:

    Wait For Message
        ↓
    Read Message
        ↓
    Queue Outbound Message

Outbound MAVLink messages are placed into an internal channel for
processing by the satellite datalink worker.

Outbound MAVLink messages that fail to transmit to the Iridium
Gateway are placed into the retransmission channel for processing
by the autopilot retry outbound message worker.

-------------------------------------------------------------------------------

Autopilot Inbound Message Worker
--------------------------------

A dedicated worker continuously monitors the inbound message channel
and sends queued MAVLink messages to the autopilot through the
serial interface.

The inbound message processing sequence is:

    Wait For Message
        ↓
    Read Queued Message
        ↓
    Send Message To Autopilot

Inbound MAVLink messages are placed into an internal channel by the
satellite datalink worker for processing by the autopilot inbound
message worker.

-------------------------------------------------------------------------------

Satellite Datalink Worker
-------------------------

A dedicated worker continuously manages datalink establishment
and the transfer of inbound and outbound MAVLink messages.

An Iridium Short Burst Data (SBD) datalink is established when:

- An outbound MAVLink message is received.
- A Short Burst Data (SBD) Ring Alert is received.
- The configured datalink establishment interval expires.

The datalink establishment sequence is:

    Wait To Establish Datalink
        ↓
    Prepare Mobile Originated Message
        ↓
    Establish Datalink
        ↓
    Process Mobile Originated Transfer Status
        ↓
    Process Mobile Terminated Transfer Status
        ↓
    Clear Mobile Originated And Mobile Terminated Buffers

Outbound MAVLink messages are prepended with the destination RockBLOCK
serial number using the RockBLOCK direct messaging format.

If a Short Burst Data (SBD) Ring Alert is received, the AT+SBDIXA command
is used to establish the datalink; otherwise, the AT+SBDIX command is used.

-------------------------------------------------------------------------------

Satellite Unsolicited Result Code Worker
----------------------------------------

A dedicated worker continuously monitors the Iridium9603 subsystem
for unsolicited result codes (URCs).

The unsolicited result code processing sequence is:

    Wait For URC
        ↓
    Identify SBD Ring Alert
        ↓
    Queue SBD Ring Alert

Short Burst Data (SBD) Ring Alerts are placed into an internal
channel for processing by the satellite datalink worker.

An interval of ten seconds is enforced between queued Short
Burst Data (SBD) Ring Alerts.

-------------------------------------------------------------------------------

Autopilot Retry Outbound Message Worker
---------------------------------------

A dedicated worker continuously monitors the retransmission channel
and queues outbound MAVLink messages for retransmission following
a specified delay.

The subsystem permits up to two retransmission attempts for each
outbound MAVLink message.

The retransmission sequence is:

    Mobile Originated Transmission Failed
        ↓
    Queue Message For Retransmission
        ↓
    Wait For Retransmission Delay
        ↓
    Decrement Remaining Retransmission Attempts
        ↓
    Queue Outbound Message

The first retransmission attempt is delayed by 30 seconds.

The second retransmission attempt is delayed by 60 seconds.

Outbound MAVLink messages that exhaust their retransmission attempts
are placed into the telemetry log channel to be logged by the
telemetry log worker.

Outbound MAVLink messages that fail to transmit from the transceiver
to the Iridium Gateway are placed into the retransmission channel by
the satellite datalink worker.

-------------------------------------------------------------------------------

Telemetry Log Worker
--------------------

A dedicated worker continuously monitors the telemetry log channel
and writes queued MAVLink messages to the telemetry log file when
telemetry logging is enabled.

Telemetry logging is enabled when a telemetry log directory is
specified. A new telemetry log (.tlog) file is created in the
specified directory each time the subsystem is started.

The filename represents the number of microseconds elapsed since
the Unix epoch at the time of file creation.

Each telemetry log entry consists of:

- An eight-byte timestamp.
- A MAVLink message.

The timestamp represents the number of microseconds elapsed since
the Unix epoch and is encoded in big-endian byte order.

The telemetry log file format is compatible with the telemetry log
file format used by QGroundControl.

Telemetry log files generated by the Uxs.Vehicle subsystem can be
merged chronologically with telemetry log files generated by
QGroundControl to provide a more complete record of MAVLink
communications during a mission.

-------------------------------------------------------------------------------

Lifecycle Management
--------------------

The subsystem exposes two lifecycle operations:

    RunAsync(...)
    StopAsync(...)

RunAsync establishes connections to the autopilot and the
Iridium9603 subsystem, starts the workers, and waits for the
workers to complete.

StopAsync initiates cancellation of the workers and waits for
the workers to complete.

A shared CancellationTokenSource is used to coordinate cancellation
of workers.

When a communication worker terminates, cancellation of the remaining
workers is initiated through the shared CancellationTokenSource.

`Exception Propagation`

Unhandled exceptions that occur within workers are propagated to
the consumer after all workers have completed.

If a single unhandled exception occurs, the original exception
is propagated. If multiple unhandled exceptions occur, an
AggregateException containing the exceptions is propagated.

`Telemetry Fault Isolation`

The telemetry log worker operates independently of the communication
workers.

Failures associated with telemetry logging do not initiate
cancellation of the communication workers.

This allows the subsystem to continue providing bidirectional
communication between the autopilot and the Uxs.Control subsystem
in the event of a telemetry logging failure.

-------------------------------------------------------------------------------

Disposal
--------

The subsystem supports deterministic disposal of internal resources.

Consumers should call StopAsync and wait for it to complete before
disposing the subsystem.

The subsystem disposes the FileStream associated with the telemetry
log, the SerialPort connected to the autopilot, and the Iridium9603
subsystem.

The CancellationTokenSource is disposed only if execution has been
initiated and StopAsync has completed the shutdown of the subsystem.

Otherwise, the subsystem does not dispose the CancellationTokenSource.
This prevents the CancellationTokenSource from being disposed while
it may still be in use. In this case, the CancellationTokenSource is
reclaimed by the garbage collector when it is no longer referenced.