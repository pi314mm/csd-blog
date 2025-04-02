+++
# The title of your blogpost. No sub-titles are allowed, nor are line-breaks.
title = "Full Abstraction Correctness of Compilation"
# Date must be written in YYYY-MM-DD format. This should be updated right before the final PR is made.
date = 2025-03-31

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
    
]
+++

Compilation correctness has always been an important concern: even if we write wonderfully pristine code, it means nothing if bugs are injected during compilation.

Compilation is the process of converting from a human-comprehensible, mathematically rich programming language into a machine-readable, operationally explicit target language, and can involve many phases stacked on top of one another.
Compilation correctness therefore relates the original program the user supplies to the target program that is output by stating that they "do the same thing" by our observation.

This blog post will explore the complexity of verifying compilation correctness, delve into its mathematical definition, and detail the specific approach we adopted in our research.

# Difficulties in Proving Compilation Correctness

"You can't compare apples to oranges" is a good way to sum up one of the main sources of complexity in compilation proofs.
The source and target languages that take part in the compilation are different languages that could have completely different memory models or operational behavior.
We (usually) can't prove that the source program and compiled program have the "same" behavior, we must instead prove that they have "related" behavior by some definition of "related" that we must clearly define.

We can use the analogy of translating between two different spoken languages, say English and Spanish, as many of the same problems occur in computer program compilation.
An English sentence is trying to carry along with it a certain logical meaning.
A translator would need to understand both English and Spanish to understand and translate the meaning.
Sometimes, the same exact meaning can be conveyed; there are even a lot of words in Spanish directly acquired from English.
However, sometimes the meaning gets morphed in translation, and information could get lost: a pun in English could be translated directly to Spanish, so the direct translation would make practical sense, but the humor of the pun would be lost in translation.

That's a lot of information, so let's unpack it sentence by sentence.

- An English sentence is trying to carry along with it a certain logical meaning

In terms of programming languages, this means that the source language is well-defined with semantics that give it logical meaning, and this meaning isn't necessarily tied down to the target language's semantics.
Sometimes, a compilation proof looks more like showing that the compiler implementation adheres to a specification (such as the C standard), and the source language doesn't need proper semantics but just exists post-compilation.
This is a reasonable viewpoint, but we will consider the case where we do have well-defined source and target languages which is more common in higher-order stages of compilation.

- A translator would need to understand both English and Spanish

This is a key point.
The metalogic to compare the source and target language must exist in a language that can comprehend both in conjunction to establish proofs of similarity between both languages.
This is where compilation correctness approaches differ: some examples include defining a cross-language relation, defining some simplified common language of program signals, or, in our case, defining a cohesive joint language strapping both languages together.

- A lot of words in Spanish directly acquired from English

Program compilation procedures often have many steps stacked on top of each other, each step dealing with some very specific task.
This means a lot of the intermediate languages are very similar, or possibly even identical, in their semantics to one another, or have one small change and keep the rest the same.
Our correctness-proof strategy is able to leverage this to its advantage, handling the easy cases with ease so we can focus on the hard cases, showcasing one of the major benefits of our approach.
On the other hand, if the source and target languages are substantially different, our proof strategy would still work in theory but become much messier and impractical to deal with compared to other approaches.

- A pun in English could be translated directly to Spanish, so the direct translation would make practical sense, but the humor of the pun would be lost in translation

The analogy helps us realize there can be several different interpretations of "compilation correctness" where some guarantees can be stronger than others.
It is fair to say that the direct translation from English to Spanish is correct as the informational content is going to be the same, but one could also argue that it is not correct because it doesn't include important contextual information about it being a pun.

This is related to the difference between operational correctness and full abstraction correctness of compilation.
Operational correctness states that the source and target programs adhere to each other when viewed as a whole-program translation, but it is context-sensitive.
Full abstraction correctness is a stronger statement because it implies operational correctness, but it is also context-insensitive.

Think of it this way: if by default we know each sentence is supposed to have a pun (pre-defined context), then the direct translation from English to Spanish is enough.
However, if we try to copy-paste this direct translation into some other paragraph where we don't know by default that every sentence is a pun (change the context), then we have created a bug in our translation because we lost the pun content.
A stronger translation would be to use a Spanish pun as well so that the reader can understand the intended message regardless of the contextual information.

With the scale of modern-day programming practices, we often need full abstraction correctness instead of just operational correctness so that our compiler can be compositionally correct.
Without it, bugs could be introduced post-compilation when we try to import a pre-compiled library in the wrong context, or if we try to link together programs from various sources and compilers that can disagree on their chosen context.

