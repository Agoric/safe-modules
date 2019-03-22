## Problems with Current JavaScript Modules

Even starting from a SES environment, if we naively incorporated
existing JavaScript module systems as is, we would have the following
vulnerabilities:

-   Imports are authority bearing. A module not only gains read or
    write access to the global scope where it's imported, it gains the
    same ambient authority as the main program, meaning that it gains
    access to all available APIs with the same execution rights.

-   The legacy linking system via global declarations and lookups is
    available and still sometimes used. There is no guarantee that all
    dependencies between modules are expressed by imports and exports
    only.

-   A module isn't limited to declarations. Loading a module executes
    code that initializes the module instance, creating stateful and
    powerful instances that are shared by universal access to one
    shared import namespace.

-   Dynamic imports are not statically analyzable. Both the CJS
    `require()` function and the ESM `import()` expression accept
    computed strings.

-   Through the one shared import namespace, any module can load any
    other module, preventing any encapsulation of internal modules
    within larger subsystems.

-   A modules can contain top-level mutable state, allowing unintended and
    unauthorized communications channels. For example, modules can
    export:
    -   An assignable variable.
    -   A mutable object.
    -   A function which closes over an assignable variable.
    -   An object which can hold state such as a Map.
