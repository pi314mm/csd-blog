+++
# The title of your blogpost No sub-titles are allowed, nor are line-breaks.
title = "Correctness of Compilation"
# Date must be written in YYYY-MM-DD format This should be updated right before the final PR is made.
date = 2025-03-31

[taxonomies]
# Keep any areas that apply, removing ones that don't Do not add new areas!
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
    
]
+++

Compilation is the process of converting a human-comprehensible, mathematically rich programming language into a machine-readable, operationally explicit target language, often involving multiple sequential phases.
Compiler correctness ensures that the original program supplied by the user and the target program produced as output "do the same thing" according to our observations.

Compiler correctness has always been an important concern: even if we write wonderfully pristine code, it means nothing if bugs are injected during compilation.
This blog post will explore the complexity of verifying compiler correctness, delve into its mathematical definition, and showcase the specific approach we adopted in our research as a proof of concept.

# Difficulties in Proving Compiler Correctness

"You can't compare apples to oranges" aptly summarizes one of the main sources of complexity in compilation proofs.
The source and target languages involved in compilation are different languages that may have completely different memory models or operational behaviors.
We (usually) cannot prove that the source program and compiled program have the "same" behavior; instead, we must prove they have "related" behavior according to a clearly defined notion of what "related" means.

We can use the analogy of translating between two different spoken languages, say English and Spanish, as many of the same problems occur in computer program compilation.
First, a translator would need to understand both English and Spanish to understand and translate the meaning.
Sometimes, the same exact meaning can be conveyed; there are even a lot of words in Spanish directly acquired from English.
However, sometimes the meaning gets morphed in translation, and information could get lost: a pun in English could be translated directly to Spanish, so the direct translation would make practical sense, but the humor of the pun would be lost in translation.

That's a lot of information, so let's unpack it sentence by sentence.

- A translator would need to understand both English and Spanish

Having grown up learning both English and Spanish, I often end up speaking *Spanglish*, a language that can freely blend English and Spanish language and grammar.
It is not a matter of switching back and forth between these two languages, but maintaining a mental understanding that comprehends both at the same time.

With our research approach, we create our own version of *Spanglish* but for the source and target computer languages.
Once we have this joint language, the metalogic used to compare the source and target programs is defined through equivalence relations in the joint language, since both source and target programs are expressible in the joint language.

- A lot of words in Spanish directly acquired from English

Program compilation procedures often involve multiple sequential steps, each dealing with a very specific task.
Consequently, many intermediate languages have very similar or possibly identical semantics, with perhaps only one small change while keeping the rest the same.
Our correctness-proof strategy leverages this advantage, handling the easy cases efficiently so we can focus on the difficult ones, showcasing a major benefit of our approach.
However, if the source and target languages are substantially different, our proof strategy would still work in theory but would become much messier and less practical compared to other approaches.

- A pun in English could be translated directly to Spanish, so the direct translation would make practical sense, but the humor of the pun would be lost in translation

This analogy helps us realize that there can be several interpretations of "compiler correctness," with some guarantees being stronger than others.
While it is reasonable to consider the direct translation from English to Spanish correct because the informational content remains the same, one could also argue that it is incomplete because it fails to convey important contextual information, such as a pun in the original text.

This relates to the difference between *operational correctness* and *full abstraction correctness* in compilation.
Operational correctness states that source and target programs behave equivalently when viewed as a whole-program translation, though this equivalence is context-sensitive.
Full abstraction correctness extends this by requiring that the translation must also be context-insensitive.

Think of it this way: if we know beforehand that each sentence contains a pun (a pre-defined context), then the direct translation from English to Spanish suffices.
However, if we insert this direct translation into another paragraph where readers don't know that every sentence should contain a pun (a changed context), we've created a translation error because we've lost the pun's content.
A stronger translation would incorporate a Spanish pun that conveys the same concept, allowing readers to understand the intended message regardless of contextual information.

Given the scale of modern programming practices, we often require full abstraction correctness in addition to operational correctness to ensure our compiler is compositionally correct.
Without this stronger guarantee, bugs could be introduced post-compilation when importing a pre-compiled library in an incompatible context, or when linking together programs from various sources and compilers that may have conflicting contextual assumptions.

