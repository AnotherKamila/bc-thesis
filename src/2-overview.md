Design overview {#overview}
==============================

\TODO{perhaps more details.}  
\TODO{maybe change structure: specification => fun design choices => important decisions => overview. Maybe not and use this structure only for "my" stuff (chapters \ref{server} and \ref{protocol}).}

------------------------------------------------------------------------------

The system consists of a server and a number of controllers. Each controller serves a single point of access, holding a copy of the access rules and evaluating them locally. The server provides controllers with rules updates and collects access logs. We provide a management+monitoring UI.

Main components
---------------

![Deadlock components. Note: this picture is horrible, \TODO{}.](src/img/architecture.png)

### Server

The server holds the authoritative version of the access rules, collects logs and provides software updates and time synchronization for the other devices. It monitors system state (and reports it to the management UI).

It is stateless -- requests are served based on just the rules and logs in the database. This simplifies the code and makes replication and failover trivial.

It is hardware-agnostic -- it runs on anything with networking and a Python environment.

### Controller

The controller controls its associated access point (e.g. unlocks its door). It takes actions (opening the door, logging) based on events observed (a card being presented, the door opens). It periodically pings the server, checking for updates.

The controller is "almost stateless" -- logs are sent to the server, and rules and firmware updates can be retrieved from the server. Therefore a device can be swapped simply by writing the correct device ID and encryption key to either the device or the database.

### Reader

Several card readers may be attached to the controller. We provide a library to interface with our readers, so they can be used independently of our controller.

Access rules
------------

The decision whether to grant access is a fuction of user identity, access point, date, time, and day of week.
Rules are of the form
$$(\textit{identity, access point, time specification}) \rightarrow \textit{allow}\,|\,\textit{deny}.$$
Default is "deny"; if multiple rules match, a "deny" rule overrides any "allow" rules.

As a simplification, identities, access points and time specifications can be grouped (even recursively).

Technical Challenges
--------------------

### Reliability

Controllers must work during network failures (without losing access logs). Solved by storing and evaluating the rules on the controller and making the protocol stateless and idempotent, allowing the controller to retry operations until they succeed and making server failover trivial.

### Security

The system must securely operate over untrusted networks, resisting passive and active attacks.
Therefore communication is encrypted and authenticated using a device-specific key via the NaCl library [@NaClSecurity]. Nonces and idempotence prevent replay attacks.

### Easy deployment and maintenance

The system must not require separate communication infrastructure nor dedicated power supplies. We use ethernet and support the Power over Ethernet standard.

Adding and replacing devices must not require substantial training. Solved by making device configuration minimal and making swapping devices with pre-configured ones trivial.

Deadlock must be usable decades from now, therefore it must depend only on components, libraries and tools which are likely to stay. This requirement needed to be taken into account when designing the hardware and software.
