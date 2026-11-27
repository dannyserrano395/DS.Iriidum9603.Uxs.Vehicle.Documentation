DS.Iridium9603.Uxs.Vehicle README
=================================

Overview
--------

DS.Iridium9603.Uxs.Vehicle is a companion-computer subsystem that provides
bidirectional MAVLink communication with DS.Iridium9603.Uxs.Control over 
an Iridium Short Burst Data (SBD) link.

DS.Iridium9603.Uxs.Vehicle enables ArduPilot to transmit HIGH_LATENCY2
messages and receive MAVLink commands from QGroundControl without requiring
an Internet connection.

-------------------------------------------------------------------------------

Features
--------

- Bidirectional MAVLink communication.
- HIGH_LATENCY2 telemetry transmission.
- Iridium SBD satellite communication.

-------------------------------------------------------------------------------

Communication Flow
------------------

    ArduPilot
        |
        v
    DS.Iridium9603.Uxs.Vehicle
        |
        v
    Iridium Short Burst Data (SBD)
        |
        v
    DS.Iridium9603.Uxs.Control
        |
        v
    QGroundControl

Communication is bidirectional. ArduPilot transmits HIGH_LATENCY2 messages
to QGroundControl, while QGroundControl sends MAVLink commands to ArduPilot.

-------------------------------------------------------------------------------

Documentation
-------------

Additional documentation is available in the following files:

- Architecture.md

-------------------------------------------------------------------------------

License
-------

Proprietary software.

All rights reserved.
