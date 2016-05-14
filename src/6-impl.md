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

- The Deadlock server is meant to be run on **a Unix-like server**. While it may work on multiple platforms, it is tested only on Debian-like Linux distributions and FreeBSD 10.
- We are targeting **Python 3.4 or newer**. This is because at the time of writing Python 3.4 is available in all relevant OSs and distributions (in particular Debian Stable), and contains useful features not present in previous versions.
- For the DBMS, **PostgreSQL >= 9.3** is required. We use non-standard Postgres-specific features, such as the PL/pgSQL in-database procedural language [@plpgsql] for rules pre-computation, or the NOTIFY/LISTEN pub/sub system for notifying `deadaux` of access rules changes.
- Several of the used libraries (at least `psycopg` and `pynacl`) use native bindings, and therefore only work with the CPython implementation of Python 3.


Database structure
------------------

As explained in section \ref{design:data}, the database structure is the complete information (except for caching/pre-computation) about the data Deadlock is concerned with.

Figure \ref{fig:erd} shows the entity-relationship diagram for the basic database schema.

\begin{sidewaysfigure}[ht]
\includegraphics[width=\textwidth]{src/img/erd.pdf}
\caption{Entity-relationship diagram for the database scheme.}
\label{fig:erd}
\end{sidewaysfigure}

In addition to this, the `in_expr` table as described by section \TODO is computed from the data. This computation is implemented in-database using PL/pgSQL [@plpgsql].


Interesting problems encountered
--------------------------------

\TODO{put it here, or have own sections about "stuff stuff problem solution stuff stuff"?}


- avoid running around with secret keys by passing just a crypto black box (TODO but do it :D) to avoid e.g. accidentally logging it
- guarantees:
    - blob version --> contents: This is implemented by using a blob's hash as the version, but this is an implementation detail not relevant for the protocol



-----------------------------------------------------------------------

References: [@UllmanDB]
