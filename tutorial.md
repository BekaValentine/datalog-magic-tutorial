# On the power of sufficiently advanced technology: demystifying Datalog's Magic

A tutorial that attempts to explain the concept of Magic used in the
implementation of Datalog. This tutorial assumes that you know a little bit
about Prolog-style logic programming languages.

## Introduction

The Datalog programming language has a long and interesting history. One of the
overriding themes, however, has been the quest for efficiency. Most languages
have strived for efficiency, however, so why is that such an important thing in
the history of Datalog? Well, because for a very long time, Datalog was so
utterly inefficient that you literally could not use it for anything beyond toy
programs.

Why was that so? Well, the standard way of evaluating Datalog is what's called
Bottom Up evaluation. If you're familiar with parsing, it's very similar to
bottom up parsing techniques. Roughly, bottom up evaluation proceeds by taking
everything that is known to be true at a given moment, and applies the rules
of the Datalog program to those facts, to derive new facts, just giving rise to
a new set of things that are known to be true. This process is repeated until
no new facts are derived by further application of rules.

Various techniques exist to make this process more efficient than just the naive
approach of iterating until you can stop, such as the semi-naive algorithm that
requires the next iteration to make use of at least one newly derived fact from
the previous iteration, thereby ensuring that all the facts derived in the next
iteration are also new. Another major development in the quest for efficiency
was Magic.

## Motivating Magic

Magic is a code transformation that converts a Datalog program into another,
supposedly more efficient, Datalog program. The motivation for magic is the
observation that when doing bottom up evaluation, the vast majority of the facts
that are derived are, well, useless. Let's consider an example program. We'll
use the archetypal ancestor program as our running example throughout.

```prolog
% The extensional predicates that define the base facts of the program.

parent(avery, blair).
parent(blair, charlie).
parent(charlie, dakota).
parent(emerson, finley).
parent(finley, greyson).

% The intensional predicates the define the inferable facts of the program.

ancestor(X,Y) :- parent(X,Y).
ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y).
```

If we run bottom up evaluation on this, we get the following facts, separated
out by which step of the evaluation they show up in:

```prolog
% Step 0

parent(avery, blair).
parent(blair, charlie).
parent(charlie, dakota).
parent(emerson, finley).
parent(finley, greyson).

% Step 1

ancestor(avery, blair).
ancestor(blair, charlie).
ancestor(charlie, dakota).
ancestor(emerson, finley).
ancestor(finley, greyson).

% Step 2

ancestor(avery, charlie).
ancestor(blair, dakota).
ancestor(emerson, greyson).

% Step 3

ancestor(avery, dakota).
```

Now, suppose we had started this program evaluation off by querying like so:

```prolog
?- ancestor(finley,Y).
```

The bottom up evaluation process would run the program by building that full set
of facts given above, and then would look up every fact of the form given by
our query. It would then find

```prolog
X = greyson
```

That's it. That's all it finds. Because of course that's all there is: Greyson
is indeed the only ancestor of Finley according to the program. But it computes
the entire set of facts because, when working bottom up, the evaluator has no
way of knowing which facts are relevant to the result. If we had queried instead
for the ancestors of Avery, or Blair, or Charlie, or Dakota, or Emerson, it
would make no difference: the exact same work would be done, the exact same set
of total facts would be derived.

And this is why the Magic transformation exists: to provide some means of making
use of the information given by the query to constrain the evaluation of the
Datalog program to only those parts that are relevant to the answer.

## Through Prolog

The core idea behind Magic is to take some cues from Prolog. Why? Because, well,
Prolog doesn't have this problem. Prolog works in a so called Top Down fashion,
which means that it starts from the query and then proceeds to decompose that
into sub-queries which, when they've all ben completed, yield the answer to the
overall query. By working in this direction, the Prolog evaluator is able to
push information from the parent queries down into the child queries, thus
limiting the search space.

Let's look at the evaluation of the Prolog version of our above Program and
query. We'll use a slightly idealized version of Prolog to avoid irrelevant
details. We start with just the query itself:

```prolog
?- ancestor(finley,Y).
```

In one step, we decompose this into two sub-queries:

```prolog
?- parent(finley,Y).
?- parent(finley,Z), ancestor(Z,Y)
```

The first of these subqueries will match against facts, and then immediately
succeed with

```prolog
Y = greyson
```

While the second of these subqueries will also match against facts, yielding a
new binding `Z = greyson`, which leads to the new query

