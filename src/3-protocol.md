Server/controller communication protocol {#protocol}
========================================

Design requirements
-------------------

The server/controller communication protocol must facilitate reliability, security, extensibility and ease of development and deployment, as defined in sections \ref{requirements:reliability}, \ref{requirements:security}, \ref{requirements:extensibility}, \ref{requirements:ease-dev} and \ref{requirements:ease-maintenance}. This led to the following decisions:

### Statelessness and idempotence

In order to keep the protocol simple, yet reliable, communication should be stateless and all messages should be idempotent. This allows for retrying anything that failed for any reason, at any time.

### Simplicity

The protocol should be simple. This includes not only low complexity of the protocol's states and messages, but also simplicity in message parsing on any device and in any programming language. Specifically, parsing must be efficient on embedded devices with low CPU frequency and memory.

### Extensibility

The protocol must be easily extensible. Additionally, a mechanism for seamlessly transitioning to a newer version in a live system should be provided. These must not interfere with the simplicity requirement.

### Security

We do not want to rely on security mechanisms provided by lower layers (as they may or may not be adequate, or present). Therefore the application layer of the protocol must provide sufficient authentication and secrecy.

### Built on standard, well-known technologies

Code is a liability. Our code has our bugs, requires specific knowledge, and is our problem. Therefore the protocol should reuse existing code and technologies wherever appropriate.

--------------------------------------------

Note that fulfilling these requirements is not at all trivial, as some, especially extensibility vs. simplicity, or security vs. simplicity, may end up contradicting each other.

Statelessness and idempotence are not strictly required, as reliability may be achieved in different ways, too. However, as shown in the following sections, statelessness is a very useful approach, as it allows to keep things simple even in the face of requirements that would otherwise need significant complexity.

Protocol design
---------------

### Alternatives considered

The original suggestion was a connection-oriented protocol, where either the server or the controller could initiate communication. This would abstract away details like maximum packet size, handle lost packet retransmissions, and allow for pushing rule or firmware updates from the server side. However, this approach presents multiple problems, such as

 - high server failover cost: in a stateful protocol, connection state would either have to be synchronized among multiple servers, which is impractical, or all communication would have to be restarted from the beginning on failure;
 - added reliability only at the cost of added complexity: again because of problems with server state;
 - more complexity on the server: the server would need to keep track of active controllers, and who has said and heard what;
 - extra CPU cycles, flash and memory usage on embedded devices: while an efficient implementation of these abstractions might be unnecessary or could be written, nonexistent code is easier to write, more efficient, and has fewer bugs.

### Chosen design: overview, rationale  {#protocol:overview}

The chosen communication protocol is connectionless and stateless, and all information exchange is of the form "controller request $\rightarrow$ server response". All requests (including retries) can be served independently of any other past, present or future requests from this or another controller.

This implies that there is in fact no strict requirement to serve all requests, or any particular subset of requests, by a single server. Multiple server instances can be used as long as the data they need can be synchronized (and even for this synchronization, eventual consistency is sufficient). Therefore, the system may be configured to use several servers, and controllers are expected to send requests, including retries, in a more or less round-robin fashion.[^moreorless]

[^moreorless]: "More or less" means that the controller is allowed to cache "server dead" information in order to skip dysfunctional servers.

The round-robin scheduling is particularly useful for retries: if there is a problem with a specific server or a specific part of the network, the controller will simply re-send the request to a different server until a good response is received. Therefore the probability that no server can be reached can be significantly reduced by deploying multiple servers in different parts of the network.

As will be shown in section \ref{protocol:messages}, under normal circumstances the server/controller communication is not latency-sensitive, so the round-robin retries approach does not pose a latency problem. Therefore the controllers use round-robin with generous timeouts and exponential back-off to avoid network congestion.

A "bad" response (such as one that cannot be parsed, or an error) is treated the same as if no response were received (except for possibly different timeouts, logging and such), i.e. a retry is sent to the next server. This allows for uniformly handling all kinds of transient and permanent problems with the server, network or other resources.

### Live system upgrades {#protocol:live-upgrades}

A welcome consequence of the message independence and round-robin retries for all errors is that even if a server cannot parse a controller's request, or a controller cannot parse a server's response, the controller will simply retry with a different server. Therefore in order to transition to an incompatible protocol version, all that is needed is deploying servers with both the "old" and the "new" protocol, and the controllers will simply retry until they find a compatible server. Together with the fact that the server can automatically deliver firmware updates, and that controllers report their firmware version to the server, this makes any and all online system upgrades trivial and fully automatic.


Network stack
-------------

The standard network stack is used: Ethernet (IEEE 802.3) as the physical and data link layers, IP as the network layer and UDP as the transport layer. For IP, both IPv4 and IPv6 are supported, and standard ARP or NDP, respectively, is supported for network to link address resolution. IP addresses may be configured statically or obtained via DHCP.

### UDP vs. TCP

A standard TCP implementation is available for all devices we will use (it is even bundled with the real-time OS used for the embedded devices). Therefore it seems like using TCP would provide benefits at no additional cost. However, as our protocol is stateless and packet-oriented, and manages retransmissions on the application layer, the only benefit of TCP would in fact be unlimited "packet" length (as opposed to 64kB for UDP [@UDP]), and other than that we would end up emulating a UDP-like service on top of TCP if we chose to use it.

While the unlimited message size looks useful, it is in fact not that helpful -- the only messages that do not fit into a single UDP packet are rule database and firmware blobs, and for these it is more efficient to deliver them in explicit chunks, so that the transfer of these large files does not need to start over in case something goes wrong.

Therefore, as the benefits of TCP are in our case not worth the TCP overhead and flash space on embedded devices is limited[^flash], we have chosen to use UDP.

[^flash]: The current controller model will have slightly less than 256kB of programmable flash memory, of which less than 128kB is usable, because we need to store two versions of the firmware when doing online firmware upgrades. Therefore the several kB saved by not compiling in the TCP stack might come in handy.


Message types, controller behavior {#protocol:messages}
----------------------------------

Controllers are expected to download a local copy of the rules database and query that instead of contacting the server whenever access is requested. They send access logs, report their status, and request updates of the rules database and firmware.

As all communication must be initiated by the controller, it must periodically contact the server in order to find out if an updated rules database or firmware is available. 

All responses have a response status tag, the value of which is one of `OK`, `ERROR` (permanent error), `TRY_AGAIN` (transient error). Any non-`OK` response must be treated as if the response did not arrive (i.e. usually a retry as in section \ref{protocol:overview} is necessary), except for possibly different timeouts, logging or scheduling. In the following, only `OK` responses are shown.

**Note:** The following describes the "semantic" data types. The details of the encoding are specified in section \ref{protocol:cbor}. The type "byte string" represents a binary-safe string, or an array of bytes of arbitrary length.

Currently, the following message types are recognized:


### `PING`: keepalive, DB and FW version info  {#protocol:ping}

Contacts the server to report current status and request info about updates. Also used to adjust controller time.

\clearpage

**`PING` request:**

Field       Type     Description
----------  -------  -------------------------------------------------
time        integer  what time the controller thinks it is
db_version  integer  version of the rules database currently in use
fw_version  integer  currently running firmware version

**`PING OK` response:**

Field       Type     Description
----------  -------  --------------------------------------------------
time        integer  server time
db_version  integer  newest available version of the rules database
fw_version  integer  newest available firmware version

The controller is expected to adjust its clock to match the server time.

### `ALOG`: transfer access logs   {#protocol:alog}

Sends access logs to the server.

Controllers attempt to send access logs as soon as possible, but in order not to lose them, they are saved to the SD card until the server confirms they have been written to disk.[^capacity] Logs may be sent in multiple batches if needed.

[^capacity]: We recommend at least 4GB SD cards in order to have enough space for flash wear leveling. As each log record is about 20-30 bytes (depending on the encoding), at a rate of 1 access per second (which is somewhat overstated) it would take about 5 years to run out of space.

**`ALOG` request:**

Field       Type                  
----------  ------------------------------
records     array[^array] of `log_record`s (defined in table \ref{table:log_record})

[^array]: The array may end with a termination symbol instead of having an explicitly specified length. See section \ref{protocol:cbor} for details of the encoding.

Field    Type         Description
-------  -----------  ----------------------------------------------
time     integer      timestamp
card_id  byte string  card that requested access
allowed  boolean      was access granted?

Table: `log_record` structure \label{table:log_record}

**`ALOG OK` response:** All sent records have been written to disk. (Response body empty.)


### `XFER`: transfer a file chunk  {#protocol:xfer}

Firmware and rule database updates are treated as opaque binary blobs by the `XFER` command. They are identified by type and version. In order to trivially support incremental downloading and arbitrary chunk sizes, the controller explicitly requests the offset and length of the chunk. The same version must always refer to an exactly identical blob (if it exists), even if requested from a completely independent server.[^identical] The server may return a smaller chunk, but never longer. A chunk of length 0 indicates end of file.

[^identical]: See section \ref{impl:fun:fileversion} for notes on how we implemented this. From the protocol's viewpoint, versions must be treated as opaque integers with this and only this guarantee.

**`XFER` request:**

Field        Type     Description
-----------  -------  -------------------------------------------------
filetype     enum     `DB` and `FW` currently supported
fileversion  integer  same version $\implies$ same contents
offset       integer  offset from the beginning of the blob
length       integer

\clearpage

**`XFER OK` response:**

Field       Type         Description
----------  -----------  --------------------------------------------------
length      integer      may be less than requested
chunk       byte string  the file chunk contents

The server will return a `TRY_AGAIN` error if the file was not found. This would usually happen because one server already received and processed an update and another one is behind. The controller will simply retry until it finds a ready server.

Note: These are the only responses longer than a few bytes. The server will send whatever size it is asked for (up to the generous packet size limit). It is each controller's responsibility not to ask for chunks that may result in replies that are too long for it to process. This is to allow maximum efficiency with controllers with different capabilities.


### `CRITICAL`: report a critical problem  {#protocol:critical}

Used to report a critical problem, upon which the server should take immediate action.

**`CRITICAL` request:**

Field    Type                  Description
-------  --------------------  ----------------------------
code     enum                  error code
message  optional text string  details of the error, if any

Currently the only recognized codes are `LOCK_FORCED_OPEN` (a physical lock was opened without permission) and `READER_NOT_RESPONDING` (a reader is not responding correctly even after multiple restarts), but we assume that more uses will emerge when preparing for real-world deployments.

**`CRITICAL OK` response:** Acknowledged, action taken. (Response body empty.)


### `ASK`: ask if access should be granted now

Because of the potentially high latency of roundtrips, local evaluation rather than querying the server should be used in production. However, we include this for special cases and as a fallback.

**`ASK` request:**

Field       Type         Description
----------  -----------  -------------------------------------------------
card_id     byte string  card that requested access

Whether access should be granted is a function of identity, time and PoA (for details see chapter \ref{rules}). In this case, this card's identity, the current (server) time and the PoA associated with this controller are used.

**`ASK OK` response:**

Field       Type         Description
----------  -----------  -------------------------------------------------
allowed     boolean      do we allow access?

### `ECHOTEST`: echo for testing purposes

Echoes the request body. This is helpful in integration testing. Live deployments are recommended to run a process that will act as a controller sending `ECHOTEST` (and possibly other) requests and report any problems. (Such a process is run by default -- see section \ref{deadaux}.)


Packet format
-------------

### Record encoding {#protocol:cbor}

All requests and responses, as well as the outer packet envelope, are "records", i.e. small key-value mappings with fixed key names and types. Therefore we originally wanted to simply transmit "C structs" (i.e. binary blobs with fixed offsets for fields) and hard-code field offsets in the server and controller firmware. However, this approach has multiple disadvantages:

- Any extension would be an incompatible change, and therefore would require the full upgrade procedure as described in section \ref{protocol:live-upgrades}. While this procedure is simple, when it is running, the system requires more servers to achieve the same level of redundancy; and it may make administrators nervous.
- We may parse a packet incorrectly without noticing, if the length matches.
- When the length does not match, we don't know anything more specific than "parsing failed".
- The blob is not self-describing, and therefore nothing is known about it without the context of the outer envelope specifying the version and the description of fields for this version.

Especially the concerns around parsing errors are significant enough to justify a self-describing encoding. Therefore we need an encoding with the following properties:

- self-describing: key names and types must be present in the the encoded data
- expressive: it must be possible to include all the necessary types and arbitrarily nest them as arrays or sub-records; optional fields must be supported
- binary-safe: able to transmit arbitrary binary data (e.g. card IDs or file chunks) without the need for extra encoding
- not incompatible by default: when a backwards-compatible change is introduced (such as adding a new optional field, or removing a field that was optional), old and new code must be able to communicate without change
- suitable for embedded devices: encoding and decoding must be fast, using small code size and producing small messages
- standard, with existing libraries: our code is our problem -- the less code we write, the less code we will need to maintain in the future

These requirements are perfectly fulfilled by the Concise Binary Object Representation (CBOR, see [@CBOR]) -- a data format designed for communicating with constrained nodes. We use arrays of CBOR semantically tagged items to represent records.[^duplicate] (These are equivalent to arrays of $(\textit{tag}, \textit{data})$ pairs, where $\textit{data}$ is strongly typed.) Unknown tags are ignored and from the parsing viewpoint all fields are optional. In this way the only thing that a server and a controller must have in common to communicate are the tag interpretations (which makes sense, if they want to use the values for something useful).

[^duplicate]: If a duplicate tag is encountered, it is considered an error. In addition to serving as a sanity check, this might prevent some overflow-related attacks.


### Requests, responses {#protocol:requests-responses}

For all requests and responses, the record as specified in \ref{protocol:messages} is tagged by a semantic tag for the corresponding message type, and in case of response records this is in turn tagged by the response status.


### "Envelope" -- version, addressing, encryption

The outer layer of the messages (common to requests and responses) provides addressing and encryption. It is a record with fields as specified in table \ref{table:protocol:envelope}. The encoded record is prepended with a 4-byte "magic number" containing the bytes $[68, 69, 65, 68]$ ('DEAD' in ASCII) identifying this as a Deadlock message.

Field                      Type  Description
------------------  -----------  -------------------------------
Version identifier      integer  unknown version must be ignored
Controller ID           integer  addressing
Nonce                  24 bytes  random bytes
Payload             byte string  encrypted request/response

Table: Message record. \label{table:protocol:envelope}

Version identifier
:   Packet must be considered invalid if this does not match a known version. This is to support live system upgrades, as detailed in section \ref{protocol:live-upgrades}.

Controller ID
:   Unique identifier of the sender or intended recipient. Serves as addressing. Including a form of addressing on the application layer decouples "logical" addressing from "physical" (i.e. network) addressing, thereby allowing Deadlock to function over NAT, with broadcast/multicast/anycast IP addresses, and such.

Nonce
:   Randomly generated bytes. Matches a response to a request: when a request nonce is $x$, the associated response's nonce must be $x \oplus 1$. Used as detailed in section \ref{protocol:security}.

Payload
:   Request/response, encoded according to section \ref{protocol:cbor}. Encrypted with the key for the given controller using the nonce, as detailed in section \ref{protocol:security}.

**Note:** Maximum message size (when encoded and encrypted) is 63kB (in order to comfortably fit into a UDP packet).


Security {#protocol:security}
--------

Security is complicated. While libraries implementing cryptographic primitives exist, they usually do not make securing an application particularly easy: the developer must be aware of what needs what kind of security; which primitives (such as cipher, cipher mode, checksums, signatures) are suitable for which use case, what they promise, what their weaknesses are and whether they are a problem in the given use case; she must consider potential side channel attacks, replay attacks, and such; and she must ensure other developers are aware of all these considerations. As the numerous vulnerability reports published each month signify, this is no easy task. 

Short of locking one's computer in a closet without electricity, the best way to secure a system is to leave it to an expert. Luckily, in 2013 the NaCl library interface specification [@NaClDoc] and several implementations were published, with the aim of providing developers with a simple, "sane defaults" crypto toolkit. See [@NaClSecurity] for a discussion of the impact of such a library.

In Deadlock, we assume operation over untrusted networks, and we must resist both passive and active attacks. Therefore we encrypt and authenticate all messages from/to a given controller with a device-specific symmetric key, using NaCl's `secret_box(nonce, key, payload)` function, which promises secrecy and integrity provided the nonce is not used more than once [@NaClDoc]. We construct the nonce by generating 24 random bytes,[^random] which ensures negligible collision probability (quick birthday paradox approximation says the probability reaches 50% after more than $10^{28}$ packets, which is a lot). Symmetric cryptography was chosen for performance, but once the actual controller hardware and firmware exists, we are planning to run benchmarks and switch to asymmetric cryptography if possible, in order to avoid the need to copy the secret to more than one place.

The default NaCl primitives in NaCl are the Salsa20 stream cipher for symmetric encryption and the Poly1305 MAC for message authentication. As detailed in [@NaClCrypto], these are secure and performant without depending on any form of hardware acceleration, which goes well with our requirements. 

[^random]: "Random" in this case does not mean cryptographically secure randomness -- nonces may be predictable (they are sent in cleartext along with the payload anyway), the only requirement is a uniform distribution to ensure low collision probability. The fact that NaCl does not require a source of good randomness is in embedded environments very welcome.

### Security guarantees

Provided a nonce is not used more than once, `secret_box(nonce, key, payload)` guarantees

 - **secrecy**: it is infeasible to decrypt a message without knowledge of the key;
 - **integrity**: if a message is decrypted successfully, no accidental or purposeful third party modification of the nonce or the encrypted payload can have occurred;
 - **resistance to timing attacks**: the implementations try to always perform the same amount of work.

Furthermore, the protocol's idempotence and use of nonces **prevents replay attacks**: if an attacker attempts to replay a request to a server, nothing bad will happen as all requests are idempotent; if she replays a response to a controller, its nonce will not match any of the responses the controller is currently expecting and therefore it will ignore the fake response.
