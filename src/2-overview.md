System overview {#overview}
===============

Existing systems: overview and comparison {#overview:existing}
-----------------------------------------

The solution currently employed at the university, as well as most commercially available alternatives, consist of a number of simple card readers and a centralized decision-making server and access management interface. They usually require custom wiring; introduce vendor lock-in because of proprietary communication protocols; and cannot operate when the server is unavailable. In contrast, Deadlock will rely on standard, hopefully already existing infrastructure; provide the communication protocol specification and libraries to extend our system; and make sure Deadlock continues to operate when the server cannot be reached.

Existing commercial solutions are also expensive (usually several hundred dollars per unit) and because of vendor lock-in parts cannot be replaced by alternatives. Deadlock aims to be almost an order of magnitude cheaper, and fully open.

<!-- \TODO{comparison with specific models} -->
<!-- not really -->

Key design principles {#overview:key-principles}
---------------------

### Modularity

Separating the functionality into independent modules with well-defined, simple, minimal interfaces simplifies development, makes the design much easier to grasp and minimizes the learning curve for a new developer. Things are better if one knows where to look for certain functionality (and where not to).

### Principle of Least Astonishment

"People are part of the system. The design should match the user's experience, expectations, and mental models." [@PrinciplesDesign]
If the design, implementation or behavior of a part of the system is obscure enough to surprise you, it should be redesigned.

We should prioritize predictability of the parts of the system which are hard to observe (such as the embedded devices), so that they do not unpleasantly surprise the user.

### State is ugly

State adds complexity to a system: if something has state, it has code managing it, and the developer must keep track of how the internal state influences the system. Also, if the system is stateful, it complicates failure recovery, as the state must be replicated or otherwise re-creatable. Therefore when at all possible, the components of Deadlock should have little internal state and depend only on explicit, well-defined, easily replicated data.


Main components
---------------

The system consists of a server and a number of controllers. Each controller serves a single point of access, holds a copy of the access rules and evaluates them locally. The server provides controllers with rules updates and collects access logs.

<!-- ![Deadlock components. Note: this picture is horrible, \TODO{}.](src/img/architecture.png) -->

### Server

The server holds the authoritative version of the access rules, collects logs and provides software updates and time synchronization for the other devices. It monitors system state (and reports it to the management UI).

Except for the contents of the database, it is entirely stateless. This simplifies the code and makes replication and failover trivial.

### Controller

The controller controls its associated point of access (e.g. unlocks its door). It takes actions (such as opening the door or logging) based on events observed (such as a card being presented or the door being opened). It periodically contacts the server, reporting its status and checking for updates.

The controller is "almost stateless" -- logs are sent to the server, and rules and firmware updates can be retrieved from the server. Therefore a device can be swapped simply by writing the correct device ID and encryption key to either the device or the database.

### Reader

Up to two card readers may be attached to one controller. A library to interface with our readers is provided, so they can be used independently of our controller.

### Hardware

The server is hardware-agnostic -- it runs on anything with networking and a Python environment. Deployments will usually use generic server hardware.

ŠVT has designed and built custom hardware for the controllers and readers. They focused on making it available and future-proof, extensible, and cheap. The schematics and other documents are available in the Deadlock source repository [@src].

In order to simplify installation, we have attempted to leverage existing infrastructure wherever possible: we use Ethernet for server/controller communication, adding optional Power over Ethernet, so we don't require any extra cables. Optionally, we can add a WiFi module to the controller for cases where electricity is available but connectivity is not. We even designed our reader boxes and connection cables to be easy to customize, so that they can be made compatible with existing holes in walls.[^customize]

[^customize]: For example, instead of using an expensive mold for the reader boxes, we make them from several layers of Plexiglas that can be cut individually. The source files for the cut pattern are available and easy to modify.


Access rules
------------

The decision whether to grant access is a function of user identity, point of access, date, time, and day of week. The access rules are designed to be simple without sacrificing generality. For details see chapter \ref{rules}.

Technical challenges
--------------------

### Reliability

As required by section \ref{requirements:reliability}, controllers must work during network failures. Continued operation is ensured by storing and evaluating the rules locally on the controller, and only needing the server for updating the local copy. Loss of access logs is avoided by saving them locally and never forgetting a log entry before it has been stored to disk by the server.

Multiple servers may be deployed for higher availability, as long as the backing database is synchronized by generic database replication mechanisms (eventual consistency is sufficient).

See chapter \ref{protocol} for details.

### Security

See section \ref{protocol:security}.

### Easy deployment and maintenance

Due to requirements in sections \ref{requirements:ease-maintenance} and \ref{requirements:availability}, communication is built on standard Ethernet, and we support the Power over Ethernet standard. Adding and replacing devices must be quick and easy, and therefore we made sure it was possible to pre-configure devices and then just plug in as needed.

Deadlock must be usable decades from now, therefore it must depend only on components, libraries and tools which are likely to stay. This requirement needed to be taken into account when designing the hardware and software.
