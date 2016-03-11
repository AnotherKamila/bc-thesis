Introduction {.unnumbered}
============

Project Deadlock is a system that controls access to a number of points of
access (eg. doors, appliances) using RFID cards. Deadlock is designed for
security and reliability, assuming untrusted and unreliable network. Unlike existing commercial solutions, Deadlock is fully open-source and open-hardware, and designed  to be flexible, maintainable, and cost-effective. We provide tools and expose all interfaces and components, making Deadlock easy to integrate with existing systems and customize to the needs of the user.

Deadlock is a project of the Student Development Team[^svt] at the Faculty of
Mathematics, Physics and Informatics of Comenius University. It is implemented
by students and supervised by faculty members.

[^svt]: [http://svt.fmph.uniba.sk](http://svt.fmph.uniba.sk)

This thesis first introduces the requirements/specification (chapter \ref{spec}) and the high-level design choices we made to fulfill it (chapter \ref{overview}). These were developed jointly by the Student Development Team. We then focus on the author's contribution: the server/controller communication protocol (chapter \ref{protocol}) and the server design and implementation (chapter \ref{server}). Then we look at the future plans (chapter \ref{future}).
