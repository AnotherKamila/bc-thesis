Server Design {#design}
=============

Like all other Deadlock components, the server design and implementation follows from the key design principles listed in section \ref{overview:key-principles}.


Data organization  {#design:data}
-----------------

Server replication and failover should be easy, and therefore any data/state that the server needs must be easy to replicate. Therefore the server is allowed to depend only on the contents of the relational database (for which replication mechanisms exist) -- it may cache or pre-compute some values, but otherwise all output must be a pure function of the database contents.

The following sections give an overview of the data the server works with.

### Point of access management and access logs

For each access point, we store its type, optional description, and which controller is attached to it.

Access logs (with data as specified in the message, according to section \ref{protocol:alog}) are saved in the database.[^ondisk] In order to fulfill the protocol idempotence guarantee, only logs with a unique combination of attributes are stored.

[^ondisk]: As the database journal is flushed to disk on commit, the logs are on long-term storage by the time we send an `OK` response to the controller.

Data about the state of the controllers, in particular the time of last PING, the rules database version, the firmware version and the local time (to measure drift) are stored (see section \ref{protocol:ping}). Other parts of the system, such as monitoring or a management UI, may use this information as they see fit (e.g. to alert if a controller has been silent for too long or is out of date).

### Access rule specification and identities management

Identities and  expressions form a DAG as described in section \ref{rulefmt:identityexpr}. We represent this by storing the edges (marked by the INCLUDE or EXCLUDE operation), and the nodes (containing a reference to either an identity, or a sub-expression) in the database.

In addition, we compute the "in expression" relation as described in section \ref{rules:inexpr}. See section \ref{impl:inexpr} for the description of the implementation.

The access rules are stored as a relation on the PoA types, identity expressions, time specifications, priority, and $\{\textit{Allow}, \textit{Deny}\}$. Time specifications may contain a list of weekdays, a date range, and a time of day, and these function as masks -- when unspecified, they match any weekday/date/time. Operations (such as union/intersection) on these masks are not supported -- this should instead be done by creating several rules with appropriate priorities.


Main components
---------------

The server functionality has been split into 3 separate components with minimalistic interfaces. These run as separate processes and it is not assumed that they run on the same machine.

### The "common files" interface  {#design:cfiles}

In order to avoid tight coupling between these modules, generally the only common interface among these (apart from the shared database) is the filesystem: when configuring the deployment, a shared filesystem directory with read and write access is given, and the components communicate by creating and accessing files in that location. This is usually sufficient, as the components are designed to run independently and only collect whatever happens to have been created by the other components. In particular, the purpose of several components is to create files meant for transfer to controllers via the `XFER` message (see section \ref{protocol:messages}), and for these we have created a simple common library for opening files meant for a specific controller (optionally with a fallback to files common to all controllers).

### `deadserver`: communicates with controllers  {#deadserver}

Listens for controller requests on a UDP socket, and sends responses according to the protocol specified in chapter \ref{protocol}. For `PING` and `XFER` requests (see section \ref{protocol:messages}), looks for files via the mechanism mentioned in section \ref{design:cfiles}.


### `deadapi`: the API for the outside world  {#deadapi}

Provides the HTTP API used by the web management and monitoring interface, and the provided commandline interface. Thereby bridges the outside world and the database via a simple CRUD REST API.

It supports pushing events via a streamed long-running HTTP response, in accordance with the Server-sent events/Eventsource specification [@sse]. Events are triggered via the LISTEN/NOTIFY pub/sub mechanism in Postgres [@pgnotify], and the database in turn contains triggers that send a NOTIFY on certain table row changes. Therefore data changes can bubble all the way to clients, which can use the standard Eventsource API to subscribe to these.

It provides a quick way to stage firmware updates: a firmware image together with a list of controller IDs can be uploaded, and `deadapi` simply drops (or links) the file into subfolders dedicated to the given controllers (see section \ref{design:cfiles}).


### `deadaux`: auxiliary jobs supporting the other components  {#deadaux}

`deadaux` is a collection of auxiliary modules that support the functionality of `deadserver` and `deadapi`, plus a very simple dispatcher that runs the modules in separate threads. By default, the following modules are part of `deadaux`:

- **`offlinedb`:** The main responsibility of this module is to build the copy of the rules database that the controllers use for local offline evaluation. Its main thread uses the pub/sub mechanism in Postgres, LISTEN/NOTIFY [@pgnotify], to be notified on rule changes. On change, it generates new versions of the files, and drops them where controllers can find them via the common files mechanism mentioned in section \ref{design:cfiles}.

- **`echotest`:** Uses the controller client library to periodically send `ECHOTEST` messages to `deadserver` and check the response. May be configured to e.g. send an email if any problem occurs.
