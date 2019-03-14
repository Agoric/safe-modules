## Read the Safe Modules Doc

[Read the document on Safe Modules](https://agoric.com/safe-modules/), originally written by Mark S. Miller, Darya Melicher, Kate Sills, and JF Paradis in December 2018.

The [Jetpack documentation](https://agoric.com/safe-modules/jetpack) by Brian Warner may also be of interest. 

## Examples

We start with a simple command line app that records and displays todos. The app uses the popular Chalk and Minimist packages. Chalk uses the built-in `os` module and the `process` global variable. The various examples show different ways to describe and attenuate that authority. 

[Clean Todo](https://github.com/Agoric/safe-modules/tree/master/examples/clean-todo)

Clean Todo rewrites the packages to be ["pure"]() and has no manifest. 

[Legacy Todo](https://github.com/Agoric/safe-modules/tree/master/examples/legacy-todo)

Legacy Todo doesn't rewrite the packages, and instead uses a manifest to provide the configuration of authority for a loader. 

[Clean Index](https://github.com/Agoric/safe-modules/tree/master/examples/clean-index)

Clean Index expresses the top-level wiring with code consisting only of scoping and invocations to do essentially the same top level modules, given that these modules have been (re)written as pure modules. The code form is much clearer. For code known to be written in this minimal wiring subset of Jessie, there's nothing less declarative about it. The JSON form doesn't do the job it describes. It is in essentially a new language "manifest" language whose meaning one has to learn, and requiring new code to implement. The code form does the job it describes. It can be understood without learning new concepts.
