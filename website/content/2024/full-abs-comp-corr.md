+++
# The title of your blogpost. No sub-titles are allowed, nor are line-breaks.
title = "Full Abstraction Correctness of Compilation"
# Date must be written in YYYY-MM-DD format. This should be updated right before the final PR is made.
date = 2024-10-27

[taxonomies]
# Keep any areas that apply, removing ones that don't. Do not add new areas!
areas = ["Programming Languages"]
# Tags can be set to a collection of a few keywords specific to your blogpost.
# Consider these similar to keywords specified for a research paper.
tags = ["compilers", "full-abstraction", "correctness", "logical-relations", "compatibility"]

[extra]
# For the author field, you can decide to not have a url.
# If so, simply replace the set of author fields with the name string.
# For example:
#   author = "Harry Bovik"
# However, adding a URL is strongly preferred
author = {name = "Matias Scharager", url = "https://pi314mm.com/" }
# The committee specification is simply a list of strings.
# However, you can also make an object with fields like in the author.
committee = [
    "Robert Harper",
    "Jan Hoffmann",
    "Long Pham",
]
+++

Compilation correctness has always been an important concern: even if we write wonderfully pristine code, it means nothing if bugs are inserted during compilation.
With this blog post, we aim not to explain WHY compilation correctness is important as the abundance of compilation correctness literature speaks for itself. Instead, we will discuss WHAT compilation correctness means at a fundamental level, and HOW our particular research approach aims to prove it.

We will develop our discussion by identifying the proper definition of code equivalence, showcasing how these various equivalences fit into compilation correctness, and then building the mechanizations to achieve proof of these equalities.

# Observation Metric

Simply put, compilation correctness means that the source and target program do the same thing by our observation.

The fundamental metric of equivalence comes from the eye of the beholder.
If you keep your eyes closed, any two photos appear the same to you; likewise, we need to observe the side effects of programs to be able to differentiate them.

While we have various options of what specific side effect to observe, the one we use is termination.
A program \\(e\\) terminates (expressed as \\(e\downarrow\\)) if there is a sequence of small operation semantic reductions (\\(\mapsto\\)) that lead to a value:

$$e\mapsto e_1 \mapsto \cdots \mapsto e_n \mapsto v\ \mathbf{val}$$

This chosen model is our "ground truth" of computation and compilation must preserve termination for correctness.

Nontermination is introduced into the language through recursive functions that can call themselves within the body of the function.
In the following rule where a function is called, the whole function is substituted for the variable identifier \\(f\\) within the body of the function \\(e\\), enabling recursion.

$$(\mathtt{fun}\ f(x : \mathtt{A}) : \mathtt{B}\ \mathtt{is}\ e)(v) \mapsto [v/x][\mathtt{fun}\ f(x : \mathtt{A}) : \mathtt{B}\ \mathtt{is}\ e/f]e$$

An example of a nonterminating program would be \\((\mathtt{fun}\ f(x : \mathtt{bool}) : \mathtt{unit}\ \mathtt{is}\ f(x))(\mathtt{true})\\).
Note that using the presented dynamic rule for recursive functions, this program steps to itself in one step, so it never ceases to take the same step over and over again and we end up in an infinite loop.
It is common to annotate this expression as \\((\lambda x. \bot)(\mathtt{true})\\) or even just \\(\bot\\) to indicate that the expression is infinitely looping.

# Kleene Equivalence

We can now define Kleene equivalence (dynamic equivalence) between two programs \\(e_1 \equiv_k e_2\\) as
$$e_1 \downarrow\quad \longleftrightarrow\quad e_2\downarrow.$$

This is the most basic formulation of equivalence, but it already takes us quite far.
The fact that there are no static types involved in this definition is both a blessing and a curse.
The blessing is that \\(e_1\\) and \\(e_2\\) need not belong to the same language, hence we can compare across compilation easily.
The curse is that we get things such as \\(true \equiv_k false\\) because these programs both terminate, but are functionally different in certain settings that can differentiate them.

