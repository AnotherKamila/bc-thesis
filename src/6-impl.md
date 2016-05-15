Server Implementation {#impl}
=====================

Chosen programming language
---------------------------

Several alternatives were considered for the server code, notably C++ and Haskell, but in the end we chose Python 3, for the following reasons:

- **Low entry cost for new developers**: Most potential future developers are already familiar with Python: the Applied Informatics branch at our faculty teaches it in the first year, and the Informatics students are expected to know Python in various courses in the curriculum. Also, Python is quite simple, and therefore becoming proficient at it is a lot easier than with most other languages.
- **Great simplicity/awesomeness ratio**.
- **Principle of Least Astonishment**: Unlike e.g. C++, Python is simple and consistent.
- **Effective constructs that encourage good design and correct code**: Language constructs such as decorators encourage modularity and composition, and e.g. context managers help ensure resources, transactions and such are managed correctly.
- **Good libraries available**: Libraries for common tasks such as interfacing with the DB, serving UDP or HTTP requests, and much more, are readily available, well known and well tested.
- **Fast prototyping**: For the above reasons, getting something up and running is quick with Python.

The obvious, and considerable, disadvantage of Python, is the lack of static typing -- without static typing, many errors which could be discovered by a compiler will only appear at runtime. In fact, the type system is the main reason for the author's ongoing desire to switch to Haskell. However, not many people know Haskell, and we want it to be easy to contribute to Deadlock, so it is a much better idea to pick a well known language.

The Python 3 language and standard libraries are documented in [@PythonDoc].


Targeted environment, dependencies
----------------------------------

- The Deadlock server is meant to be run on **a Unix-like server**. While it may work on multiple platforms, it is currently tested only on Debian-like Linux distributions and FreeBSD 10.
- We are targeting **Python 3.4 or newer**. This is because at the time of writing Python 3.4 is available in all relevant OSs and distributions (in particular Debian Stable), and contains useful features not present in previous versions.
- For the DBMS, **PostgreSQL >= 9.3** is required. We use non-standard Postgres-specific features, such as the PL/pgSQL in-database procedural language [@plpgsql] for rules pre-computation, or the NOTIFY/LISTEN pub/sub system for notifying `deadaux` of access rules changes.
- Several of the used libraries (at least `psycopg` and `pynacl`) use native bindings, and therefore only work with the CPython implementation of Python 3.


Database structure
------------------

As explained in section \ref{design:data}, the database structure is the complete information (except for caching/pre-computation) about the data Deadlock is concerned with.

Figure \ref{fig:erd} shows the entity-relationship diagram for the basic database schema.

\begin{sidewaysfigure}[h!]
\includegraphics[width=\textwidth]{src/img/erd.pdf}
\caption{Entity-relationship diagram for the database scheme.}
\label{fig:erd}
\end{sidewaysfigure}

### "in expression" pre-computation  {#impl:fun:in_expr}

In addition to the above, the "in expression" relation as described by section \ref{rules:in_expr} is computed from the data.

To save time and resources, this computation is implemented in-database using PL/pgSQL [@plpgsql]. Figure \ref{fig:in_expr_erd} shows the tables used.

\begin{figure}[h!]
\centering
\includegraphics[width=0.5\textwidth]{src/img/in_expr_erd.pdf}
\caption{Entity-relationship diagram for the "in expression" pre-computation.}
\label{fig:in_expr_erd}
\end{figure}

The re-computation function is triggered by changes on the `in_expr_edge` table using the `CREATE TRIGGER` mechanism in SQL. Upon change, it traverses the identity expression DAG recursively, marking what needs recomputing in the auxiliary `_mr_recalculate` table. The recomputation happens inside a transaction, so other queries cannot see partial changes in the `in_expr` table -- integrity during recomputation is ensured.


Some noteworthy implementation issues  {#impl:fun}
-------------------------------------

### The CryptoBox abstraction

In order to avoid accidentally exposing the private keys (e.g. as part of logged tracebacks), and to provide a good abstraction of the cryptography used, we have created the CryptoBox interface: a black box that can perform encryption and decryption for a particular controller. In this way, we avoid passing the secret key directly, thereby reducing the risk that it will end up where it shouldn't. (Naturally, we cannot really hide it from the process, as it must be mapped in the same address space in conventional circumstances, but we can at least avoid revealing it without noticeable effort.) This also abstracts away all of the specifics of the particular cryptographic primitives used, thereby allowing for switching to a different method (e.g. to assymetric cryptography) without needing to change any of the code using the CryptoBox. This API is inspired by the Python API to the NaCl library [@PyNaClDoc].


### The "file version implies file contents" guarantee

The protocol guarantees (section \ref{protocol:xfer}) that a given file version always points to the same contents, to the last byte. In order to be able to guarantee this, we:
- always write into a temporary file and rename it to a recognized filename only after it is ready, relying on atomicity of the POSIX `rename` call within the same filesystem [@posix]; and
- derive the file name from the contents: we compute the 64-bit FNV-1a hash[^fnv] [@fnv] while writing the file and use that as the version.

[^fnv]: The FNV-1a hash algorithm was chosen for low collision probability in the 64-bit variant, and for good performance especially on longer inputs.
