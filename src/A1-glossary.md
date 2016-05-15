\newpage

# Glossary

access rule
:   A function from $(identity, PoA, time)$ to $Allow$ or $Deny$ that defines whether to grant access to the given PoA. See chapter \ref{rules}.

controller
:   Controls its associated point of access (e.g. unlocks its door) based on the access rules. Communicates with the server.

`deadapi`
:   The HTTP API for Deadlock management, access rules configuration and status monitoring.

`deadaux`
:   Auxiliary jobs supporting tasks such as offline database creation.

`deadserver`
:   The Deadlock server that communicates with controllers.

PoA, point of access
:   A door lock, a printer or any other device, access to which is controlled by a Deadlock controller.

reader
:   The user-visible box at PoAs that reads RFID cards and provides visual and auditory feedback about whether access is granted. Communicates with its controller.

ruleset
:   A tagged set of access rules. Every access rule in the system belongs to exactly one ruleset. Creating, updating or deleting a ruleset is an atomic operation.

server
:   The centralized data store and "manager" of the system. Communicates with controllers (`deadserver`, see esp.\ chapter \ref{protocol}), provides an API for the outside world (`deadapi`), and performs auxiliary tasks (`deadaux`).