We can think of Kleene equivalence as compiler correctness under "whole program compilation."
If \\(e_1\\) was the whole program we cared about, and it would never be imported or used within a separate library elsewhere, and \\(e_2\\) was what it compiled to, Kleene equivalence is all we need to describe the correctness of compilation.

We can layer more specifications onto our definition of Kleene equivalence to avoid permitting things like \\(true \equiv_k false\\), and some approaches go this route.
However, in the next section, we find that the compatible closure of our Kleene equivalence elegantly covers these strange cases.

# Contextual Equivalence

With the scale of today's software industry practices, we have an increasing need for compositional approaches in compilation that are in disagreement with Kleene equivalence's "whole program compilation" approach.
For instance, we might want to compile our standard library once, and then link the object files whenever we use a part of the library later in our development.
We might even have several different languages in one repository that are all combined at the assembly level post-compilation.
This calls for a stronger notion of equivalence known as contextual equivalence.

If Kleene equivalence is a "single test case" then contextual equivalence means "all possible test cases."
A program context (test case) is defined as a program with a missing hole in it \\(C[\cdot]\\).
We can fill in this hole with the expression we are observing \\(C[e_1]\\) which can be thought of as testing \\(e_1\\) within the test case \\(C[\cdot]\\).
We thus arrive at contextual equivalence \\(e_1 \equiv e_2\\) being defined as
$$\forall C.\quad C[e_1] \downarrow\quad \longleftrightarrow\quad C[e_2]\downarrow$$
which informally says \\(e_1\\) and \\(e_2\\) are Kleene equivalent under every possible test case.

Even though \\(true \equiv_k false\\) under Kleene equivalence, it is not the case that \\(true \equiv false\\) under contextual equivalence because the context \\(\mathtt{if}\ [\cdot]\ \mathtt{then}\ ()\ \mathtt{else}\ \bot\\) differentiates between them.
One branch terminates and the other branch infinitely loops.

For this definition of contextual equivalence to be valid, we need to make sure that the context \\(C[\cdot]\\) is accepting both \\(e_1\\) and \\(e_2\\) as valid input to the hole, so we need to quantify this via types.
We thus define contextual equivalence at a specific type annotated as
$$e_1 \equiv e_2 : A.$$

We also consider the possibility of \\(e_1\\) and \\(e_2\\) being open terms that depend on variables defined in \\(C[\cdot]\\).
We define \\(\Gamma\\) as a mapping from variables to their appropriate types, hence we now annotate contextual equivalence as

$$\Gamma\vdash e_1 \equiv e_2 : A.$$

Under this definition, contextual equivalence is the coarsest consistent congruence with respect to Kleene equivalence.
That automatically gives us powerful lemmas such as compatiblity and substitutivity that we can use to prove code equivalence.

# Cross Language Equivalence

A drawback to contextual equivalence is that it is no longer clear how to use it across compilation because the source and target programs are from two different languages.
There isn't one result type \\(A\\) and typing context \\(\Gamma\\) to describe both programs if they don't even adhere to the same language.
To resolve this, we find answers within the literature on polymorphic abstraction.

## Representational Independence

In the famous Reynolds 1983 paper on Types, Abstraction, and Parametric Polymorphism we are presented with two conflicting definitions of complex numbers: we either have a pair of width and height, or a radius and angle, otherwise known as cartesian and polar coordinates.
We know the point \\((1,1)\\) is equivalent to the point \\([\sqrt{2}, \frac{\pi}{4}]\\) because they are both interpretations of the same point (share the same object in the model).

To describe an equivalence between these two points, we must describe a pair of functions that lets us coerce from one representation into the other.
We define the functions \\(\mathtt{over}\\) and \\(\mathtt{back}\\) as

$$\mathtt{over}(x,y) := [\sqrt{x^2+y^2}, \tan^{-1}(y/x)]$$

$$\mathtt{back}[r,\theta] := (r\cos(\theta), r\sin(\theta))$$

