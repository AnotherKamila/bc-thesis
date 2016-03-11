Design overview {#overview}
==============================

Existing systems: overview and comparison {#overview:existing}
-----------------------------------------

The solution currently employed at the university, as well as most commercially available alternatives, consist of a number of simple card readers and a centralized decision-making server and access management interface. They usually require custom wiring; introduce vendor lock-in because of proprietary communication protocols; and cannot operate when the server is unavailable. While centralized access management is a requirement (and Deadlock will implement this schema), we will rely on standard, hopefully already existing infrastructure; provide the communication protocol specification and libraries to extend our system; and make sure Deadlock continues to operate when the server cannot be reached.

Existing commercial solutions are also expensive (usually $\sim\$10^2$ per unit) and because of vendor lock-in parts cannot be replaced by alternatives. Deadlock is aiming to be about an order of magnitude cheaper, and open.

Main components
---------------

The system consists of a server and a number of controllers. Each controller serves a single point of access, holds a copy of the access rules and evaluates them locally. The server provides controllers with rules updates and collects access logs. We provide a management and monitoring UI.


![Deadlock components. Note: this picture is horrible, \TODO{}.](src/img/architecture.png)

### Server

The server holds the authoritative version of the access rules, collects logs and provides software updates and time synchronization for the other devices. It monitors system state (and reports it to the management UI).

It is stateless -- requests are served based on just the rules and logs in the database. This simplifies the code and makes replication and failover trivial.

### Controller

The controller controls its associated access point (e.g. unlocks its door). It takes actions (opening the door, logging) based on events observed (a card being presented, the door opens). It periodically contacts the server, checking for updates.

The controller is "almost stateless" -- logs are sent to the server, and rules and firmware updates can be retrieved from the server. Therefore a device can be swapped simply by writing the correct device ID and encryption key to either the device or the database.

### Reader

Several card readers may be attached to the controller. We provide a library to interface with our readers, so they can be used independently of our controller.

### Hardware

The server is hardware-agnostic -- it runs on anything with networking and a Python environment. Deployments will usually use generic server hardware.

We have designed and built custom hardware for the controllers and readers. We focused on making it available and future-proof (using components which are available today, will probably remain available in the foreseeable future and can be replaced easily), extensible, and cheap. The schematics and other documents are available in the Deadlock source repository.

In order to simplify installation, we have attempted to leverage existing infrastructure wherever possible: we use Ethernet for server/controller communication, adding Power over Ethernet, so we don't require any extra cables. Optionally, we can add a WiFi module to the controller for cases where electricity is available but connectivity is not. We even designed our reader boxes and connection cables to be easy to customize, so that they can be made compatible with existing holes in walls.


Access rules
------------

The decision whether to grant access is a fuction of user identity, access point, date, time, and day of week.
Rules are of the form
$$(\textit{identity, access point, time specification}) \rightarrow \textit{allow}\,|\,\textit{deny}.$$
Default is "deny"; if multiple rules match, a "deny" rule overrides any "allow" rules.

As a simplification, identities, access points and time specifications can be grouped (even recursively, e.g. "*CS students* := *Bachelor CS students* and *Master CS students*; *staff* := *PhD students* and *faculty members*; *workdays* := Mon to Fri 8am to 6pm; allow *CS students* and *staff* to access *computer rooms* on *workdays*").


Technical Challenges
--------------------

### Reliability

Controllers must work during network failures, without losing access logs. The first one is provided by storing and evaluating the rules locally on the controller, and only needing the server to update the local copy. Continued correct operation is ensured by making the protocol stateless and idempotent, which allows the controller to retry operations until they succeed. It also makes implementing server failover trivial: as no state is stored on the server, we can simply employ IP's anycast mechanism (as described in [@Anycast]).

### Security

The system must securely operate over untrusted networks, resisting passive and active attacks.
Therefore communication is encrypted and authenticated using a device-specific key via the NaCl library [@NaClSecurity]. Nonces and idempotence prevent replay attacks.

### Easy deployment and maintenance

The system must not require separate communication infrastructure nor dedicated power supplies. Therefore we use a simple protocol implemented on standard UDP over Ethernet, and we support the Power over Ethernet standard.

Adding and replacing devices must not require substantial training. Therefore we had to minimize device configuration and make swapping devices with pre-configured ones trivial.

Deadlock must be usable decades from now, therefore it must depend only on components, libraries and tools which are likely to stay. This requirement needed to be taken into account when designing the hardware and software.