Our research demonstrates a new proof technique that can be used to verify operational correctness and full abstraction correctness via type-directed joint language merging.
We have shown the efficacy of this approach by proving the correctness for two well-known phases of compilation: continuation passing style translation and closure conversion.

# Defining Correctness

We will explain the desired theorem of full abstraction correctness and operational correctness bottom-up, starting with defining the equivalence of two programs in the same language.

## Program Equivalence

First, a *program context* is defined as a whole program with a single hole that should be filled.
Think of this as a partially written program with a fill-in-the-blank where we will copy-paste our implementation.
Notably, the empty context (just a blank space and nothing more), is a valid program context: filling the blank space with the implementation just gives us back the implementation as a whole program.

*Kleene equivalence* of programs is the context-sensitive definition of program equivalence where we assume the programming context is empty.
For our purposes, it is defined as mutual termination, meaning that the two programs are Kleene equivalent if they either both loop forever or both reach a related final value.
Kleene equivalence is enough to show the operational correctness of compilation because it guarantees the context-sensitive case of whole-program correctness.

*Contextual equivalence* is the context-insensitive definition of programming equivalence, which is defined as mutual termination within *any* arbitrarily chosen program context.
This implies Kleene equivalence as the empty context is one such context.

You can understand this distinction by thinking of the chosen program context as a set of test cases.
We can opt for the context-sensitive version of correctness where we only test one particular set of test cases, or we could make the context-insensitive approach of quantifying over all possible test cases we can possibly write, even ones we didn't think of, and still show they are equivalent under these arbitrary test cases.
The context-insensitive approach might seem too good to be true: no matter how many test cases pass, the worry of an obscure, failing test case persists.
However, we can define this helpful tool known as a logical relation for these proofs of contextual equivalence.

A *logical relation* is defined inductively on the type structure of the programs it is comparing, so we can leverage type information about the program to prove *logical equivalence*.
We must also demonstrate that this logical equivalence corresponds to contextual equivalence in that they relate the same things, but rest assured that for our research, we have formally verified this complex result in a proof assistant.

## Full Abstraction Correctness