To be pedantic, we constrain the polar representation so that the radius is non-negative and assume that \\(\tan^{-1}\\) gives us the proper angle in the range of \\(0\\) to \\(2\pi\\) radians.

To show that \\((1,1)\\) is equivalent to the point \\([\sqrt{2}, \frac{\pi}{4}]\\), we simply show either that \\(\mathtt{over}(1,1) = [\sqrt{2}, \frac{\pi}{4}]\\) or that \\((1,1) = \mathtt{back}[\sqrt{2}, \frac{\pi}{4}]\\).
In other words, we need to coerce the coordinates into the same representation, then equivalence is equality under that representation.

Notice that \\(\mathtt{back}(\mathtt{over}(1,1)) = \mathtt{back}[\sqrt{2}, \frac{\pi}{4}] = (1,1)\\) and that \\(\mathtt{over}(\mathtt{back}[\sqrt{2}, \frac{\pi}{4}]) = \mathtt{over}(1,1) = [\sqrt{2}, \frac{\pi}{4}]\\).
More generally, we can prove that \\(\mathtt{over}\\) and \\(\mathtt{back}\\) are inverses of each other for all points.
This is what makes this back-and-forth coercion valid: we can always switch between two representations of the same point.

So what does this have to do with compilation correctness?
We have our underlying model of observation (the small-step relation definition of termination), and the source and target programs have different ways of interpreting this model.
Since they are different interpretations of the same object in the model, there is some logical way (some definition of \\(\mathtt{over}\\) and \\(\mathtt{back}\\)) to coerce one interpretation into the other interpretation.
Compilation correctness is defined as the adherence of the compiler to this logical coherence between the interpretations.

As a side note, \\(\mathtt{over}\\) is not the same as the compiler itself, so this adherence is significant.
\\(\mathtt{over}\\) is directly tied to the underlying model of observation which drives the definition of program equivalence, while the process of compilation rips apart a term and reconstructs a new term in the target language.
In other words, \\(\mathtt{over}\\) acts as merely as an interface to comprehend the source program within the target language without being aware of how the source program works, while the compiler is aware of the construction of the source program and uses it to build a target program in the target language.

## Full Abstraction Correctness

Our overall end goal in compilation correctness is full abstraction correctness.
If we have two contextually equivalent source programs \\(e_1 \equiv e_2\\), and they compile to programs \\(d_1\\) and \\(d_2\\) respectively, then full abstraction correctness definitionally tells us that \\(d_1 \equiv d_2\\).

Representational independence of the source and target languages makes this fairly straightforward to prove: \\(d_1 \equiv \mathtt{over}(e_1) \equiv \mathtt{over}(e_2) \equiv d_2\\) (since contextual equivalence is closed under symmetry, transitivity and compatibility).

With this, we finalize our definition of WHAT it means for a compiler to be correct, what is left to do is show HOW we proved this representational independence for compilation.

# Type-Directed Language Merging

The biggest distinction in compilation correctness approaches comes from HOW we define the cross language \\(\mathtt{over}\\) and \\(\mathtt{back}\\) relations as it decides the model we base our correctness on.

One option is to keep the source and target languages separate and define some non-symmetric version of contextual equivalence.
The drawback to this is that the model for which we are basing our \\(\mathtt{over}\\) and \\(\mathtt{back}\\) relations lie outside the framework provided by the small step semantics, we just need to accept that the non-symmetric version of contextual equivalence we define depicts some reasonable model, but this definition could be rather messy depending on how complicated our compilation is.
In particular, it might make it harder to prove full abstraction correctness as our earlier proof sketch no longer works without symmetry.

Instead of this approach, we can utilize the small-step semantics model of our languages as much as possible by merging the source and target language into one giant joint language that preserves the small-step semantics of both languages.
This is the approach taken by Perconti and Ahmed where the joint language consists of all the syntactic constructs of both languages and stapling constructors \\(^\tau \mathcal{ST}(e)\\) and \\(\mathcal{TS}^\tau(e)\\) bridging the two languages called the boundary terms.

