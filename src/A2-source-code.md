\newpage

# Appendix: Source code and documentation

Project Deadlock is and will for some time remain under development. The newest source code, hardware schematics and documentation for all components is available at [https://github.com/fmfi-svt-deadlock/](https://github.com/fmfi-svt-deadlock/).

Due to time constraints, the attached source code does not yet implement everything as described in the thesis. The notable differences are:

 - parts of the HTTP API are not implemented yet (section \ref{deadapi});
 - identity expressions precomputation is not incremental (section \ref{impl:inexpr});
 - the `CRITICAL` message handler is not implemented yet (section \ref{protocol:critical});
 - the command-line interface directly queries and modifies the database instead of contacting the HTTP API.

-------------------------------------------------------------------------

\noindent\textbf{Attached}: CD with server source code, as of May 16, 2016.[^1]\
The newest version can be found at [https://github.com/fmfi-svt-deadlock/server](https://github.com/fmfi-svt-deadlock/server).

[^1]: It is strongly recommended to look at and use the newer online version rather than the attached version of the code.
