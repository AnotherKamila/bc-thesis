Access rules  {#rules}
============

Generality vs. convenience
--------------------------

The key observation when designing the access rules is: 

*Generality comes at the cost of complexity. For any given application, most rules will look the same, and therefore if the rules are general, then they will be unnecessarily complex in the typical use case. Most of the time, the user will be annoyed by inputting similar rules every time, instead of making use of the generality.*

Because of this problem, we have decided to create two distinct layers of access rules: a low-level layer, which is general and simple, and a domain-specific high-level layer, which is optimized for the typical usage in the given domain. The high-level layer builds on the primitives provided by the low-level layer, and different high-level rules should be developed for different use cases (such as campuses vs. hotels). This allows for flexibility and convenience at the same time.

In order to support both the typical use case and the unique snowflakes in a single installation, high-level rules implementations are expected to not assume anything about the rules installed in the system -- they are not allowed to carelessly delete existing rules, or assume only rules they know about exist. They should display the low-level representation for rules they cannot interpret in their high-level model.

To facilitate this cooperation between high-level and low-level rules, and also to ensure consistency, we have come up with the notion of a *ruleset*: every rule in the system is tagged as belonging to exactly one ruleset, and the high-level layer can create, update or delete only whole rulesets, not individual rules. Operations on rulesets are atomic. An application implementing the high-level rules should operate only on rulesets created by that application. A mechanism for enforcing this restriction exists.[^rowlevel]

[^rowlevel]: This is implemented in our DBMS of choice, PostgreSQL, by the row-level security mechanism [@pgrowlevel].

The low-level, internal rules must be generic enough to support any use case, yet easy to compile by both computers and humans.

Rules format  {#rulefmt}
------------------------

In order to cover all possible use cases, the straightforward approach is to allow access rules to be specified as any Boolean formula over identities, access points and time specifications. However, this brings the following problems:

 - it is hard for humans to quickly reason about the result of any given query
 - complete evaluation on every query is necessary, which might be costly in memory or time; it is impractical to pre-compute anything
 - for offline functionality, the evaluation logic and all data required for evaluation need to be embedded in the controllers, which violates the "keep embedded devices simple" design principle;
 - a small change in input data or formulas can have arbitrarily large effects, which hinders attempts at both understanding why something happens and pre-computation.

In order to avoid these problems, we have instead chosen the following model:

**Every access point is of exactly one _type_; for each type, rules that match a _time specification_ and an _identity expression_ to an _Allow_ or _Deny_ response may be added. Rules are strictly ordered by priority.**

The evaluation flow, as depicted in in figure \ref{fig:rule-eval}, is as follows:

1. Find this AP's type, select its rules.
2. Select rules with matching time specification.
3. Select rules with this ID in the rule's expression.
4. Select the (single) rule with the highest priority.

This selects a single rule, which unambiguously allows or denies access.

\begin{figure}[h]
\input{src/img/rule-eval.pdf_tex}
\caption{Rule evaluation flow}
\label{fig:rule-eval}
\end{figure}

### Identity expressions

An identity expression is a (restricted) Boolean formula over identities only, and it implements a generalization of access control by groups.

Implementing general Boolean formulas (e.g. using AND, OR and NOT operators) would be possible, but to support NOT we would have to either store the complement, which may require a lot of memory for a small input, or make the computation less straightforward, which clashes with keeping controllers simple. Therefore identity expressions use the INCLUDE and EXCLUDE operators, which are equivalent to set union and set difference. These are equivalent (even in expression complexity) to general Boolean formulas as long as the set of "interesting" identities is given (which it is, as "any ID whatsoever except for this one" is not a useful rule).

### Rationale for the separation

Splitting rule evaluation into identity expressions and time+place expressions means that rules are easier to evaluate: a human (or a computer) can evaluate the two independently, and "why does this happen" questions are easier to answer. Moreover, in this way classes of equivalence on inputs are easier to find, as in this model a single time+place rule matches a single identity expression instead of arbitrary combinations. This makes it practical to pre-compute some rules, and implement re-computing this incrementally on change.


Answering access queries
------------------------

Typically, rules will be queried often (especially when creating local rules databases for controllers) and changed infrequently. Therefore we can save work and time by pre-computing some information. Currently, we assume that in a typical deployment there will be few rules and multi-level identity expressions. Therefore we pre-compute an "in identity expression" relation -- for every identity (i.e. for all leaves of the expressions) we climb the expression DAG and save the $(identity, expression)$ tuple when the identity is included by an expression (taking into account the INCLUDE/EXCLUDE operations). As the expressions are acyclic, whenever we need to INCLUDE/EXCLUDE a sub-expression, we can just wait until we've computed it, thereby computing everything once.

An access query selects a single rule (and a single expression) via the mechanism described in \ref{rulefmt}, and then we just check whether an $(expression, identity)$ tuple exists to answer the query.

When an identity expression changes, it is easy to incrementally re-compute only the affected parts: we simply search the DAG, re-computing nodes as we visit them.

See section \TODO{} \ref{??} for notes on the implementation of re-computation.


Local evaluation on embedded devices
------------------------------------

The local database copy on the controllers builds on this two-level approach of separating identity expressions and time specifications. Note that any embedded device serves a single point of access, and therefore the "where" part of the rules is already taken care of -- every accesspoint selects only rules belonging to its type.

The server provides a mechanism to re-create the local databases on rules change. The specific format of the local databases is out of scope of this thesis.


Integration with existing systems
---------------------------------

As required by section \ref{??}, data may be imported from other systems, and transparently "patched" into access rules. This is done via an application that generates flat identity expressions of the form $X := [\textrm{INCLUDE}\; student_1, \textrm{INCLUDE}\; student_2, ...]$ for every group $X$ that needs to be imported. These groups are marked and considered to be primitives, and they may be modified only by creating a group $Y$ that INCLUDEs $X$ and further INCLUDEs or EXCLUDEs what needs to be "patched" in the imports.