```prolog
?- ancestor(greyson,Y)
```

This expands similarly to our initial query into

```prolog
?- parent(greyson,Y)
?- parent(greyson,Z), ancestor(Z,Y)
```

Both of these now fail because no parent facts for `greyson` exist. We end with
only the binding `Y = greyson`, and we've done only a very small amount of work.

These flat lists of atomic sentences that constitute our queries and subqueries
can be seen as the unexplored frontier nodes of a tree of atomic propositions
that form the proof tree for the query we're performing. This is one reason why
it's also not uncommon to hear Prolog's standard evaluation strategy be
described as `depth first`.

# Inventing Magic

We come to the design of Magic by observing the structure of recursion that's
being flattened out here in the evaluation of Prolog, and mix that with the
standard abstract machines found in functional programs. One of the most common
such machines for functional languages is the CEK machine, which we'll steal
from. In particular, the CEK machine has two different kinds of machine states:
one kind of state that designates when a part of the program is about to be
entered into for evaluation, and another kind of state that designates when a
part of the program has finished being evaluated.

Without using fancy notation, let's just consider how a CEK machine evaluates
an expression such as `1 + (2 + 3)`. We start out by entering `1 + (2 + 3)`,
because that's the start of the entire problem. We then proceed to the left
child, so we step the machine to be entering `1`. Since `1` is a number literal,
it's already a value and we can move on to exiting it with the value of `1`.
From there we step to entering `2 + 3`, and so on. Eventually we find ourselves
exiting `3` with the value `3`. Since this is the last operand inside `2 + 3`,
we now can compute that sum, and exit from `2 + 3` with the value `5`. This then
lets us again compute a sum, and exist `1 + (2 + 3)` with the value `6`.

Written out, these steps are as follows:

```
   Enter 1 + (2 + 3)
-> Enter 1
-> Exit 1 with 1
-> Enter 2 + 3
-> Enter 2
-> Exit 2 with 2
-> Enter 3
-> Exit 3 with 3
-> Exit 2 + 3 with 5
-> Exit 1 + (2 + 3) with 6 (i.e. 1 + 5)
```

Our Prolog evaluation can be given a similar treatment. Because the evaluation
is non deterministic, there are multiple traces for this query, only one of
which exists from the initial query. The rest fail.

```
% Trace 0

   Enter ancestor(finley,Y)
-> Enter parent(finley,Y)   % using ancestor(X,Y) :- parent(X,Y).
-> Exit parent(finley,Y) with Y = greyson
-> Exit ancestor(finley,Y) with Y = greyson

% Trace 1

   Enter ancestor(finley,Y)
-> Enter parent(finley,Z)   % using ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y).
-> Exit parent(finley,Z) with Z = greyson
-> Enter ancestor(greyson,Y)  % ancestor(Z,Y) substituting for Z
-> Enter parent(greyson,Y)  % using ancestor(X,Y) :- parent(X,Y).
-X FAIL because there are no values for Y

% Trace 1

   Enter ancestor(finley,Y)
-> Enter parent(finley,Z)   % using ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y).
-> Exit parent(finley,Z) with Z = greyson
-> Enter ancestor(greyson,Y)  % ancestor(Z,Y) substituting for Z
-> Enter parent(greyson,Z)  % using ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y).
-X FAIL because there are no values for Z
```

What we can sort of notice here is that in all of these top down executions, the
set of states that we've visited only grows. That is to say, if we've reached
the situation of having gone through a trace `S_0 -> ... -> S_n`, then if we
look at any of the already-visited states and "re-run" the execution from there,
we simply re-generate the states after it. We don't get new states unless we've
running the evaluation from the most recently generated states.

