# Safe JavaScript Modules

## Motivation

If you do an `npm install`, the package that is installed can do tremendous damage. It can read your files, write to your files, and send your files and data over the network, among other things.

One proposed solution is to stop using other people's code. However, we think this is a non-starter. Code reuse creates a rich programming environment where we can build on each other's work. Having to write everything ourselves is a significant drag on creative activity. 

The solution is to allow the use of third-party code, but to prevent that code from gaining access to authority (like file system or network access) unless explicitly given. Moreover, instead of allocating authority in large chunks ("here's the entire file system") we can *attenuate* authority in fine-grained pieces such that only what is absolutely necessary is given. For instance, maybe append-only access to a particular file is all that is necessary.

## Methods

[SES](https://github.com/Agoric/SES) building on [Realms](https://github.com/tc39/proposal-realms) provides this safe execution environment. 

## Read the Safe Modules Doc

[Read the document on Safe Modules](https://agoric.com/safe-modules/), originally written by Mark S. Miller, Darya Melicher, Kate Sills, and JF Paradis in December 2018. (The raw markdown is located in `/docs`)

## View the examples

Sample code is located in `/examples`.