In this setting, we ground our model of correctness in the small-step semantics of the joint language, but we need to add small-step semantics for reasoning about the boundary terms.
To do this, we define the \\(\mathtt{over}\_\tau\\) and \\(\mathtt{back}\_\tau\\) relations on values and add the following rules into the small step semantics of the language.

$$\frac{\mathtt{over}\_\tau(v) = v'}{^\tau \mathcal{ST}(v) \mapsto v'} \qquad \frac{\mathtt{back}\_\tau(v') = v}{\mathcal{TS}^\tau(v') \mapsto v}$$

These rules demonstrate how closely associated the \\(\mathtt{over}\\) and \\(\mathtt{back}\\) relations are to the model of the language because the model itself is defined via the relations.
The drawback is that these added rules to the small step semantics overcomplicate the model and add significant effort in proving the type-safety of the joint language to demonstrate the model still makes sense.

What if we utilize the small step semantics we have to define \\(\mathtt{over}\_\tau(v) = v'\\) as \\(\mathtt{over}\_\tau(v) \mapsto \cdots \mapsto v'\\) (without observable side effects in the \\(\mapsto \cdots \mapsto\\))?
Our two rules would now look like

$$\frac{\mathtt{over}\_\tau(v) \mapsto \cdots \mapsto v'}{^\tau \mathcal{ST}(v) \mapsto v'} \qquad \frac{\mathtt{back}\_\tau(v') \mapsto \cdots \mapsto v}{\mathcal{TS}^\tau(v') \mapsto v}$$

In this setting, we don't need these two rules anymore!
We can discard the boundary terms altogether because our base model of small-step semantics of the joint language is capable of representing the nature of the \\(\mathtt{over}\\) and \\(\mathtt{back}\\) relations innately.
This means we keep our model of the joint language very simplistic, matching the standard practice of defining small-step semantics, then define the \\(\mathtt{over}\\) and \\(\mathtt{back}\\) relations as functions within the joint language.

It might seem like a small change at first, but the significance of having a simpler model is not to be overlooked.
For one, having a simpler model makes it more evident that this is a valid proof of compilation correctness as less complicated assumptions are being made in the definition of our model.
The more practical result is the reusability of our model for various separate compilation phases stacked one after another vertically where before we would have needed to define a new boundary term model for each new compilation phase.
We can now maintain the same joint language model, prove each phase separately, and then merge our proofs through the transitivity property granted by sharing the same model.

We call this strategy a type-directed joint language because the source and target languages are separated by their types rather than synaptically via the boundary terms.
This is particularly nice for when there is significant overlap in the source and target languages as is the case for several intermediate compilation procedures because the model of the joint language becomes closely related to the models of the intermediate languages with few extra rules.
In our case, the joint language is nearly identical to the source language itself, and compilation steps for the most part refine the language's logic into more rudimentary machine-like languages as is the norm in compilation, so this type-directed merging strategy for the joint language is greatly beneficial.

To summarize our new compilation correctness approach, we first define our source and target languages, then take all their components and use them to create a joint language.
We verify that each one of these languages is a valid model by proving the type-safety properties of the structural operational semantics.
We then define a compilation procedure between the source and target language, and \\(\mathtt{over}\\) and \\(\mathtt{back}\\) functions within the language that describe the compilation procedure within our model of small-step semantics.
We achieve compilation correctness by demonstrating that our compilation procedure respects (through contextual equivalence) the \\(\mathtt{over}\\) and \\(\mathtt{back}\\) functions and hence preserves the properties of small step semantics through compilation.

# Logical Equivalence

Contextual equivalence by itself is difficult to work with.
How would one even approach thinking about "every possible test case" and develop a proof about it?

The answer is that every test case can only use the expression in a specific way expressed by the interface that the type provides.
For example, if our program is a function \\(A\to B\\), then the only way a test case can interact with this function is by calling the function with values of type \\(A\\).

