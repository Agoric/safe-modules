## Appendix B: Rules for a Purity Checker

This purity checker would verify modules as pure, conditional on the
modules it imports also being pure.

We aren't trying to write the purity checker such that it transitively
applies to the modules it imports because that makes things like
cycles hard to check. Rather, if a given loader only imports
conditionally-pure modules, i.e., modules that would be pure if their
imports are pure, then a self contained graph of such conditionally
pure modules should be pure. Thus, a conditionally-pure module is pure
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

Both `makePoint` and `makeCounter` are purifiable - if hardened they
are pure. The record returned by `makePoint` is purifiable if and only
if `x` and `y` are purifiable. The harden function will reach both `x`
and `y` and attempt to harden them as well. If harden returns
successfully, then the record, the value of `x`, and the value of `y`
are all hardened. If harden throws, then we do not assume any of these
are hardened.

If you harden the counter returned by `makeCounter`, harden will be
applied to the `countUp` method. But the `countUp` method captures a
lexical `count` variable that is assigned to. Therefore the `countUp`
method is not purifiable, and therefore the object containing it is
not purifiable either.

A purifiable expression is one that we know evaluates only to
purifiable values.

Now let's take this example:

```js
function makePoint2(x, y) {
 return {x: +x, y: +y}
}
```

Unary `+` coerces its operand to a number, se we know that the `x` and
`y` properties are numbers, and the object that is returned is
therefore purifiable.

Our starting rule for determining that a module is pure is that it is
only exporting purifiable values, that is, values that become pure
when hardened.

In order to know that we need to take a look at the expression
evaluating to those values.

Most of our observations apply equally to CJS modules and ESM modules,
where *exports* means the values exported by a module either by static
`export` declarations or assignments to the `exports` variable, and
*imports* means the values obtained either by static `import`
declarations or calls to `require`. CJS and MSM modules also have the
following problematic edge cases.

   * Our rules are unsound for import cycles. For both CJS and ESM,
     import cycles can cause the observation of uninitialized or
     partially initialized state. Checking modules in isolation means
     checking them in ignorance of such cycles.

   * In a CJS module, we effectively treat `require` as a keyword.  We
     allow a CJS module to use `require` only by calling it. We
     conservatively reject any use of `require` in any other way.

   * In a CJS module, we effectively treat `exports` as a keyword.  We
     allow a CJS module to use `exports` in only one of two ways:
     either by a top level statement like `exports = expr;`, exporting
     only the value of `expr`, or by top level statements like
     `exports.foo = expr;`, which we treat as separate named
     exports. We conservatively reject modules that use both of these
     styles, or that use `exports` in any other way.

   * We reject any ESM module with a top-level `await`.

   * We reject any ESM module that exports a live binding, i.e., that
     contains an assignment to an exported variable.

