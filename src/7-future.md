Future plans {#future}
============

Real-world deployment
---------------------

As we prepare Deadlock for deployment at our faculty, more issues will certainly surface. We intend to make Deadlock a reliable long-term solution for our faculty. Later we are planning to expand to larger deployments.

Testing
-------

Due to time constraints, currently Deadlock does not have unit tests, although a simple integration test, plus the continuously running `ECHOTEST` sanity check included in `deadaux` (as described in section \ref{deadaux}), exist. Unit tests and more comprehensive integration tests would ease development. We are planning to reach 100% unit test coverage and setup continuous integration as soon as time allows.


System status monitoring
------------------------

If one wants a system to work, one needs to monitor it. In particular, metrics assessing the system health and performance should be exported; when a problem occurs, actions that can be taken automatically should be automatically taken; and actions that require human intervention should alert a human. A way to monitor the system and take appropriate actions (ideally based on an existing general solution) should be found.

Some basic watchdog functionality is present in Deadlock itself: controllers have a hardware watchdog that restarts them on lockup, and the integration test included in `deadaux` (see section \ref{deadaux}) can alert a human if things obviously don't work. However, we intend to explore more comprehensive solutions.


High-level rules and UI optimized for usage at universities
-----------------------------------------------------------

As part of deploying at our university, a domain-specific rules model will be developed, and the corresponding rules management interface will be created.


Tools and libraries
-------------------

Tools and libraries that further ease deployment and integration should be provided. In particular, a tool for importing data from often used systems, such as directory databases using the LDAP protocol or SQL databases, will be made available prior to the faculty-wide deployment estimated for autumn 2016.


A server implementation in Haskell
----------------------------------

The current server implementation (in Python) is production-ready. However, the lack of compile-time type checking is a considerable weakness. The type system in Haskell is very strong, and therefore it can find bugs which would normally not be found at compile time. We believe that a Haskell implementation would be far more trustworthy, and intend to write one.
