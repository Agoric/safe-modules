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