We've now reduced the problem to the individual kinds of expression.
The exports of a module result only from the module's initialize-time
computation, producing the module's *top level state*. Modules mostly
do declarations at initialization time, postponing most interesting
computation till later. We start with the expressions most relevant to
analyzing top level declarations:

   * A ***constant***, like `3`, that has a primitive value. It is
    obviously purifiable because it's already pure.

   * A ***variable name***, like `x`, is purifiable if
       * is the name of a whitelisted global like `Object` which we
         assume to be pure.
       * `x` is bound by an `import` declaration, which we assume to
         be pure.
       * `x` is initialized to the value of a purifiable expression,
         either by a `const` or `let` declaration.  (We reject `var`
         for now.)
       * `x` is initialized to a purifiable function or class, by a
          named `function` or `class` declaration or expression.
       * `x` is not assigned to, making its declaration effectively a
         `const` declaration.
       * The variable `x` is not used in any way that might mutate its
         value before it is hardened.
       * The value of `x` is not used in any way that might mutate it
         before it is hardened. We mention this separarely because it
         requires alias analysis.

   * A ***regexp literal***, like `/.*/` is unconditionally purifiable.

   * An ***array literal***, like `[x,y,...rest]` is purifiable if 
       * the element expressions `x` and `y` are purifiable.
       * The value of the `rest` expression is unpacked into the
         array. Normally, if `rest` is purifiable, then it will unpack
         into purifiable array elements. But if `rest` might define a
         programmatic iterable, it could violate this assumption. Such
         an analysis is well beyond the ambition of this document, so
         we conservatively reject an array containing `...rest` as not
         purifiable.

   * An ***object literal***, like
   
     ```js
     ({__proto__: p,
       x: x1
       y,
       m() { return q; },
       get a() { return q; },
       set a(newa) { q = newa; },
       [e1]: e2,
       ...rest,
     })
     ```
   
     is purifiable if it inherits from a purifiable object and all of
     its properties are initialized to purifiable values.
        * `__proto__: p` looks like a property definition, but it is
          not. It is specially recognized syntax saying that the new
          object inherits from the value of `p`. It is purifiable if
          the `p` expression is purifiable. If omitted, the new object
          inherits from `Object.prototype` which we assume to be pure.
        * `x: x1` is a normal data property definition. It is
          pureifiable if the `x1` expression is purifiable.
        * `y` is equivalent to `y: y`. It is purifiable if `y` is
          purifiable.
        * `m() { return q; }` is concise method syntax that, for our
          purposes, is equivalent enough to

          ```js
          m: function() { return q; },
          ```
          
          This property is purifiable if the function expression is
          purifiable. The function expression must be anonymous
          because no variable `m` is in scope in the function's
          body. The differences are 
             * The new function's `name` property would actually be
               set to `m`. This does not affect purifiability.
             * The new function does not have construct
               behavior. Calling `new` on it would cause an
               error. This does not affect purifiability.
             * The new function has no `prototype` property.
             
        * `get a()..., set a(newa)...` defines `a` as an accessor
          property. The *value* of the property is defined
          operationally by what the getter returns. But for purposes
          of purifiability, we don't care about the value of an
          accessor property, we care about its getter and setter
          functions. The property is purifiable if both of these
          functions are purifiable.
          
        * `[e1]: e2` evaluates `e1` to determine the name of the
          property to initialize. It initializes it to the value of
          `e2`. This property is purifiable if the `e2` expression is
          purifiable. We don't care about the expression `e1` because
          its value will be coerced to either a string or symbol, both
          of which are pure.
          
        * `...rest` unpacks the enumerable own properties of `rest` to
          define additional properties of the new object, initialized
          to the *value* of these properties on `rest`. As with
          `...rest` in arrays, normally, if `rest` is purifiable, then
          it will unpack into purifiable array elements. However, as
          with arrays, this can also trigger programmatic behavior --- if
          `rest` has accessor properties or is a proxy. So we also
          reject `...rest` in an object literal as not purifiable.
   
   * A ***function*** is purifiable if every variable that it captures
     is not assigned to and each of those variables are initialized to
     pure values. For example:

     ```js
     let x = {};                 // x is purifiable
     function f() { return x };  // f is not purifiable
     ```

     Even though the empty object `x` is purifiable, `f` is not
     purifiable in this example. The reason is that hardening `f` does
     not harden `x`. Therefore `f` can be used as a communications
     channel. For example, Alice and Bob could both have a hardened
     `f` and then they both call it.  They get the same mutable
     object, and can use it communicate with each other. By contrast:

     ```js
     let x = harden({});         // x is pure
     function f() { return x };  // f is purifiable
     ```

     The function's `prototype` property has to be a purifiable
     value. If the function's `prototype` is never mentioned in the
     module, it is still purifiable, because the `prototype` is
     implicitly initialized to an empty object that inherits from
     `Object.prototype`. Hardening such a function will harden its
     unmentioned prototype.

   * A ***class*** is purifiable if it contains no elements that are
     beyond the ES2017 standard. Elements from proposals that are
     still in flux at the time of this writing (such as decorators)
     should fail the purity checker. Purifiable if all of its static
     properties are purifiable, if all of its methods are purifiable,
     and if it extends (i.e. inherits from) a pure class (not just a
     purifiable class).

