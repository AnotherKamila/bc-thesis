Introduction {.unnumbered}
============

Project Deadlock is a system that controls access to a number of points of
access (e.g. doors, appliances) using RFID cards. Deadlock is designed for
security and reliability, assuming untrusted and unreliable network. Unlike existing commercial solutions, Deadlock is fully open-source and open-hardware, and designed  to be flexible, maintainable, scalable, and cost-effective. We provide tools and expose all interfaces and components, making Deadlock easy to integrate with existing systems and customize to the needs of the user.

Deadlock is a project of the Student Development Team[^svt] at the Faculty of
Mathematics, Physics and Informatics of Comenius University. It is implemented
by students and supervised by faculty members.

[^svt]: [http://svt.fmph.uniba.sk](http://svt.fmph.uniba.sk)

This thesis first lists the requirements (chapter \ref{requirements}) and introduces the high-level design choices we made to fulfill them (chapter \ref{overview}). These were developed jointly by the Student Development Team. We then focus on the author's contribution in the rest of the thesis. Chapter \ref{protocol} describes the design and implementation of the server/controller communication protocol, focusing on the conflicting requirements of reliability, extensibility and simplicity. Chapter \ref{rules} describes the access rules format and evaluation, especially the compromise of generality vs. user-friendliness. Chapters \ref{design} and \ref{impl} provide an overview of the server design and implementation, telling the tale of how good design choices and modularity led to a clean and simple implementation. We conclude with the future plans for Project Deadlock (chapter \ref{future}).
