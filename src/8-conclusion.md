Conclusion {.unnumbered}
========================

Project Deadlock is still a work in progress and not yet production-ready. In particular, specialized controller hardware and firmware, and auxiliary tools such as data importing utilities or rule management UIs, are not ready yet. However, the main challenges have been solved, and the requirements fulfilled. The server is ready to be deployed at small scale for testing/evaluation purposes.

The unexpected challenges we encountered included:

- Security: we did not take security lightly, but the first version of the communication protocol was still prone to replay attacks.
- Sheer size: despite the efforts to keep things as simple as possible, the Deadlock server, as well as the other components, turned out to be more complex than originally envisioned. If not for the good design choices, implementing the server would be impossible within the given time and personnel constraints.
- Constant need to refactor: in many cases, the first attempt at something was not an adequate solution, despite devoting a lot of effort to thinking before typing. Fortunately, the modular architecture of Deadlock meant that the cost of experimenting with multiple approaches was quite low.

The author is particularly pleased by the effects of good design on the resulting software, as well as on the development process. Starting from the base principles of simplicity, statelessness and modularity, we came up with  a simple and consistent set of abstractions. These made the implementation relatively straightforward and enjoyable (in contrast to what would normally happen in a project of this scale). Perhaps even more importantly, they transformed many difficult problems into trivial ones, sometimes unexpectedly so. In the author's opinion, Deadlock is a very good example of how good design choices led to a clean and elegant implementation.