Following this same trend of figuring out "how can my program be used" we arrive at an inductive definition known as the _logical relation_ which gives us a new form of equivalence known as _logical equivalence_.
Logical equivalence provides us a framework to reason about contextual equivalence by utilizing the properties of the type structure.

We will not dive deeply into logical relations as they are a well-known result.
However, we can claim a contribution to this field as the first to formally verify the correspondence between logical equivalence and contextual equivalence for a language with explicit control flow, polymorphism, and nontermination within the Coq proof assistant.
Control flow effects add an extra layer of complexity in the realm of formal verification that past literature has not deeply explored.
It is now good to know that past theoretical results are backed by the trusted core of the Coq proof assistant.

# Proof of Concept

To demonstrate that our compilation correctness procedure works, we proved the correctness of two separate compilation phases and then joined these two proofs together to achieve full abstraction correctness.

## Continuation Passing Style Phase

The continuation passing style (CPS) translation is commonly used to cleanly express the control flow of programs.
We convert a term \\(e\\) into a pair \\((k,d)\\) where \\(d\\) is a computation, and \\(k\\) is a continuation for which the computation returns a value once it is finished running.
Normally, these are a one-to-one correspondence: \\(d\\) only produces one value which gets sent to the continuation \\(k\\) at one location, but this translation phase becomes most interesting when we add control flow effects into our language which breaks this one-to-one correspondence.
The CPS translation can tame all these control flow effects, leaving us in a simpler language without explicit control flow operations.

Our source language contains the explicit control flow operations of \\(\mathbf{letcc}\\) and \\(\mathbf{throw}\\).
\\(\mathbf{letcc}\\) allows us to record the current state of the computer and save it as a continuation while \\(\mathbf{throw}\\) lets us jump to some previous state of the computer by calling the respective continuation.
You can pretend it's similar to an assembly jump operation, so you can imagine how it could be used to express conditional branching and short-circuiting of computations.

Interestingly, \\(\mathbf{over}\\) and \\(\mathbf{back}\\) for the CPS translation must use these control flow mechanisms within its definition.
You may have missed the small detail earlier that we require no "observable side effects in the \\(\mapsto \cdots \mapsto\\)" within the definition of \\(\mathbf{over}\\) and \\(\mathbf{back}\\) yet here we are including side effects into that definition.
Graciously, we can prove that these particular side effects are benign meaning they won't influence our observation of the program, so we can maintain the properties we want with some extra effort.

## Closure Conversion Phase

A closure is an inline function that can utilize the initialized variables in the context within its definition.
Closure conversion (CC) translation takes these closures and turns them into global function definitions that can be moved into the top level instead of inlined.
It achieves this by packaging the variables in the context that the closure depends on into a large tuple and passing them as input into the global function to maintain the dependency.

Central to the proof of this phase is the representational independence relation between the original closure function and the resulting global function.
Function application becomes a parametric packing operation into an existential type before calling the function.
This allows us to utilize the full power of Reynold's parametricity in our logical relation to push through the proof of compiler correctness.

## Joining it All Together

Our final theorem for full abstraction correctness states that if we have \\(e_1 \rightsquigarrow_{CPS} (k, d_1) \rightsquigarrow_{CC} (k, t_1)\\) and similarly \\(e_2 \rightsquigarrow_{CPS} (k, d_2) \rightsquigarrow_{CC} (k, t_2)\\), and we know that \\(\Gamma \vdash e_1 \equiv e_2 : \tau\\), then \\(\overline{\Gamma}, k : \overline{\tau}\to0\vdash t_1\equiv t_2 : 0\\).
This essentially says that the equivalent \\(e_1\\) and \\(e_2\\) programs compile into equivalent programs, even with both phases of compilation put together.

We successfully demonstrate that this correctness approach works for these two phases of compilation by rigorously proving this theorem.
In the future, we hope to include more phases of compilation into this framework as we build a large, cohesive joint language model that can represent various compilation passes and connect them.