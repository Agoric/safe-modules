
## Misc to be reorganized:

We treat require, exports, module, and def as keywords. We prevent
their shadowing.

The argument to require can only be a literal string.

A root realm has a root loader which only loads pure modules. This
loader is configured by the realm's creator or during the realm's
creation, such as where it finds its source files.

Each compartment has its own compartment loader which can load
resource modules. Thus, loaders and realms have a one-to-one mapping,
with the additional constraint that a SES root realm has a
purity-enforcing loader. This purity-enforcing loader must be
sufficiently pure, which we need to state carefully.

It is acceptable for much existing code, even modules following best
practices, to be rejected by this module system. However, for the
system to be successful, a critical mass of available libraries should
work (or be easily changed to work) with this module system.
Experience with Caja at Google and the Locker Service at Salesforce
shows that there is indeed a tremendous amount of existing code that
can operate within these constraints.

### Simultaneous Constraints

-   Resource (stateful) modules expect to share by common import, at
    least within a package.

-   No one author knows all the included modules or imports needed,
    and so no one author can write policy for all authority
    subdivision.

-   Author of separately authored subsystem must be able to revise
    internals without breaking the enclosing system.

-   Full import graph must be statically resolved before modules
    execute.



    -----

     The thought
process of verifying that the above module is purifiable would go like
this:

-   Hardening the returned record freezes that record and hardens the
    function foo.

-   The function `foo` captures the variable `x`, but `x` is a `let`
    that is never assigned to and so is effectively `const`.

-   The value that `x` is initialized to is the result of `require`
    and thus assumed to be pure.

-   No variable can be accessed during their temporal dead zone
    [[5](#5)]. Within cyclic imports, this gets tricky to verify
    [[6](#6)].

-   Note that the `y` parameter of `foo` may not be frozen. The array
    returned by `foo` is not frozen. All this is fine in SES and does
    not violate the purity of `foo`. Pure values can accept, create,
    and return impure values.



### Concerns / Problems we anticipate

In the root realm, the purity-enforcing loader violates one of the
fundamental constraints of normal purity: it causes external I/O. In
particular, asking it to load a module named "foo" causes it to go out
to the file system or the network, and retrieve a file or http
resource named "foo" in some namespace to serve as the source text of
that module. Depending on who can observe this traffic, this can be
used to communicate externally. Instead, just as we allow a program to
rely on the unobservability of its underlying implementation memory,
we allow SES programs to rely on the stability and unobservability of
this external source of module source text, whatever it is. This
reliance statement is not a source of confidence. Rather, it is a
shifting of responsibility. Just as the implementation memory is part
of the platform trusted computing base (TCB) for programs, the code
that sets up and configures a realm is part of the platform TCB for
code running within the realm. Thus, we consider the root realm
purity-enforcing loader to be pure, enabling us to continue to
consider a SES root realm as a whole to be pure even though it now
contains this loader. For the API used to configure a realm, we need
to clearly document this responsibility, the dangers, and how to cope.


### Benefits

#### Support for the Realm proposal

For engine authors, supporting a safe module system is a much more
tangible security benefit than a low-level API like Realms.

#### Incentive to create pure modules

The creation of a compartment and re-evaluation of the source code of
resource modules requires some overhead, whereas creating pure modules
incurs no overhead.


### Implementation

Supporting CJS modules first will help us support EcmaScript modules
later. For CJS modules, we can shim loader we'd like.

We can do a transitive freeze (def) of all imports/exports in general
without breaking much.

Given that freeze/def, many purifiable modules won't need a rewrite.

[Mark explains here how packages can be
contained](https://docs.google.com/document/d/1C-TvgKiEIfTgWqALRyk-1x0vggWn4VnDR-EL28LlhzA/edit?usp=sharing).



<a name="5">[5]</a> The temporal dead zone is the interval of time
from when the variable comes into existence, until the variable is
initialized. If the variable is accessed, by reading or assigning to
it, before it is initialized, then an error is thrown. The result is
that this is an observable change of state and therefore a possible
communications channel.

<a name="6">[6]</a> It is differently tricky for CJS and EcmaScript
modules and weird when these are mixed. If needed, we can treat an SCC
(strongly connected component, i.e., an import cycle among modules) as
a single module for the purpose of analysis, and require only that the
SCC as a whole only imports pure modules and only exports pure values.
However, this requires that the whole SCC be available for one joint
static analysis.
