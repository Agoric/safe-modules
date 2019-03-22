
## Appendix A: Definitions

A module is a source file (or textual compilation unit).

The result of loading a module is a module instance.

A pure value:

-   Is everything transitively reachable in the semantic state:

-   Including encapsulated pointers, even if optimizable out

-   Excludes implementation artifacts absent from semantic state

-   Cannot directly cause or sense I/O.

-   Does not provide any ability to cause or sense effects (mutation or I/O).

-   Is not a host object.

-   Contains no semantic mutable state:

-   No configurable or writable properties

-   Captures no var variables

-   Captures no assignable variables

-   Can mutate parameters.

-   Can make fresh mutable state.

Sharing pure values do not provide a communication channel.

Ideally, a pure value is deterministic --- when called with the same
inputs it will always produce the same outcome. However, JavaScript
implementations suffer from fatal errors, such as out of memory,
non-deterministically, and in ways that programs can purposely
provoke. So the most we can realistically require is that pure values
are fail-stop deterministic. When called with the same inputs and
successfully producing a non-fatal outcome, it will always produce the
same outcome.

Note the following:

-   As a legacy compromise, a let variable that is only initialized
    and read, but never assigned to, is considered equivalent to
    const.

-   A lexical variable that might be accessed during temporal dead
    zone is mutable state.

-   Symbol registry is not mutable state. (This section of the
    EcmaScript spec should be rewritten to describe the semantics of
    the Symbol registry in a stateless manner.)

-   Being fail-stop deterministic means that completed calls with same
    inputs must have same completion outcome.

-   Fatal failures, like out of memory, can be non-deterministic

-   Resource use, time, space, GC, can be non-deterministic

-   Debugging info, like stack frames, is available only to privileged
    entities

-   Unprivileged computation cannot sense non-determinism

For example, if a counter has a mutable count and methods for mutating
it, but the execution of the program would never call these methods,
then the counter, in the context of that program, does not actually
bring about the causing or sensing of external effects. But we still
would not call it pure because it violates the structural
characteristics that we reason about to verify that something is pure.

