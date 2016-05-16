Requirements {#requirements}
============

Project Deadlock aims to create a complete system to allow cards compatible with the ISO/IEC 14443a standard (commonly known as *RFID cards*), such as International Student/Teacher Identification Cards, to be used to unlock doors and access other electronic equipment (hereafter *points of access* or *PoA*).

For this system to be useful at our university, Deadlock must meet the requirements outlined below.


Trustworthiness
---------------

As Deadlock may be used to protect valuable resources, such as computer rooms or labs, it must allow access when and only when it should.[^power] We must provide the user with reasons to trust this promise.

### Reliability {#requirements:reliability}

Points of access should be accessible even when things go wrong; specifically partial power or network outages must not prevent the system from allowing access, nor cause loss of access logs. Temporary server failure must cause no problems. Furthermore, the design and implementation should allow for a simple failover mechanism.

### Security {#requirements:security}

Deadlock must not allow illegitimate access. To protect privacy, logs or card IDs must not leak. We cannot assume a private communication channel. Therefore all communication in both directions must be authenticated and kept confidential.

[^power]: See section \ref{requirements:other:power-outage} for the discussion of behavior during power outages.


Practicality
------------

Deadlock must be an effective solution for our use case. This must hold even if the use case changes in the future.

### Extensibility {#requirements:extensibility}

In order to be prepared for the future, and to make incremental development possible, all software and all hardware must be modular, with well-defined interfaces, and extensible.

Functions not implemented in the first iteration, but expected to be added in the future, are

- arbitrary communication with the card,[^comm]
- controlling arbitrary appliances, not just door locks,
- WiFi module (for cases when power is available but Ethernet is not).

[^comm]: RFID cards are capable of complex actions, such as cryptographic verification of identity, or local data storage. Currently we only support reading the card's ID, but the communication stack is ready for extension.

### Ease of development {#requirements:ease-dev}

In the future Deadlock will likely be developed and maintained by students, not full-time developers. Therefore the codebase must be simple, easy to understand and change, the tools and libraries must be easy to use, and the overhead of introducing a new developer to the project must be minimal. When possible, general, well-known solutions should be used instead of solutions developed in-house.

### Ease of use {#requirements:ease-use}

Setting up access rules should be simple and convenient. This should not come at the expense of generality. Synchronization with the university's electronic information system is required, so that card information and groups like "CS teachers" or "PhD students" can be imported automatically.

The system should notify the operator if human intervention is required, but simple tasks and predictable issues should be handled automatically.

### Ease of deployment and maintenance {#requirements:ease-maintenance}

Deployment should be simple and with minimal overhead. On the hardware side, it should be possible to leverage existing infrastructure in order not to need extra cables for communication or power. On the software side, importing data from existing sources (such as our university's Academic Information System) should be possible. Replacing any failed components should be quick and should not require substantial training. The system should check its state and automatically fix whatever can be fixed automatically, e.g. reboot a device if it gets locked up.

### Availability {#requirements:availability}

Hardware should be cheap to manufacture and components should either be available in the foreseeable future or painlessly replaceable by newer alternatives.

In order to make Deadlock as available as possible, we release both the hardware schematics and the software to the public under the MIT license.


Further considerations {#requirements:other}
----------------------

### Power outage behavior {#requirements:other:power-outage}

In case of a power outage at entrance/exit PoAs, some doors should stay locked (to avoid the risk of breaching security), and some doors should open (e.g. emergency exits). Both can be supported by using different lock hardware and changing configuration.

### Emergency open

The hardware locks on entrance/exit PoAs must support manual opening and locking by authorized personnel. This is useful in emergencies.
