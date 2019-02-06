
#### Typeclasses

Haskell typeclasses are one of the most important and complex features of its type system, that distinguishes Haskell among other well-known functional languages.
Implementing typeclasees using Code Rules shows that it has expressive power that is sufficient even for advanced type systems.

Note about notions: by the word "constraints" are meant Code Rules constraints, whereas typeclass constraints are called in this qualified way ("typeclass constraints"), or with an uppercase letter ("Constraints") to avoid ambiguity.


##### Typeclass Constraints

In essence, typeclasses extend type system by adding typeclass constraints on type variables bound by universal quantifier.
<!-- And everything that follows from that. -->
Everything else is mostly a matter of other language aspects (structure, editor and constraints aspects) and not of a type system.
Still, it's important to note, that type system doesn't need to check that type variables are properly scoped, because it is handled by "constraints" language aspect.
It simplifies the type system.
    <!-- MAYBE MOVE IN PREVIOUS SECTION ABOUT STLC? -->

So, existing type system needs to be modified only in the places concerned with universal types:
- modification in `forall` handler;
- additions in macro table `types` and `recover` handler.

Other parts of typechecking typeclasses are placed in their own handlers:
- handling declarations of typeclasses and its instances --- `typeclasses` handler;
- processing typeclass constraints (collecting and checking them) --- `typeConstraints` handler;
- auxiliary Set implementation used mainly in `typeConstraints` handler --- `set` handler.

Now, let's move through all these places.


##### Set Data Structure

Handler `set` implements a Set data structure together with several utility functions.
It is represented as a number of `set` constraints in the constraint store.
Each `set` constraint has 2 arguments: the first argument is a free logical variable representing handle for the data structure and the second argument is an element belonging to it.
As an example, a set `{ a, b, d }` (where a, b and d are some dataforms or free logical variables) will be represented as three constraints: `set(S1, a)`, `set(S1, b)` and `set(S1, d)`, where `S1` is a Set handle.
Programmer can manipulate the set with a handle.
The implementation is quite straightforward.
There's a single rule that mantains the set invariant, i.e. that it can't contain two equal elements.
<!-- Notice that the `equals` predicate is used here. -->

![](img/set_removeDupl.png)  
_(set data structure implementation)_

The beauty of such representation of a set is that merging sets comes essentially for free and, moreover, is performed implicitly.
To merge two sets a programmer only needs to unify the Set handles.
And there one crucial feature of Code Rules enters the picture: constraint reactivation.

Consider the following example.
There're two sets `S1 = {a, b}` and `S2 = {a, c}` and the constraint store contains four constraints representing them: `set(S1, a)`, `set(S1, b)`, `set(S2, a)` and `set(S2, c)`.
Next, unification of logical variables `S1` and `S2` happens (`S1 = S2`).
Due to automatic reactivation, all inactive constraints in the store that refer to these unified variables will be activated again.
Because of this they will be tried for matching on the rule maintaining set invariant.
In this particular case it will be triggered on the constraints `set(S1, a)` and `set(S2, a)` (remember, here `S1=S2`) and discard one of them.
After this constraint store will contain only three `set` constraints: `set(S1, a)`, `set(S1, b)` and `set(S1, c)`.
The implementation of a set is also an example of one useful Code Rules pattern: maintaining some invariant assumed by other rules using a top-level rule, that will be matched first and ensure invariant.

Handler `set` also declares several utility functions with straightforward implementations:
- copying a set;
- making a Cons-list out of a set with removing the original set;
- making a Cons-list out of a set while preserving the original set;
- testing whether one set is a subset of another.


##### Representation of Typeclass Constraints

Typeclass constraints are used in 2 roles during type checking: in definitions of type variables bound by quantifiers and as requirements on types, signifying that these types must satisfy them.
Handler `typeConstraints` declares 2 constraints that correspond to these 2 roles: `tdefConstraints` and `typeConstraints`.
They map a type variable to a set of its typeclass constraints.
Contrast it with how type variables are represented: they're explicitly stored in a list in the `Forall` dataform.
So, typeclass constraints on a type variable are stored implicitly, not inside the `Forall` dataform, but in the constraint store in the form of a constraint. 
Their representation as a set greatly simplifies processing them.

During the process of type inference free logical variables representing yet unknown types can be unified.
Previously we haven't given much thought to it, unification just happened without our attention.
<!-- But what happens now, when they can carry sets of typeclass constraints? -->
But now consider this question: what changes when free type variables can carry sets of typeclass constraints?
The answer is that we should merge these sets when variables are unified.
But at which point? The problem is that unification can happen at any point (in any rule) during type inference.
And that's where constraint reactivation again helps us, now for `typeConstraints` and `tdefConstraints`.
The following rule is a top rule in handler `typeConstraints` to ensure, that each type variable has a single set of typeclass constraints, a single source of truth.

![](img/typeConstraints_mergeSets(v2).png)  
_(merging Constraints sets of unified type variables)_


<!-- ##### Rules -->