Our research demonstrates a new proof technique that can be used to verify full abstraction correctness via type-directed joint language merging.
We have shown the efficacy of this approach by proving full abstraction correctness for two well-known phases of compilation: continuation passing style translation and closure conversion.

# Defining Full Abstraction Correctness

We will explain the desired theorem of full abstraction correctness bottom-up, starting with defining the equivalence of two programs in the same language.

## Program Equivalence

First, a program context is defined as a whole program with a single hole that should be filled.
Think of this as a partially written program with a fill-in-the-blank where we will copy-paste our implementation.

Notably, the empty context (just a blank space and nothing more), is a valid program context: filling the blank space with the implementation just gives us back the implementation as a whole program.
Kleene-equivalence of programs is the context-sensitive definition of program equivalence where we assume the programming context is empty, and for our purposes, it is defined as mutual termination.
Kleene-equivalence is enough to show the dynamic correctness of compilation because it guarantees the context-sensitive case of whole-program correctness.

Contextual equivalence is the context-insensitive definition of programming equivalence, which is defined as mutual termination within *any* arbitrarily chosen program context.
This implies Kleene-equivalence as the empty context is one such context.

You can understand this distinction by thinking of the chosen program context as a set of test cases.
We can opt for the context-sensitive version of correctness where we only test one particular set of test cases, or we could make the context-insensitive approach of quantifying over all possible test cases we can possibly write, even ones we didn't think of, and still show they are equivalent under these arbitrary test cases.
This might seem too good to be true, however, we can define this helpful tool known as a logical relation to help aid us in these proofs of contextual equivalence.

A logical relation is defined inductively on the type structure of the programs it is comparing, so we can leverage type information about the program to prove logical equivalence.
We must also demonstrate that this logical equivalence corresponds to contextual equivalence in that they relate the same things, but rest assured that for our research, we have formally verified this complex result in a proof assistant.

## Full Abstraction Correctness

