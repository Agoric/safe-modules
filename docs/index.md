# Safe JavaScript Modules

by Mark S. Miller, Darya Melicher, Kate Sills, JF Paradis


## Introduction

Recently, the [event-stream
incident](https://blog.npmjs.org/post/180565383195/details-about-the-event-stream-incident)
demonstrated that current JavaScript module systems leave application
developers widely vulnerable to malicious attacks. This proposal
outlines a JavaScript module loader that is significantly safer, to
the point of preventing similar malicious attacks entirely.

The goal of this proposal is enforce the [Principle of Least Authority
(POLA)](https://agoric.com/references/#pola) on all imported modules,
meaning that the modules should only be given the authority that they
need to accomplish their tasks and no more. To accomplish this goal,
we must have the ability to isolate modules from each other as well as
the ability to cut off access to powerful resources such as the file
system or network by default.

[Realms](https://github.com/tc39/proposal-realms), an ECMAScript
proposal currently at Stage 2, was proposed by some of us as a
necessary tool to enforce the principle of least authority. The Realm
API enables the creation of distinct global environments. This global
environment, called a realm, has its own global object, copy of the
standard library, and primordials[1]. A realm can contain many
co-existing featherweight compartments. However, with Realms alone,
these compartments do not provide genuine protection from each other.

To that end, we built [Secure EcmaScript
(SES)](https://github.com/agoric/ses) on top of Realms. SES hardens
the root realm such that these compartments are genuine protection
domains. Each compartment shares its root realm's primordials but has
its own global object, global scope, and named evaluators (the eval
function and the Function constructor), which evaluate code in that
compartment's global scope. With SES, we successfully achieve the
isolation of code execution.

However, while Realms and SES provide security, they do not yet
incorporate a module system that easily allows for code reuse. This
document outlines a plan for a module system that is easy to use and
secure enough to prevent incidents like the event-stream exploit.


## Problems with Current JavaScript Modules

Prior to the ES module system, linking various libraries could only be
achieved by mutation and lookup in the global scope. For example,
libraries such as jQuery were imported by loading a script file that
would add a property (such as `$`) to `window`. Mutating the global
scope in this way was accident-prone and unable to scale to multiple
layers of dependencies. To address this problem, two module systems
emerged for JavaScript: Common JS and the EcmaScript module system.

The Common JS module system (CJS) emerged from efforts to run
JavaScript on the server side and became the module system used by
Node.js. Because CJS modules could not be used in the browser, other
module systems emerged, including the standard EcmaScript module
system (ESM), which enables static linkage analysis and is now
available on all platforms. In the meantime, CJS modules became
sufficiently common that host independent CJS modules are now
supported on all platforms, sometimes through "bundlers," code that
transforms a collection of CJS and ESM modules into standard scripts.

Both CJS and ESM have become entrenched to the point that any
realistic module system for JavaScript must accommodate their
co-existence, even though standards documents only acknowledge the
existence of ESM. Collections of both kinds of modules are delivered
in `packages`, often distributed through npm or similar systems. For
npm and some others, a package comes with a manifest, named
`package.json`. There is now a massive legacy of useful
packages. Systems are now built by linking together many modules
written by many different parties into a single program. Often, this
goes well, creating programs of great functionality by composing
together the functionality expressed by this great number of modules.

However, even though an application can be divided into small units of
independent and reusable code, nothing was done to protect these
modules from each other. Currently modules can interfere with other
modules, either unintentionally (as in the case of accidental
tampering) or intentionally, as in the case of malicious code. In both
cases, such interference limits the scale of systems we can compose
reliably. We must find ways to use modules and compose them together,
so that they can cooperate as we intend when things go well, but limit
the damage when things go badly.

The current design of the JavaScript language and module systems give
rise to the following vulnerabilities that allow for destructive
interference:

-   All these modules are typically loaded into one shared realm with
    one set of primordials, one global scope, and one global object,
    all of which are mutable. As mentioned above, this leads to
    accidental or intentional interference between
    modules. Fortunately, a widely recognized best practice is for
    libraries to avoid mutating the shared primordials.

-   Imports are authority bearing. A module not only gains read or
    write access to the global scope where it's imported, it gains the
    same ambient authority as the main program, meaning that it gains
    access to all available APIs with the same execution rights.

-   There is no clear contract between a module and the objects
    accessible via the global scope. Some modules can only work in a
    browser, others only in Node.js.

-   The legacy linking system via global declarations and lookups is
    available and still sometimes used. There is no guarantee that all
    dependencies between modules are expressed by imports and exports
    only.

-   A module isn't limited to declarations. Loading a module executes
    code that initializes the module instance, creating stateful and
    powerful instances that are shared by universal access to one
    shared import namespace.

-   Dynamic imports are not statically analyzable. References to other
    modules can be constructed at runtime.

-   The platform provides the program with ambient authority in the
    form of objects representing powerful and dangerous capabilities
    (such as accessing the file system) populating the global scope
    and global object, available to all modules in that program, even
    though most modules need few or none of those.

-   Through the one shared import namespace, any module can load any
    other module, preventing any encapsulation of internal modules
    within larger subsystems.

-   Modules can hold and share a state, allowing unintended and
    unauthorized communications channels. For example, modules can
    export:

    -   A variable.

    -   A mutable object.

    -   A function which closes over a variable.

    -   An object which can hold a state, for example, a WeakMap.


## Threat Model

We follow the threat model as described in the Wyvern paper [[2](#2)],
including two common scenarios: malicious third-party code and
fallible in-house code. Malicious third-party code is a major
problem. Currently in Node, seemingly simplistic packages can access
powerful resources such as network access, and could be sending our
todo data to anywhere. Furthermore, because they also have access to
the file system, they can send all our data anywhere. As in the Wyvern
paper, we also care about fallible in-house code. For instance, we
might be concerned that our own application will overwrite an
important file if misconfigured.


## Proposal

[TODO: Summary paragraph here]


### Pure and Resource Modules

Our proposal depends on the idea that some modules can be identified
as intrinsically safer than other modules. For instance, a module that
doesn't have access to the file system can't do as much damage as a
module that *does* have access. We therefore define two kinds of
modules: **pure modules** and **resource modules**. These names come
from the Wyvern programming language [[2](#2)], which features a
capability-based module system. (One of the main contributors to
Wyvern, Darya Melicher, is one of the authors of this proposal.) We
adapt Wyvern's definitions as follows:

**Pure values** (pure objects in Wyvern) are values that (see the
detailed [definition below](#appendix-a-definitions)):

1.  do not encompass system resources (i.e. network access),

1.  do not contain or transitively reference any mutable state,

1.  have no side effects.

**Pure modules** are those modules that[3]:

1.  do not encompass system resources (i.e. network access),

1.  do not contain or transitively reference any mutable state,

1.  have no side effects,

1.  do not import any resource module,

1.  only export pure values, and

1.  declare themselves to be pure.

**Resource modules** are defined as modules that either:

1.  encapsulate system resources,

1.  contain mutable state,

1.  have side effects,

1.  use other resource modules, or

1.  export non pure values.

For example, the popular `request` package would be a resource module
because it imports Node's core `http` module which gives access to the
network.

From these definitions, it follows that pure modules can only import
other pure modules. If a module imports a resource module instance it
gets automatically categorized as a resource module itself, even if it
would otherwise be pure.

In addition, we can define a SES root realm as a realm that only
encompasses pure values.


### Loaders

Our distinction between pure and resource modules allows us to treat
them differently. Verifiably pure modules can be loaded into a SES
root realm together and shared with all its compartments without a
loss of security. Resource modules, however, (and this includes
everything that is not verifiably pure), must be loaded into separate
SES compartments.

SES provides a `harden`[4] function (see
[@agoric/harden](https://github.com/Agoric/Harden#harden) for more
information), which performs a transitive walk over properties that
aren't inherited, freezing every object it encounters. Harden
additionally verifies that the objects that these objects inherit from
are themselves already hardened.  Loaders for both pure and resource
modules will apply harden automatically to all exported and imported
values. For example, a loader would apply harden at the following
points:

```js
let x = harden(require("x"));

function foo(y) {
  return [x, y];
}

exports = harden({x, foo});
```

Given the use of harden and the assumption that module "x" is also
verifiably pure, the above module is a pure module. The thought
process of verifying that the above module is a pure module would go
like this:

-   Hardening the returned record freezes that record and hardens the
    function foo.

-   The function `foo` captures the variable `x`, but `x` is a `let`
    that is never assigned to and so is effectively `const`.

-   The value that `x` is initialized to is the result of `require`
    and thus assumed to be pure.

-   No variable can be accessed during their temporal dead
    zone[5]. Within cyclic imports, this gets tricky to verify[6].

-   Note that the `y` parameter of `foo` may not be frozen. The array
    returned by `foo` is not frozen. All this is fine in SES and does
    not violate the purity of `foo`. Pure values can accept, create,
    and return impure values.


### Access Control and Wiring

At this point, we have pure modules together in the root realm and
resource modules in their own compartments. However, we have not yet
described how these modules may be used, or how access to dangerous
resources can be granted. There are two ways in which access control
is specified in our proposed system: a manifest and authority-handling
modules. A manifest is associated with a group of modules, or
`package`, and can be thought of as similar to an npm package's
`package.json` file. A manifest is declarative, and can express how
packages are wired to each other, but can't fully express complex
authority decisions. Complex authority decisions are best expressed in
code, in authority-handling modules. In these modules, the
application's authority can be subdivided, attenuated, or virtualized.


#### Concrete Example: Command Line Todo App

To explain how this would work, let's use an concrete example. Let's
say we are writing a command-line todo app. We can input and display
tasks (we will ignore deleting tasks for now), and the task data is
saved to a file. Our todo application runs on Node.js and imports the
[chalk](https://www.npmjs.com/package/chalk) and
[minimist](https://www.npmjs.com/package/minimist) packages, two of
the [most widely used npm
packages](https://www.npmjs.com/browse/depended).  Chalk displays our
todo list in various colors based on the task's priority, and Minimist
parses our command line arguments for us.

[Minimist](https://github.com/substack/minimist/blob/master/index.js)
is a pure module because it doesn't encompass any system resources,
doesn't import any resource modules, doesn't contain or transitively
reference any mutable state, has no side effects, and only exports a
function for parsing args. We shall assume in our example that it
declares itself pure. Because minimist is a pure module, minimist is
loaded into the root realm.

[Chalk](https://github.com/chalk/chalk), on the other hand, needs to
know what system it is running on in order to know what colors it can
use, so it imports a package called `supports-color`, which imports
Node's `os` module. The `os` module includes functions that return the
current platform and release. It also includes a function to set the
scheduling priority of any process. Because of this, the `os` module
is a resource module. And, because chalk transitively imports the `os`
module, it is also a resource module.

There are two distinct approaches that can be taken to use these
modules securely. First, we can rewrite the modules such that the
dangerous resources are passed in as a parameter and we can pass in
virtualized or attenuated versions instead. Thus, `chalk` exports a
function that has parameters (`os`, `process`) and in turn passes
these on to `supports-color`. The second approach, the legacy
approach, doesn't require a rewrite of the packages. Instead, a
manifest details the wiring that should occur. For instance,
`supports-color` shouldn't get the actual `os` module, but rather an
attenuated version that drastically limits the properties available.

Both of these approaches may be used simultaneously in the same code
base. Modules that can be rewritten in the functional style are
preferred, but we assume that given the vast number of JavaScript
packages, we need to be able to use legacy code as well.

We've created repositories that contain an example for both of these
approaches.

#### Functional Approach

([example code](/examples/clean-todo))

The functional approach is the most straightforward. Our todo app
functionality uses three dangerous resources: the built-in node
modules (`fs` and `os`) and the global variable process. It uses `fs`
directly and `os` and `process` through chalk.

![](./assets/clean-todo-import-diagram.png)

By default, only the todo app itself has access to the build-in node
modules and global variables. By using SES to provide featherweight
secure compartments for loading modules, at this point, chalk and
supports-color are loaded in their own compartments and do not have
any access to `os` and `process`. We need to figure out how to grant
access selectively and securely. Chalk, being a third party module,
fits our first threat model of possibly malicious third party
code. The direct use of `fs` by our todo app fits our second threat
model of fallible in-house code. Furthermore, this example contains
both resource modules and global variables, allowing us to show how to
handle both.

Let's start with a working version using current Node.js. The file
starts like this:

```js
const fs = require('fs');
const parseArgs = require('minimist');
const chalk = require('chalk');
const todoFile =  'todo.txt'
const addTodoToFile =  (todo, priority='Medium')  =>  {
  fs.appendFile(todoFile, `${priority}: ${todo} \n`,  (err)  =>  {
    if  (err)  throw err;
      console.log('Todo was added');
    });
  }
  ...
```

The first step we need to take to enforce POLA is to attenuate the fs
module. By attenuate, we mean reducing the authority to only what is
essential. We can create a new module attenuate-fs to do this:

``` js
const harden = require('@agoric/harden');
const todoPath =  'todo.txt';
const checkFileName =  (path)  =>  {
  if  (path !== todoPath)  {
    throw Error(`This app does not have access to ${path}`);
  }
};

const attenuateFs =  (originalFs)  => harden({
  appendFile:  (path, data, callback)  =>  {
    checkFileName(path);
    return originalFs.appendFile(path, data, callback);
  },
  createReadStream:  (path)  =>  {
    checkFileName(path);
    return originalFs.createReadStream(path);
  },
});

module.exports  = attenuateFs;
```

Now, instead of using the original `fs` when we try to append a todo,
we can use an attenuated version that will error if we try to 1) use
any fs functionality other than appendFile and createReadStream, or 2)
if we try to call these methods with any file other than our
predefined todo.txt file. The attenuated version looks like this:

```js
const fs = require('fs');
const parseArgs = require('minimist');
const chalk = require('chalk');
const attenuateFs = require('attenuate-fs');
const altFs = attenuateFs(fs);

const todoFile =  'todo.txt';

const addTodoToFile =  (todo, priority =  'Medium')  =>  {
  altFs.appendFile(todoFile, `${priority}: ${todo} \n`, err =>  {
    if  (err)  throw err;
    console.log('Todo was added');
  });
};
...
```

Thus, we've given our own code the least authority version of the file
system - only the one file and only certain functions with that
file. In a much larger example, attenuation like this can help a great
deal with fallible in-house code that may accidentally misuse
authority.

But let's turn to the problem of potentially malicious third party
code. Minimist is a pure module, so we are ok giving it zero access to
the external world and using the pure module loader to check that it
matches our definition of a pure module. Chalk, on the other hand,
needs access to os and process to do its legitimate job, so we need to
give access to those resources but at the same time, ensure that no
additional access is provided.

We start by rewriting chalk as a function that takes in `os` and
`process`(see
[diff](https://github.com/chalk/chalk/compare/master...katelynsills:master)):

```js
const pureChalk =  (os, process)  =>  {
const stdoutColor = pureSupportsColor(os, process).stdout;
...
```

Now we need to write rewrite supports-color in the same way
([diff)](https://github.com/chalk/supports-color/compare/master...katelynsills:master):

```js
const pureSupportsColor =  (os, process)  =>  {
const  {env}  = process;
...
```

Note that supports-color no longer imports `os` itself and it only has
access to the `os` and `process` passed to it. It has no access to the
`process` variable provided by Node.js.

Now that we are in control of the resources that chalk and
supports-color are using, let's attenuate them before passing them
on. Going back to our todo-app index.js, it would look like this:

```js
// built-in modules
const fs = require('fs');
const os = require('os');

// our rewritten modules
const parseArgs = require('minimist');
const pureChalk = require('chalk');

// our attenuating modules
const attenuateProcess = require('attenuate-process');
const attenuateOs = require('attenuate-os');
const attenuateFs = require('attenuate-fs');

// attenuate
const altProcess = attenuateProcess(process);
const altOs = attenuateOs(os);
const altFs = attenuateFs(fs);
const chalk = pureChalk(altOs, altProcess);

const todoFile =  'todo.txt';

const addTodoToFile =  (todo, priority =  'Medium')  =>  {
  altFs.appendFile(todoFile, `${priority}: ${todo} \n`, err =>  {
    if  (err)  throw err;
    console.log('Todo was added');
  });
};
...
```

And attenuate-os and attenuate-process would look like this:

```js
const attenuateOs =  (originalOs)  =>
  // we know the result is pure
  harden({
    release: originalOs.release,
  });

module.exports  = attenuateOs;
```

```js
const attenuateProcess =  (originalProcess)  =>
  // this is not pure - stdout and stderr are resources
  harden({
    env: originalProcess.env,
    platform:  'win32',
    versions: originalProcess.versions,
    stdout: originalProcess.stdout,
    stderr: originalProcess.stderr,
  });
module.exports  = attenuateProcess;
```

Now, chalk and supports-color cannot use os to do dangerous things
like change the priority of processes.

But what happens if we don't have the opportunity to rewrite the
modules we want to use? Then we can use the legacy approach.

#### Legacy Approach

([example code](https://github.com/katelynsills/legacy-todo))

In the legacy approach, our todo app code doesn't do the attenuating -
it looks like a normal Node.js app again. We also don't touch chalk or
supports-color. To control the access that chalk and supports-color
have, we instead use a manifest to describe the relationship, and then
our loaders enforce the manifest, including using attenuating
modules. The manifest would look like your normal package.json, but
with an additional property, `resources`:

```json
"resources":  {
  "index": {
    "modules": {
      "fs":  "alt-fs",
      "chalk": true
    }
  },
  "chalk":  {
    "modules":  {
      "supports-color":  true
    }
  },
  "alt-fs":  {
    "modules":  {
      "fs":  true
    }
  },
  "alt-os":  {
    "modules":  {
      "os":  true
    }
  },
  "alt-process":  {
    "globals":  {
      "process":  true
    }
  },
  "supports-color":  {
    "modules":  {
      "os":  "alt-os"
    },
    "globals":  {
      "process":  "alt-process"
    }
  }
}
```

Instead of `fs`, our todo-app index.js is handed the `alt-fs` module
by the module loader. From the todo-app perspective, it thinks it's
getting `fs`, and would only find out otherwise if it goes outside the
bounds of what is provided to it.

`alt-fs` is simply the attenuated version of `fs`, exported as a module.

```js
const attenuateFs = require('attenuate-fs');
const fs = require('fs');
module.exports  = attenuateFs(fs);
```

We build similar attenuating modules for `os` and `process` - `alt-os`
and `alt-process`.


### Implementation Details

#### The Manifest

-   Modules: Modules can only import what their manifest says they
    depend on. This is enforced by having no access by default and the
    loader inserting ??

-   Globals: Which non-pure-JS global variable names it uses, i.e.,
    which non-whitelisted variables it uses freely. From the free
    variables in the modules, we can check or generate this part of
    the manifest. The manifest describes which of these variables are
    assigned to.

-   Whether it accesses its global object, i.e., by a top-level
    this. We do not assume that a module that says window freely
    accesses the global object. We treat this as any other global
    variable reference. Currently this is not yet an issue for
    modules, but may become one depending on
    [tc39 proposal-global](https://github.com/tc39/proposal-global).

-   Which properties of the global object it uses via property
    access. This cannot be accurately determined statically, but can
    be under-approximated.

-   Which of its modules it alleges are pure and should therefore be
    loaded by the root-realm's pure loader if it is indeed pure. Such
    pure modules are not multiply instantiated and are instantiated
    separately from their package. Thus, modules declared pure do not
    cause identity discontinuities.

A new component of an application's startup logic is a configurator,
which expresses the application's policy decisions about how top-level
packages are wired to each other and how the application's authority
is to be subdivided, attenuated, or virtualized among them. It does
this by:

-   Creating the compartments

-   Assigning each top-level module-group instance to a
    compartment. Note that the same package may be instantiated in one
    compartment one way and in another compartment another way. Such
    multiple instantiation will cause identity discontinuities.

-   Populating each compartment's global object and/or global lexical
    scope with the objects that module-group instance should be able
    to access freely. Because some of these are interesting
    attenuations or virtualizations, this cannot be expressed purely
    declaratively but must be able to contain code.

-   All variable names that the manifest says are used as global
    variables but not as properties of the global object and are never
    assigned to can be placed into the compartment's global lexical
    scope rather than its global object.

-   If a global variable or property is assigned to and is shared
    across compartments, we have to do an interesting dance, defining
    it as an accessor property on all these global objects. Hopefully
    we can just reject all these now and postpone this issue till much
    later.

-   Wiring the loaders to each other according to which module-group
    instance should be able to import/require module instances from
    which other module-group instance. This should project into the
    declared static dependencies among packages.

-   Note that all import/requires of a module declared pure always
    delegates to the root realm's purity-enforcing loader.


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


## Appendix B: Rules for a Purity Checker

This purity checker would verify modules as pure, conditional on the
modules it imports also being pure.

We aren't trying to write the purity checker such that it transitively
applies to the modules it imports because that makes things like
cycles hard to check. Rather, if a given loader only imports
conditionally-pure modules, i.e., modules that would be pure if their
imports are pure, than a self contained graph of such conditionally
pure modules should be pure. Thus, a conditionally pure module is pure
when loaded by that loader.

We can assume that the loader hardens our exports. We don't have to
harden the exports ourselves.

A value is purifiable if hardening the value makes it pure. Let's use
the following examples to explain this further:

```js
function makePoint(x, y) {
 return {x, y}
}

function makeCounter(count) {
 return {countUp() { return ++count; }}  
}
```

Both makePoint and makeCounter are purifiable - if hardened they're
pure. The record returned by makePoint is purifiable if and only if x
and y are purifiable. The harden function will reach both x and y and
attempt to harden them as well. If harden returns successfully, then
the record, the value of x, and the value of y are all hardened. If
harden throws, then we do not assume any of these are hardened.

If you harden the counter returned by makeCount, harden will be
applied to the countUp method. But the countUp method captures a
lexical variable that is assigned to, and therefore the countUp method
is not purifiable, and therefore the object containing it is not
purifiable either.

A purifiable expression is one that we know evaluates only to
purifiable values.

Now let's take this example:

```js
function makePoint(x, y) {
 return {x: +x, y: +y}
}
```

'+' coerces to a number, and now we know that the x and y properties
are numbers, and the object that is returned is therefore purifiable.

Our starting rule for determining that a module is pure is that it is
only exporting purifiable values, that is, values that become pure
when hardened.

In order to know that we need to take a look at the expression
evaluating to those values.

We need to make sure that the value of the exports variable at the
moment that the module finishes executing is itself purifiable. The
rules that we conservatively apply will support exports = purifiable
expression.

We won't try to support the incremental export yet. In other words,
 having export.foo = bar throughout the module file. We can come back
to this.

We've now reduced the problem to the individual kinds of expression:

-   An expression that is just a constant value, like 3, is obviously
    purifiable because it's already pure.

-   An expression that is a variable name like x, to check if it's
    purifiable, check two conditions. We look up the definition of x
    and see if we can verify that x is initialized to a purifiable
    value, and we also check that x is never assigned to.

-   To verify a function call, you'd have to look at the function
    definition, and if you don't have the function definition in the
    same module, then you say that a call to a function that isn't
    locally defined is disqualified. The expression that evaluates to
    the arguments, each of those expressions have to be pure (harden
    already applied) - not just purifiable - expressions. The
    transitive hardening rules won't reach the arguments. Then we can
    analyze the body of the function assuming that the parameters are
    pure. In makePoint, if we see a call to makePoint with pure
    arguments, then we know it's returning a purifiable object. If we
    see a call to makePoint with arguments that aren't pure, then have
    to say the function call isn't purifiable. We would then determine
    whether any value returned is purifiable - if so then the function
    call is purifiable.

-   Arguments to the function call - the implicit `this` argument must
    be pure.

The method call expression bob.foo(carol) - we just say this is not
purifiable for now - it's too hard to reliably figure out what
function is being called. Given that we reject method calls, we can
consider `this` to be purifiable. If we ever did accept method calls,
we would have to ensure that bob (the implicit this, in the example)
is pure as well as carol is pure.

-   A function expression is purifiable if every variable that it
    captures is not assigned to and each of those variables are
    initialized to pure values. For example:

```js
let x = {};
function f() { return x };
```

Even though the empty object x is purifiable, f is not purifiable in
this example, and the reason is that hardening f does not harden
x. Therefore f can be used as a communications channel. For example,
Alice and Bob could both have a hardened f and then they both call it,
they get the same mutable object, and can use it communicate with each
other. By contrast:

```js
let x = harden({});
function f() { return x };  // f is purifiable
```

The function's prototype property has to be a purifiable value. If the
function's prototype is never mentioned in the module, it is still
purifiable, because the prototype is initialized to an empty object
that inherits from Object.prototype.

-   Object literal - the object literal syntax can express
    initializations of its properties, and it can express what it
    inherits from. The simple statement of what we are trying to
    achieve: the object literal is purifiable if the state of what it
    inherits from is a pure expression - it has to inherit from a pure
    object, and if for its properties, we need to distinguish between
    data and accessor properties. A data property has to be
    initialized to a purifiable expression and there are two cases of
    that - "propertyName :" and the method syntax. We have to insist
    that the method itself in the method syntax is purifiable. The
    accessor properties which are the things with get and set -
    there's two function definitions, the getter function and the
    setter function and we have to insist that both of the functions
    are purifiable. Harden is doing a transitive reflective property
    walk of the own properties where for each property it does an
    operation "getOwnPropertyDescriptor" If it's a data property then
    it walks the value of the data property. If it's an accessor
    property it checks the getter function and the setting function
    but does not invoke the getter. If you did a normal access (a
    non-reflective one) you would be invoking the getter, not
    obtaining the getter function.

-   Array literal - this is purifiable if all the argument expressions
    are purifiable.

-   Regex literal - unconditionally purifiable.

-   Classes - A class is purifiable if it contains no elements that
    are beyond the ES2017 standard. Elements from proposals that are
    still in flux at the time of this writing (such as decorators)
    should fail the purity checker. Purifiable if all of its static
    properties are purifiable, if all of its methods are purifiable,
    and if it extends (i.e. inherits from) a pure class (not just a
    purifiable class).

-   Construct / 'new' expression - we treat it like a function
    call. The implicit arguments include the function itself, what we
    are doing the new on. That has to be pure, not just purifiable,
    and all the explicit arguments need to be pure. And the function
    itself needs to be visible in the same way as a normal call
    (within the same unit of analysis, the module). Thus, the same
    analysis that we do with functions is safe to apply to the new
    case. You can also do a new on a class, which makes an instance of
    the class. We consider all instances to be not purifiable for now,
    which can be revisited later if necessary.

-   If the value of a purifiable expression is subsequently mutated,
    like by property assignment, before they are hardened, we have to
    assume that they aren't purifiable.

-   We would need to do a deeper analysis - take a look at everywhere
    that the object may have gotten to, and we have to check two
    things -

-   The purifiable value doesn't escape from the module before it is
    exported. For example, you could import something, and pass the
    object to it. Without violating purity, the receiver could mutate
    this argument before it is hardened.

-   We need to make sure that there is no code within this module that
    directly mutates it, such as by property assignment.

For example, let's say we have a function foo, and somewhere in the
module an expression like foo.x = [something not purifiable], then we
have to say that foo is not purifiable.

We will go even stricter - If foo.x = [anything], then we say foo is
not purifiable.

-   Property lookup foo.x - not a purifiable expression. Same issue as
    with method calls. We may try to make this purifiable later, but
    we'd have to be very cautious.

-   If you just do foo.x in the module, does that make foo itself no
    longer purifiable? I think the answer is yes. Just property lookup
    can cause side effects, and it would take a deep analysis to be
    able guarantee that it isn't causing side effects.

-   Implicit coercions - if you have an expression like x + y, that
    expression might cause the toString() or valueOf() method of x or
    y to be invoked. Those could cause side effects and that could be
    before the purifiable things are hardened. In order for an object
    literal to be considered purifiable, we will start off saying that
    it doesn't have a toString or valueOf method. Later we can relax
    it, but for now we need to say those make it not purifiable.

-   Weird new things to reject because too complicated - dynamic
    import expression, import.meta expression. If we see either of
    those in the module, we reject the whole module. Same thing with
    direct eval syntax - reject the whole module.

-   Rejected if there are var declarations, we are only processing
    let, const, function and class.

-   Temporal dead zone - In JavaScript a variable can be accessed
    before it is initialized.

```js
{
  // this throws an error - the variable is in scope but 
  // in an uninitialized state
  foo();
  let x = 3;
  function foo() {
    return x;
  }
}
```

Temporal dead zone: the interval of time from when the variable comes
into existence, until the variable is initialized. If the variable is
accessed, by reading or assigning to it, before it is initialized,
then an error is thrown. The result is that this is an observable
change of state and therefore a possible communications channel.

TC39 would have prefered a static guarantee that the variable cannot
be accessed during the temporal dead zone, but for backwards
compatibility we gave up on trying to have a static rule in the
standard because we would have to reject too many common coding
patterns. The purity checker needs to make a static check and thus
needs to be stricter than we (as tc39) were willing to be for the
language in general, so the purity checker must verify that all
potential significant variables are never accessed during their
temporal dead zone.

We can verify this by adopting Doug Crockford's rule that a variable
can only be used textually below the declaration (i.e. only in later
statements) - it has to verify this for all variables, not just the
one in question. If this is true of all variables, our example of this
would be correctly identified as not pure. We need to make one
exception - a consecutive sequence of function declarations is
considered one unit, where the functions within the unit can refer to
each other even if a use is above a declaration - i.e. in the case of
mutually recursive functions.


## References and Footnotes

<a name="1">[1]</a> Primordials are all of the JavaScript objects that
are mandated by the ECMAScript spec to exist before the code starts
running, but not including the global object. Host-mandated objects,
such as document or require, are not primordials. All intrinsics are
primordials.

<a name="2">[2]</a> A Capability-Based Module System for Authority
Control. Darya Melicher, Yangqingwei Shi, Alex Potanin, and Jonathan
Aldrich. European Conference on Object-Oriented Programming (ECOOP),
2017.

<a name="3">[3]</a> These definitions assume that the module’s exports
are hardened by the loader, and that the module’s imports are
themselves pure as enforced by the loader.

<a name="4">[4]</a> Previous named `def`

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
SCC as a whole only imports pure modules and only exports pure
values. However, this requires that the whole SCC be available for one
joint static analysis.

-----

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
work (or be easily changed to work) with this module
system. Experience with Caja at Google and the Locker Service at
Salesforce shows that there is indeed a tremendous amount of existing
code that can operate within these constraints.

### Simultaneous Constraints

-   Resource (stateful) modules expect to share by common import, at
    least within a package.

-   No one author knows all the included modules or imports needed,
    and so no one author can write policy for all authority
    subdivision.

-   Author of separately authored subsystem must be able to revise
    internals without breaking the enclosing system.

-   Full import graph must be statically resolved before modules execute.