It is clear how to collect typeclass constraints during the process of type inference.
But what we further need to do with this set of typeclass constraints depends on what will happen to the type variable.
There're two cases: it either remains free until the point of generalization (i.e. let-binding) or it is bound to a type.


##### Checking Typeclass Constraints

When free type variable becomes bound to something, we need to check that typeclass constraints collected up to this point are satisfied.
Due to constraint reactivation, another rule that starts the check will be triggered.

![](img/typeConstraints_discharge.png)  
_(rule playing a role of entry point to typeclass constraints check)_

There're 2 cases: it can be bound to a ground type (e.g. function type, Bool) or a type variable reference,
They differ and require different checks.

![](img/checkVarConstraints.png)  
_(check collected typeclass constraints for a type variable: 2 cases)_

####### Strength Check

First, let's consider a slightly less obvious case of a type variable reference.
It refers to a type variable bound by some universal quantifier, and it has its own set of typeclass constraints, as part of its definition.
This set must be more restrictive than the set of collected typeclass constraints we need to satisfy.
In other words, the type variable can't be instantiated to a type (here, type variable reference) that doesn't satisfy its typeclass constraints.
To check it we need to ensure that for each Constraint from the set of collected Constraints there's a matching Constraint to satisfy it.
It can be done with a `isSubset` check.

![](img/strengthCheck.png)  
_(strength check: checking the restrictiveness of Constraints sets)_

####### Instance Check

The second case of a usual type is more intuitive and much more common, but its implementation is a bit more involved.
The first two rules in it do nothing special.
The first makes a list out of the set and the second traverses it, triggering actual instance check for each typeclass constraint on a type.

![](img/instanceCheck(1-2).png)  
_(instance check, first two rules)_

The next two rules try to find a matching instance among all instances of the typeclass with a help of `instance` constraint.
It is present in the constraint store for each declared typeclass instance in a program.
Typeclass instances can be declared not only for types but also for type schemes.
For example, Haskell has such instance of a `Monoid` typeclass: `instance Monoid [a]` where instance is declared for a universal type `forall a. [a]`. Compare it to `instance Monoid Bool` where we have a usual type `Bool`.
That's why we need to try instantiating the type scheme from instance declaration to the type in question.
It is done with a combination of `inst` constraint and unification.
It's important to note, that instantiation here may trigger recursive `instanceCheck`.
For example, consider how `instanceCheck( Pair(Bool, Bool), {Constraint(Monoid)} )` proceeds in presence of `instance Monoid Bool where ...` and `instance (Monoid a, Monoid b) => Monoid (a, b) where ...` in a program.
The last rule simply fails in case when no matching instance is found.
The typecheck is failed by triggering `eval(false)`.
Of course, it isn't the best mechanism for reporting type system errors, but it's a temporary solution.


<!-- ![](img/instanceCheck.png)   -->
![](img/instanceCheck(3-5).png)  
_(instance check: searching for a typeclass instance for a type)_


##### Universal Types and Typeclass Constraints

Typeclasses require only a little change to `forall` handler.
In generalization of a type variable we have to add only a single production that moves collected typeclass constraints (in `typeConstraints`) to a definition (`tdefConstraints`).

![genTypeVars rule](img/genTypeVars.png)  
_(generalization of a single type variable: notice activation of `produceTypeConstraints`)_
<!-- _(notice `produceTypeConstraints` production)_ -->

![](img/produceTypeConstraints.png)  
_(make collected typeclass constraints a part of the type variable definition)_

This constraint `tdefConstraints` is used in an instantiation of a type variable.
We essentially do the opposite: we copy typeclass constraints from definiton (`tdefConstraints`) to a new set (in `typeConstraints`) for a freshly instantiated type variable.

![instTypeVars rule](img/instTypeVars.png)  
_(instantiating single type variable: notice activation of `typeConstraints` with a copied set)_
<!-- _(notice activation of `typeConstraints` with a copied set)_ -->

Consider this example.
    **TODO**
    <!-- ...generalization -->
    <!--     full example: say, mappend? -->
    <!-- ...instantiation -->
    <!--     example: function (prototype) application -->


##### Typeclass and Instance Declarations

Handler `typeclasses` extends `annotation` handler and declares no new constraints.
It ensures the well-formedness of typeclass and instance declarations.

...
...

![](img/getType_Typeclass.png)  
_()_

The rule for instance declaration expands a macro for its type scheme and produces `instance` constraint that is used in `instanceCheck`.

![](img/getType_Instance.png)  
_()_

Typeclass contains a number of function Prototype declarations (i.e. function variable name and type).
The rule for them just produces their type given type annotation.

![](img/typeOf_Prototype.png)  
_()_

Instance, correspondingly, contains Prototype implementations.
The most involved rule is for checking them.
...
...
...

![](img/typeOf_PrototypeImpl.png)  
_()_

TODO
    `types` --- trivial
    `recover` --- nothing special, has already been described.
        but SHOW recovering Constraints?


<!-- ![](img/)   -->
<!-- _()_ -->

<!-- .to mention -->
<!--     aux constraint for 'recover' to avoid unnecessary dependencies -->