Formally, full abstraction correctness is defined as: if two source programs \\(e\\) and \\(e'\\) are contextually equivalent, and they compile to target programs \\(d\\) and \\(d'\\) respectively, then \\(d\\) and \\(d'\\) are contextually-equivalent.

Why is this what we want?
Well, if we look at the given that \\(e\\) and \\(e'\\) are contextually equivalent, we can comprehend this as if we were to hot-swap \\(e'\\) for \\(e\\) in our program, there would be no observable difference.
The logical message that \\(e\\) and \\(e'\\) are trying to convey is the same, even if the two particular ways of explaining that message are different.
By knowing that \\(d\\) and \\(d'\\) are contextually equivalent after compilation, we learn that the logical message gets preserved post-compilation: the compiler is somewhat agnostic to the particular way the logical message is represented in the source language, it compiles in such a way that preserves the logical message the source programs are trying to convey.

When viewed from a practical perspective, this ability to hot-swap before and after compilation is the driving force of modern software practices.
If we want to change a single line of code in the source language, but we already spent many minutes compiling the old program, we can just recompile the single line on its own and hot-swap the old version of the line post-compilation, and compilation correctness would still be preserved.
This notion of correctness also enables optimizations at any stage of compilation: we can do a partial hot-swap of code at any stage of compilation and still maintain the overall correctness proof.
The structure of this definition also allows for stacking arbitrarily many compilation phases together, you just need to show that contextual equivalence is preserved at each stage.

## Proving Full Abstraction Correctness

If the source and target languages were the same and the statics of programs didn't change during compilation (not a very interesting compiler), then just knowing that the source and target programs are contextually equivalent is enough to prove full abstraction correctness by symmetry and transitivity of contextual-equivalence.

$$d \equiv e \equiv e' \equiv d'$$

However, in most cases, the source and target programs are not directly contextually equivalent but are contextually equivalent over a chosen interface.
This interface is defined through mutually inverse functions \\(\mathbf{over}\\) and \\(\mathbf{back}\\) that relate the logic of the source and target languages.
We must thus prove that \\(e \equiv \mathbf{back}(d)\\) or equivalently that \\(\mathbf{over}(e) \equiv d\\) to get full abstraction correctness through compatibility of contextual-equivalence.

$$d \equiv \mathbf{over}(e) \equiv \mathbf{over}(e') \equiv d'$$

You might be wondering, is \\(\mathbf{over}\\) the same as the compilation itself?
If it were, then this result is rather obvious because we would be comparing the compilation to the compilation.
To illustrate the difference, let's look at an example program and a silly compilation phase that computes every function argument twice.
Assume that \\(\overline{e1}\\) and \\(\overline{e2}\\) are the fully compiled versions of \\(e1\\) and \\(e2\\) respectively.

$$(\lambda (x : A) : \mathbf{unit} .\ e1)\ e2\ \rightsquigarrow\ (\lambda^2 (x : A) (x : A) : \mathbf{unit}\ .\ \overline{e1})\ \overline{e2}\ \overline{e2}$$

$$\mathbf{over} = \mathbf{back} = (\lambda (x : \mathbf{unit}) : \mathbf{unit} .\ x)$$

Here we see that the program is rather complex in its implementation and compilation, but it overall has the base type \\(\mathbf{unit}\\).
The compiler can observe how the program is defined and can chop up the syntax however it likes to produce a compiled result, but the \\(\mathbf{over}\\) and \\(\mathbf{back}\\) functions only know that the program has the type \\(\mathbf{unit}\\), and that is a base type that remains consistent before and after compilation, so the \\(\mathbf{over}\\) and \\(\mathbf{back}\\) functions are the identity function.
In this sense, the compiler is a conversion of the *intensional* behavior of the program where the way the program is implemented does matter, whereas the \\(\mathbf{over}\\) function is a relation of the *extensional* behavior of the program.
The \\(\mathbf{over}\\) function is an interface wrapping up the program in a way that abstracts away the particular implementation, leaving you only able to condition the final result value you get from executing the program.
By comparing the compiler to the \\(\mathbf{over}\\) function, we are stating that the *intensional* changes the compiler makes adhere to the *extensional* properties of the original program, which is a really strong statement that grants us full abstraction correctness of compilation.

Now what remains is defining \\(\mathbf{over}\\) and \\(\mathbf{back}\\) for a given compilation phase.
In our research, we take the approach of taking all the syntax and semantics of the source language and combining them with the syntax and semantics of the target language, creating one large joint language that can comprehend both.
The novel contribution of our research is that we can then define \\(\mathbf{over}\\) and \\(\mathbf{back}\\) as functions within the joint language itself, and use contextual-equivalence within the joint language to show that \\(\mathbf{over}\\) and \\(\mathbf{back}\\) are mutual inverses.

This proof approach works really well when the source and target languages have similar semantics, and in practice, especially for higher-order compilation, it is often the case that the target language is a simplified subset of the source language, meaning we can just utilize the source language itself as the joint language.
This way, we separate the proof of the type-safety of the joint language from the overall compilation correctness proof, and in fact, we find we can reuse the same joint language for multiple layered phases of compilation, each with its definition of \\(\mathbf{over}\\) and \\(\mathbf{back}\\).

To reiterate, the inductive cases of compilation correctness that do not take part in the compilation procedure itself, such as pairs when the compilation only involves functions, are rather easy to solve with this approach.
The \\(\mathbf{over}\\) and \\(\mathbf{back}\\) are defined similarly to an eta-expansion at those types, in other words, they just carry the induction hypothesis along smoothly without doing anything complicated.

# Proof of Concept

To demonstrate that our compilation correctness procedure works, we proved the correctness of two separate compilation phases and then joined these two proofs together to achieve full abstraction correctness.

## Continuation Passing Style Phase

The continuation passing style (CPS) translation is commonly used to cleanly express the control flow of programs.
We convert a term \\(e\\) into a pair \\((k,d)\\) where \\(d\\) is a computation, and \\(k\\) is a continuation for which the computation returns a value once it is finished running.
Normally, these are a one-to-one correspondence: \\(d\\) only produces one value which gets sent to the continuation \\(k\\) at one location, but this translation phase becomes most interesting when we add control flow effects into our language which breaks this one-to-one correspondence.
The CPS translation can tame all these control flow effects, leaving us in a simpler language without explicit control flow operations.

Our source language contains the explicit control flow operations of \\(\mathbf{letcc}\\) and \\(\mathbf{throw}\\).
The constructor \\(\mathbf{letcc}\\) allows us to record the current state of the computer and save it as a continuation while \\(\mathbf{throw}\\) lets us jump to some previous state of the computer by calling the respective continuation.
You can pretend it's similar to an assembly jump operation, so you can imagine how it could be used to express conditional branching and short-circuiting of computations.

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

With this, we have completed our proof of full abstraction compilation correctness for two compilation passes layered together utilizing a single joint language definition.
In future work, we hope to add more layers of compilation to our proof and tackle some difficult compilation phases such as memory allocation and garbage collection.