This sounds an *awful* lot like Datalog's Bottom Up evaluation, and the reason
is that it *is* a kind of bottom up evaluation. In particular, the above traces
are proofs of the transitive closure of a binary step relation between machine
states. We denoted each step in that transitive closure via `S_i -> S_i+1`, and
the whole sequence of those is part of the transitive closure `S_i ->* S_j`,
which can be computed bottom up That's what `ancestor` is, after all: a
transitive closure of `parent` (though in that case it's `parent+` not `parent*`).

With this in mind, we can reinvent Magic. The above traces through a Prolog
program have machine states with atomic propositions as arguments, but Datalog
doesn't have compound data like that, so we couldn't very well have something
like this, for example:

```prolog
step(enter(ancestor(X,Y)), exit(ancestor(X,Y))) :- parent(X,Y).
```

Prolog would permit that of course, but we're using Datalog, so we need another
solution. But Datalog provides one: we can instead treat the step relation as
just the single step of Datalog evaluation, and instead of having states like
`enter(ancestor(X,Y))`, we instead have propositions like `enter_ancestor(X,Y)`.
The inference rules we need for the enter direction are then sort of "backwards"
compared to normal rules, because they correspond to the downward direction of
the Prolog CEK machine's traversal of the program. The rules ought to look
something like this:

```prolog
enter_parent(X,Y) :- enter_ancestor(X,Y).
exit_ancestor(X,Y) :- exit_parent(X,Y).
```

This is only a few of the rules that we might expect to have, to capture some of
the way the states evolve in the Prolog evaluator. But if we do this, we notice
that there's a bit of a problem. Namely, we aim to evaluate queries like
`ancestor(finley,Y)`, which have free variables, which means the values of those
free variables ought to be found by evaluation. That is to say, they have to be
provided bottom-up. But if we start out this process by initializing our program
with the fact `enter_ancestor(finley,Y).`, and then try to let the Datalog
evaluator proceed as normal, it will complain: `Y` is not bound to a
value, so `enter_ancestor(finley,Y)` is not a valid fact!

What's missing from the translation so far is the separation of information into
flow directions. The whole point of this re-framing of the program is to push
information down from the query to the sub-queries, and percolate query results
back up. All of our `enter_foo` predicates push information down, and don't
percolate anything back up, so they don't *need* those free variables at all.
So let's remove them. The various state transitions we want to have are now
captured in the following rules:

```prolog
exit_ancestor(X,Y) :- enter_ancestor(X), parent(X,Y).
enter_ancestor(Z) :- enter_ancestor(X), parent(X,Z).
exit_ancestor(X,Y) :- enter_ancestor(X), parent(X,Z), exit_ancestor(Z,Y).
```

We have no rules for entering or exiting `parent` propositions because those
are all facts that don't derive from inference, and so we don't need to worry
about adding new useless `parent` facts during evaluation. Correspondingly, we
don't need to distinguish between entering and exiting of `parent` propositions
in our rules, because the totality of those is already written in the program.

Each of these rules corresponds directly to one of the steps we need to
capture:

```
% Steps for computing `ancestor(X,Y)`
% using the rule `ancestor(X,Y) :- parent(X,Y).`

Enter ancestor(X,Y) -> Enter parent(X,Y)
Exit parent(X,Y) -> Exit ancestor(X,Y)

% Steps for computing `ancestor(X,Y)`
% using the rule `ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y)`

Enter ancestor(X,Y) -> Enter parent(X,Z)
Exit parent(X,Z) -> Enter ancestor(Z,Y)
Exit ancestor(Z,Y) -> Exit ancestor(X,Y)
```

Notice that the first step for the recursive `ancestor` rule is actually the
same as one of the steps for the non-recursive `ancestor` rule, so our Datalog
has only 4 rules to capture the steps.

You might be wondering, though, why some of our rules have `enter`
on the right hand side of an `exit` rule. Namely,

```prolog
exit_ancestor(X,Y) :- enter_ancestor(X), exit_parent(X,Z), exit_ancestor(Z,Y).
```

This is because we want to ensure that we compute this exit step only because
we've reached it by first having entered into a computation of this ancestor
proposition. If we didn't include these, then our program would include all of
the original Datalog program as a sub-program, and we'd still have the original
inefficiency. These additions therefore force the bottom-up evaluation to happen
only *after* we've performed the simulation of top-down evalution to push in
the constraining contextual information.

More visually, these rules correspond to being at different points in the
evaluation of the original rules, seen as Prolog top-down rules:

```prolog
exit_ancestor(X,Y) :- enter_ancestor(X), parent(X,Y)
```

corresponds to having used the rule

```prolog
ancestor(X,Y) :- parent(X,Y).
```

to decompose a proposition, and now having successfully completed the `parent(X,Y)`
sub-query, which grants us the ability to finish the whole `ancestor(X,Y)` query
and propagate the results up. We can denote this by putting an `@` sign in the
rule at the location we're in:

```prolog
ancestor(X,Y) :- parent(X,Y) @.
```

If we do this with all the Prolog rules, putting `@`s in all the possible places,
we get

