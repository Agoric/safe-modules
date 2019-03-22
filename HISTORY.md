## The History of JavaScript Modules

Prior to JavaScript module systems, linking various libraries could
only be achieved by mutation and lookup in the global scope. For
example, libraries such as jQuery were effectively "imported" by
loading a script file that would add a property (such as `$`) to
`window` (the browser's global object). Mutating the global scope in
this way was accident-prone and unable to scale to multiple layers of
dependencies. To address this problem, two module systems emerged for
JavaScript: the Common JS module system (CJS) and the standard
EcmaScript module system (ESM).

CJS emerged from efforts to run JavaScript on the server side and
became the module system used by Node.js. Because CJS modules could
not be used in the browser, other module systems emerged including
ESM, which enables static linkage analysis and is now available on all
platforms. In the meantime, CJS modules became sufficiently common
that host independent CJS modules are now supported on all platforms,
sometimes through "bundlers," code that transforms a collection of CJS
and ESM modules into standard scripts.

Both CJS and ESM have become entrenched to the point that any
realistic module system for JavaScript must accommodate their
co-existence, even though standards documents only acknowledge the
existence of ESM. Collections of both kinds of modules are delivered
in `packages`, often distributed through npm or similar systems. For
npm and some others, a package comes with a manifest named
`package.json`. There is now a massive legacy of useful
packages. Systems are now built by linking together many modules
written by many different parties into a single program. Often, this
goes well, creating programs of great functionality by composing
together the functionality expressed by this great number of modules.

However, even though an application can be divided into small units of
independent and reusable code, nothing was done to protect these
modules from each other. Currently modules can interfere with other
modules, either accidentally (bugs), or intentionally (malicious
code). In both cases, such interference limits the scale of systems we
can compose reliably. We must find ways to use modules and compose
them together, so that they can cooperate as we intend when things go
well, but limit the damage when things go badly.