Formally, full abstraction correctness is defined as: if two source programs \\(e\\) and \\(e'\\) are contextually equivalent, and they compile to target programs \\(d\\) and \\(d'\\) respectively, then \\(d\\) and \\(d'\\) are contextually equivalent.

Contextual equivalence between programs inherently defines a type interface between the programs and an abstraction guarantee that the programs provide the same results under that interface.
A compiler that is fully abstract preserves these abstractions through compilation, meaning that the same interface that exists between \\(e\\) and \\(e'\\) still exists between \\(d\\) and \\(d'\\) without imposing additional restrictions as to how \\(d\\) and \\(d'\\) should be used to preserve this interface.

A reasonable example of breaking abstraction would be when a source program requires a specific memory location \\(m\\) to be "nonzero" and only allows you to reference this memory location through functions that preserve \\(m\neq 0\\).
Naively compiling this could accidentally exposes \\(m\\) to other parts of the program. Abstraction is preserved only if those other parts of the program abide by the rule of \\(m\neq 0\\).
Since this restriction is not enforced by the types or the language itself, it creates opportunities for a wide variety of bugs resulting from failure to preserve this invariant.

From a practical perspective, full abstraction correctness gives us the ability to "hot-swap" code at any stage of compilation, since contextual equivalence is preserved through compilation and is both compatible and transitive.
This capability is the driving force of many modern software practices.
If we want to change a single line of code in the source language but have already spent significant time compiling the old program, we can just recompile that single line on its own and hot-swap the old version post-compilation while still resulting in a correct compilation.
This notion of correctness also enables optimizations at any stage of compilation: we can perform a partial hot-swap of code at any compilation stage and still maintain the overall correctness proof.
The structure of this definition also allows for stacking arbitrarily many compilation phases together; we just need to show that contextual equivalence is preserved at each stage.

Looking carefully at the definition of full abstraction correctness, we find that a way to trivialize it is to compile every source program into a single target program: this would preserve the abstraction guarantees but it wouldn't be a very useful compiler.
We still need to prove operational correctness in addition to full abstraction correctness to show the compiler makes sense.
In the next section, we present a method of proving operational correctness that additionally grants us full abstraction correctness directly.

## Proving Correctness

If the source and target languages were the same and the statics of programs didn't change during compilation (not a very interesting compiler), then just knowing that the source and target programs are contextually equivalent is enough to prove full abstraction correctness by symmetry and transitivity of contextual-equivalence.

$$d \equiv e \equiv e' \equiv d'$$

However, in most cases, the source and target programs are not directly contextually equivalent but are contextually equivalent over a chosen interface.
This interface is defined through mutually inverse functions \\(\mathbf{over}\\) and \\(\mathbf{back}\\) that relate the logic of the source and target languages.
We must thus prove that \\(e \equiv \mathbf{back}(d)\\) or equivalently that \\(\mathbf{over}(e) \equiv d\\) to get full abstraction correctness through compatibility of contextual-equivalence.

$$d \equiv \mathbf{over}(e) \equiv \mathbf{over}(e') \equiv d'$$

We define \\(\mathbf{over}\\) and \\(\mathbf{back}\\) at base answer types such as \\(\mathbf{unit}\\) or \\(\mathbf{nat}\\) to be the identity function, thus showing that the source and target programs at those base types result in the same answer.
With this in mind, operational correctness is the special case of \\(e \equiv \mathbf{back}(d) \equiv d\\) for closed source programs \\(e\\) of the base answer type and their respective compiled program \\(d\\).
Operational correctness is proved through inducting over open source programs of arbitrary types with the generalization of this property as the induction hypothesis, and then extracting the special case afterwards.

You might be wondering, is \\(\mathbf{over}\\) the same as the compilation itself?
If it were, then this result is rather obvious because we would be comparing the compilation to the compilation.
To illustrate the difference, let's look at an example program and a silly compilation phase that computes every function argument twice.
Assume that \\(\overline{e1}\\) and \\(\overline{e2}\\) are the fully compiled versions of \\(e1\\) and \\(e2\\) respectively.

$$(\lambda (x : A) : \mathbf{unit} .\ e1)\ e2\ \rightsquigarrow\ (\lambda^2 (x : A) (x : A) : \mathbf{unit}\ .\ \overline{e1})\ \overline{e2}\ \overline{e2}$$

$$\mathbf{over} = \mathbf{back} = (\lambda (x : \mathbf{unit}) : \mathbf{unit} .\ x)$$

Here we see that the program is rather complex in its implementation and compilation, but it overall has the base type \\(\mathbf{unit}\\).
The compiler can observe how the program is defined and can chop up the syntax however it likes to produce a compiled result, but the \\(\mathbf{over}\\) and \\(\mathbf{back}\\) functions only know that the program has the type \\(\mathbf{unit}\\), and that is a base type that remains consistent before and after compilation, so the \\(\mathbf{over}\\) and \\(\mathbf{back}\\) functions are the identity function.
In this sense, the compiler is a conversion of the *intensional* properties of the program where the way the program is implemented does matter, whereas the \\(\mathbf{over}\\) function is a relation of the *extensional* behavior of the program.
The \\(\mathbf{over}\\) function is an interface wrapping up the program in a way that abstracts away the particular implementation, leaving you only able to condition the final result value you get from executing the program.
By comparing the compiler to the \\(\mathbf{over}\\) function, we are stating that the *intensional* changes the compiler makes adhere to the *extensional* properties of the original program, which is a really strong statement of correctness.

Now what remains is defining \\(\mathbf{over}\\) and \\(\mathbf{back}\\) for a given compilation phase.
In our research, we take the approach of taking all the syntax and semantics of the source language and combining them with the syntax and semantics of the target language, creating one large joint language that can comprehend both.
The novel contribution of our research is that we can then define \\(\mathbf{over}\\) and \\(\mathbf{back}\\) as functions within the joint language itself, and use contextual-equivalence within the joint language to show that \\(\mathbf{over}\\) and \\(\mathbf{back}\\) are mutual inverses.

This proof approach works really well when the source and target languages have similar semantics, and in practice, especially for higher-order compilation, it is often the case that the target language is a simplified subset of the source language, meaning we can just utilize the source language itself as the joint language.
This way, we separate the proof of the type-safety of the joint language from the overall compiler correctness proof, and in fact, we find we can reuse the same joint language for multiple layered phases of compilation, each with its definition of \\(\mathbf{over}\\) and \\(\mathbf{back}\\).

A related work that showcases this is Crary's work on Fully Abstract Module Compilation.
This employs a phase-separation algorithm for splitting modules into static and dynamic portions without reference to the module language.
This means we can go from a rich expressive language with modules as the source language, compile away the modules through this phase-separation technique, and arrive at a simpler target language without modules.

To reiterate, the inductive cases of compiler correctness that do not take part in the compilation procedure itself, such as pairs when the compilation only involves functions, are rather easy to solve with this approach.
The \\(\mathbf{over}\\) and \\(\mathbf{back}\\) are defined similarly to an eta-expansion at those types, in other words, they just carry the induction hypothesis along smoothly without doing anything complicated.

# Proof of Concept

To demonstrate that our compiler correctness procedure works, we proved the correctness of two separate compilation phases and then joined these two proofs together.

## Continuation Passing Style Phase

The continuation passing style (CPS) translation is commonly used to cleanly express the control flow of programs.
We convert a term \\(e\\) into a pair \\((k,d)\\) where \\(d\\) is a computation, and \\(k\\) is a continuation for which the computation returns a value once it is finished running.
Normally, these are a one-to-one correspondence: \\(d\\) only produces one value which gets sent to the continuation \\(k\\) at one location, but this translation phase becomes most interesting when we add control flow effects into our language which breaks this one-to-one correspondence.
The CPS translation can tame all these control flow effects, leaving us in a simpler language without explicit control flow operations.

Our source language contains the explicit control flow operations of \\(\mathbf{letcc}\\) and \\(\mathbf{throw}\\).
The constructor \\(\mathbf{letcc}\\) allows us to record the current state of the computer and save it as a continuation while \\(\mathbf{throw}\\) lets us jump to some previous state of the computer by calling the respective continuation.
You can pretend it is similar to an assembly jump operation, so you can imagine how it could be used to express conditional branching and short-circuiting of computations.

Interestingly, \\(\mathbf{over}\\) and \\(\mathbf{back}\\) for the CPS translation must use these control flow mechanisms within its definition.
Thankfully, we can prove that these particular control flow effects within the \\(\mathbf{over}\\) and \\(\mathbf{back}\\) are benign so we can maintain the properties we want with some extra effort.

## Closure Conversion Phase

A closure is an inline function that can utilize the initialized variables in the context within its definition.
Closure conversion (CC) translation takes these closures and turns them into global function definitions that can be moved into the top level instead of inlined.
It achieves this by packaging the variables in the context that the closure depends on into a large tuple and passing them as input into the global function to maintain the dependency.

Central to the proof of this phase is a parametricity relation between the original closure function and the resulting global function.
Parametricity allows us to show that two programs are contextually equivalent even if they are of different types, as long as we can state an interface where the types are related.
Function application through compilation becomes a parametric packing operation into an existential type before calling the function, granting us the interface we need in the proof of full abstraction correctness.

## Joining it All Together

Our final theorem for full abstraction correctness states that if we have \\(e_1 \rightsquigarrow_{CPS} (k, d_1) \rightsquigarrow_{CC} (k, t_1)\\) and similarly \\(e_2 \rightsquigarrow_{CPS} (k, d_2) \rightsquigarrow_{CC} (k, t_2)\\), and we know that
$$\Gamma \vdash e_1 \equiv e_2 : \tau\ \ \text{implies}\ \ \overline{\Gamma}, k : \overline{\tau}\to0\vdash t_1\equiv t_2 : 0.$$
This essentially says that the equivalent \\(e_1\\) and \\(e_2\\) programs compile into equivalent programs \\(t_1\\) and \\(t_2\\), even with both phases of compilation put together.
We successfully demonstrate that this correctness approach works for these two phases of compilation by rigorously proving this theorem.

With this, we have completed our proof of full abstraction compiler correctness for two compilation passes layered together utilizing a single joint language definition.
In future work, we hope to add more layers of compilation to our proof and tackle some difficult compilation phases such as memory allocation and garbage collection.