More interesting top level computation is comparatively rare, but
still common enough for us to accommodate it when we can.

   * A ***function call*** to a named function, like `makePoint(3,5)`.
     To determine whether it is purifyable, we look at the function
     definition.
       * The function must be named and defined in the same module.
         Otherwise the function call is rejected.
       * We analyze the function's body by inlining it in the context
         of that function call, treating each parameter as a `let`
         declaration initializing that parameter to that argument.
         While being careful not to confuse scopes, within the body of
         the function we analyze uses of the parameter variables by
         our variable rules above.

     Thus, a function call to `makePoint` with purifiable arguments is
     itself a purifiable expression. Any other call to `makePoint` is
     conservatively rejected.

   * A ***method call***, like `bob.foo(carol)`. In the rare cases
     when we can reliably determine what function is looked up by
     `bob.foo`, we treat the method call as a function call with an
     implicit `this` argument. For non-arrow functions, we treat each
     occurence of `this` as if it is a use of a parameter variable
     initialized to that argument. For arrow functions, `this` is like
     a lexical variable, defined by the implicit `this` parameter of
     the closest enclosing non-arrow function.

   * A ***`new` construction***, like `new Point(3,5)`, is treated
     like a function call. The implicit arguments include the function
     itself, what we are doing the new on. That has to be pure, not
     just purifiable, and all the explicit arguments need to be
     pure. And the function itself needs to be visible in the same way
     as a normal call (within the same unit of analysis, the
     module). Thus, the same analysis that we do with functions is
     safe to apply to the new case. You can also do a new on a class,
     which makes an instance of the class. We consider all instances
     to be not purifiable for now, which can be revisited later if
     necessary.

Notes

   * If the value of a purifiable expression is subsequently mutated,
    like by property assignment, before they are hardened, we have to
    assume that they aren't purifiable.

   * We would need to do a deeper analysis - take a look at everywhere
    that the object may have gotten to, and we have to check two
    things -

   * The purifiable value doesn't escape from the module before it is
    exported. For example, you could import something, and pass the
    object to it. Without violating purity, the receiver could mutate
    this argument before it is hardened.

   * We need to make sure that there is no code within this module that
    directly mutates it, such as by property assignment.

For example, let's say we have a function `foo`, and somewhere in the
module an expression like `foo.x = <something not purifiable>`, then we
have to say that foo is not purifiable.

We will go even more conservative - If `foo.x = <anything>`, then we say
`foo` is not purifiable.

   * Property lookup `foo.x` - not a purifiable expression. Same issue as
    with method calls. We may try to make this purifiable later, but
    we'd have to be very cautious.

   * If you just do `foo.x` in the module, does that make `foo` itself no
    longer purifiable? I think the answer is yes. Just property lookup
    can cause side effects, and it would take a deep analysis to be
    able guarantee that it isn't causing side effects.

   * Implicit coercions - if you have an expression like x + y, that
    expression might cause the toString() or valueOf() method of x or
    y to be invoked. Those could cause side effects and that could be
    before the purifiable things are hardened. In order for an object
    literal to be considered purifiable, we will start off saying that
    it doesn't have a toString or valueOf method. Later we can relax
    it, but for now we need to say those make it not purifiable.

   * Weird new things to reject because too complicated - dynamic
    import expression, import.meta expression. If we see either of
    those in the module, we reject the whole module. Same thing with
    direct eval syntax - reject the whole module.

   * Rejected if there are var declarations, we are only processing
    let, const, function and class.

   * Temporal dead zone - In JavaScript a variable can be accessed
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