```prolog
ancestor(X,Y) :- @ parent(X,Y).
ancestor(X,Y) :- parent(X,Y) @.
ancestor(X,Y) :- @ parent(X,Z), ancestor(Z,Y).
ancestor(X,Y) :- parent(X,Z), @ ancestor(Z,Y).
ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y) @.
```

The first of these annotated rules is unnecessary because `parent` is extension,
as previously noted, so entering it as a sub-query immediately succeeds with a
lookup. The third is similarly unnecessary, leaving only

```prolog
ancestor(X,Y) :- parent(X,Y) @.
ancestor(X,Y) :- parent(X,Z), @ ancestor(Z,Y).
ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y) @.
```

That is to say, we can be finishing `parent(X,Y)` while working on `ancestor(X,Y)`
so we can finish `ancestor(X,Y)`, or we can be starting `ancestor(Z,Y)` while
working on `ancestor(X,Y)` and after having found `parent(X,Z)`, or we can be
finishing `ancestor(Z,Y)` while working on `ancestor(X,Y)` and after having
found `parent(X,Z)`.

Again we note a similarity to parsing, where you find things such as
`Term -> Term • + Term` to represent the progress of a parser as it works its
way through a particular grammar rule.

The final definition of Magic we have, now, is a transformation that looks like
this:

Given a rule of the form

```
p(X, ...) :- q0(Y, ...), q1(Z, ...), ..., qn(W, ...)
```

generate rules of the form

```
enter_q0(Y, ...) :- enter_p(X, ...).
enter_q1(Z, ...) :- enter_p(X, ...), exit_q0(Y, ...).
...
enter_qn(W, ...) :- enter_p(X, ...), exit_q0(Y, ...), exit_q1(Z, ...), ..., exit_qn-1(...).
exit_p(X, ...) :- enter_p(X, ...), exit_q0(Y, ...), ..., exit_qn(Z, ...)
```

This isn't *completely* correct, there is still some nuance, but the shape of it
is correct.

My personal favorite visualization is a local subtree of the proof structures,
with marks for the entering and exiting "sides" of a proposition, and arrows
connecting them in the appropriate way:

!!! IMAGE

I find these images to be very easy to conjure , and they lend themselves nicely
to manually generating the result of Magic. So while the above symbolic
representation is useful, I find the visualization to be vastly more useful for
actually remembering just what's going on with Magic.

# The Nuance

There is some nuance that I've omitted in this discussion. In particular, there
are two issues which are crucial for actually implementing Magic which I've left
out entirely because, while necessary, they obscure the essence and intuition of
Magic. The first of these is binding patterns. We discussed above that Magic
works to push information "down" from the query context so that it can constrain
the bottom-up computation. To properly do magic, then, we need to account for
the different ways in which a predicate can be used. Our example query was
`ancestor(finley,Y)`, so we had one bound variable for `ancestor` and one free
variable. But we could have had the reverse, and had the query `ancestor(X,greyson)`.

In implementations of Magic, we want to distinguish because all of these
different usage patterns, as different versions of the original rules. This then
gives rise to different subsets of the general pattern of Magic described at the
end of the last section, because we only ever enter a predicate with bound
variables.

The other bit of nuance is that the order in which we compute subqueries is
undefined in Datalog but it's left to right in Prolog. When defining Magic above,
the rules we generated via Magic took a left to right approach to mirror Prolog,
but what we actually need to do is propagate information between subproblems
based on the binding of information. For instance, in the recursive rule for
`ancestor`

```prolog
ancestor(X,Y) :- parent(X,Z), ancestor(Z,Y).
```

We probably want to compute `parent(X,Z)` first, get values for `Z`, the proceed
to `ancestor(Z,Y)`, but some other programs might have different orders that are
right to left, or some other permutation. For instance, we might instead have
had the rule

```prolog
ancestor(X,Y) :- ancestor(X,Z), ancestor(Z,Y).
```

which can be done in either direction. The choice of ordering, called a Sideways
Information Passing Strategy (a SIPS), is something that any actual
implementation has to establish. However, the choice is usually based on
performance considerations, and so I've omitted discussion of SIPS in the
preceding sections because it's a separate matter of efficiency with its own
research and pragmatic factors.

# Conclusion

Hopefully this post has somewhat clarified that Magic is not, in fact, magic,
but rather it is merely Sufficiently Advanced Technology. Namely, it is the
Datalog encoding an abstract machine's state transition relation!
