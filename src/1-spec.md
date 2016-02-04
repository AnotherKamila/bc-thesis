Specification {#spec}
=============

Project Deadlock aims to create a complete system to allow ISO/IEC 14443a-compatible cards (commonly known as *RFID cards*), such as International Student/Teacher Identification Cards, to be used to unlock doors and access other electronic appliances (hereafter *points of access*).

For this system to be useful at our university, Deadlock must meet the requirements outlined below.


Reliability {#spec:reliability}
-----------

Points of access should be accessible even when things go wrong, specifically partial power or network outages must not make controllers stop allowing access nor lose access logs. Server failure must also cause no problems.

Furthermore, allowing for a simple implementation of server failover would be a good idea.


Security {#spec:security}
--------

As Deadlock may be used to protect valuable resources, such as computer rooms or labs, it must allow access if and only if it should.[^power] Logs or card IDs may be private, so they must not leak. Deadlock will be employed in publicly accessible places, meaning we cannot assume a private communication channel. Therefore all communication in both directions must be secret and authenticated.

[^power]: See \ref{spec:other:power-outage} for the discussion of power outages.


Extensibility {#spec:extensibility}
-------------

In order to be prepared for the future, and also to make incremental development possible, all software and all hardware must be modular, with well defined interfaces, and extensible.

Functions not implemented in the first iteration, but expected to be added in the future, are

- arbitrary communication with the card,
- controlling arbitrary appliances, not just door locks,
- WiFi module (for cases when power is available but Ethernet is not).


Ease of development {#spec:ease-dev}
-------------------

In the future Deadlock will likely be developed and maintaned by students, not fulltime developers. Therefore the codebase must be simple, easy to understand and change, the tools and libraries must be easy to use, and the overhead of introducing a new developer to the project must be minimal.


Ease of use {#spec:ease-use}
-----------

Setting up access rules should be simple and convenient. Synchronization with the university's electronic information system is required, so that card info and groups like "CS teachers" or "PhD students" can be imported automatically.

It should bother a human \iff human intervention is required -- simple tasks and predictable issues should be handled automatically.


Ease of deployment and maintenance {#ease-maintenance}
----------------------------------

Replacing any failed components should be quick and should not require substantial training.

Deployment should be simple and with minimal overhead. On the hardware side, it should be possible to leverage existing infrastructure in order to not need extra cables for communication or power. On the software side, importing data from existing sources (such as our university's Academic Information System) should be possible.

The system should check its state and automatically fix whatever can be fixed automatically, e.g. reboot a device if it gets locked up.


Availability {#spec:availability}
------------

Hardware parts should be cheap to manufacture and components for them should either be available in the future or painlessly replaceable by their newer alternatives.

In order to make Deadlock as available as possible, we will release both the hardware schematics and the software to the public under the MIT license.


Further considerations {#spec:other}
----------------------

### Power outage behavior {#spec:other:power-outage}

In case of a power outage, some doors should stay locked (to avoid the risk of breaching security), and some doors should open (e.g. emergency exits). While both can be supported, our use case requires only the "default close" behavior and therefore the current controller model is hard-wired for this case.

### Emergency open

The system must implement a "force open" command which will unlock the door. This is useful in emergencies (as long as power is available).
