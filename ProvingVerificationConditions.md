# Proving CLP programs originated from Verification Conditions #

CLP (also called Constrained Horn Clauses, CHC) provides a suitable formalism for expressing veriﬁcation conditions that guarantee the correctness of imperative, functional, or concurrent programs. Since FTCLP is a solver for CHC we can use it in order to prove whether a set of verification conditions (VCs) holds.

These VCs can come from safety or termination as well as from different programming languages. Once we translated those to Constrained Horn clauses we can use directly FTCLP to prove them. This is one of the key advantages of using CLP as intermediate language since we do not need to implement different tools for different languages or properties. Moreover, since the use of CLP as intermediate representation in verification tools is becoming more popular, it is much easier to communicate FTCLP with other verifiers.

The VCs must be encoded in such way that they hold iff its corresponding CLP program has no solutions.

<a href='Hidden comment: 
The solver is a direct application of [https://code.google.com/p/ftclp/ Failure-Based Tabling Constraint Logic Programming (FTCLP)]. In FTCLP, the Horn clauses are symbolically executed in a _top-down_, depth-first search manner similarly to CLP (Constraint Logic Programming) systems. The main difference is that since relations can be recursive, FTCLP uses interpolation to generate candidates in order to prove by induction that the search can be safely stopped for clauses with _infinite derivations_. FTCLP has similarities to other tools based on Horn clauses
such as [http://www7.in.tum.de/tools/hsf/ HSF] and [http://research.microsoft.com/en-us/um/people/leonardo/muze.pdf muZ] from [http://z3.codeplex.com/ Z3] but it exhibits some important differences as well. To mention some of them, HSF solves non-recursive Horn clauses originated from an abstracted program using predicate abstraction. muZ relies on a _bottom-up_ datalog engine which can be instantiated with different algorithms (e.g., PDR, Bounded model checking, etc). FTCLP can be implemented as an extension to any CLP system. One of the advantages of this is that it can benefit from years of research in the CLP community. For instance, FTCLP can be implemented on the top of an efficient Tabling system.
'></a>

# Installation #

We provide a script called _horn-prover_ that specializes FTCLP (by choosing some suitable options) for proving verification conditions.

Go to https://code.google.com/p/ftclp and install FTCLP.

SVN checkout the repository of this project, and update your PATH variable as follows:

```
export PATH=${PATH}:path_to_svn_repo/bin/horn_prover
```

# Usage #

```

horn_prover filename.pl -entry E [ <options> ]
       E     is a single atom between single quotes (e.g., 'prove')
options:
       -integer-arithmetic interpret all constraints over integer arithmetic (default real arithmetic)
       -min-unroll <n>     minimum num of unrollings for recursive clauses before attempt child-parent subsumption
       -max-unroll <n>     maximum num of unrollings for recursive clauses (default: no limit)
       -dump-interpolants  write interpolants to filename.intp
       -show-proof         write the proof tree in a .dot file
```

# Examples #



Read this [document](http://www.clip.dia.fi.upm.es/~jorge/docs/ftclp_appendix.pdf) to learn more about how to encode C programs and, for instance, their safety properties into Constrained Horn clauses. Also, this document shows examples of safe inductive invariants computed by the tool.

As an example, suppose the following C program fragment from the paper "A Practical and Complete Approach to Predicate Refinement":

```
 x=i; y=j;
 while (x!=0){
   x--;
   y--;
 }
 if (i==j)
    assert(y<=0);
```

Next, we translate the above program into a set of CHCs using an encoding based on a **small-step** semantics form:

```
state(X1,...,Xn) :- C(X1,...,Xn,X1',...,Xn'), next_state(X1',...,Xn').
```

<a href='Hidden comment: 
where basically each basic block is translated into a CHC. Note that we do not encode the verification conditions originated from proving the above program. Instead, we simply translate the program as it is into CHCs:
'></a>


```
prove :- 
	clp_meta([ X .=. I, Y .=.J ]), l(X,Y,I,J).

:- tabled(l(num,num,num,num)).
l(X,Y,I,J):- 
	clp_meta([ X .<>.0,  
	           X1 .=. X-1, Y1 .=. Y-1]), 
	l(X1,Y1,I,J).
l(X,Y,I,J):- 
	clp_meta([X .=. 0]), 
	err(X,Y,I,J).

:- tabled(err(num,num,num,num)).
err(_X,Y,I,J):- clp_meta([I .=. J, Y .>. 0]).

```


First note that we have encoded the loop using the relation l/4. The first rule of l/4 is recursive and corresponds to the body of the loop. The second rule of l/4 corresponds to the exit of the loop. Note that the err/4 relation encodes the safety property and it is the negation of the condition that appears in the C assert. The initial state has been encoded within the relation prove/0 before we call l/4. It should not been hard to see that the original C fragment is safe iff the relation prove/0 has no solutions.

Alternatively, we can encode the verification conditions originated form the above program using **Hoare-logic** based encoding:

```

prove :- 
	inv(X,Y,I,J),
	clp_meta([X .=. 0]), 
	clp_meta([I .=. J]),
	err(X,Y,I,J).

:- tabled(inv(num,num,num,num)).
% base case
inv(X,Y,I,J):- 
	clp_meta([ X .=. I, Y .=.J ]).
% inductive case
inv(X1,Y1,I,J):- 
	clp_meta([ X0 .<>.0,  
	           X1 .=. X0-1, Y1 .=. Y0-1]),
	inv(X0,Y0,I,J).
          
:- tabled(err(num,num,num,num)).
err(_X,Y,I,J):- clp_meta([Y .>. 0]).
```

FTCLP reports no solutions with either the above small-step or Hoare-logic encodings which means that the original C program is safe.

<a href='Hidden comment: 
Another classic example is the [http://en.wikipedia.org/wiki/McCarthy_91_function McCarthy 91 function]:

```
mc(n) {
 if (n>100) 
    n-10;
 else 
    mc(mc(n+11));
}
```

This program requires interprocedural reasoning in order to prove that for all n, mc(n) >= 91.

We would encode the McCarthy 91 function as follows:

```
prove :- 
	mc(X,R),
	clp_meta([ R .<. 91]).

:- tabled(mc(num,num)).
mc(X,R):- 
	clp_meta([
                  X .>. 100, 
                  R .=. X - 10
                 ]). 

mc(X,R):- 
        clp_meta([
                  X .=<. 100,
	          X1 .=. X + 11
                 ]),
	mc(X1,Y), 
	mc(Y,R).
```

Note that mc/2 is non-linear in the sense that the head appears more than once on the clause body. For FTCLP non-linear clauses are treated in the same way than linear ones.
'></a>

# Grammar for Constraints #

The relation **clp\_meta/1** takes as its only argument a list (in Prolog format) of type Constraint. The grammar for Constraint is defined as follows:
```
     Constraint  := <Expr> <RelOp>   <Expr> 
     Expr        := <Expr> <ArithOp> <Expr> | <Var> | <Number> 
     RelOp       := .<. | .>. | .=. | .<>. | .=<.| .>=.   
     ArithOp     := + | - | * | /  
```
This is the standard format used in Constraint Logic Programming systems.

# Program annotations #

All relations should be annotated via **tabled/1** directives:

```
:- tabled(foo(_,num)).
```

Tells FTCLP that it should do tabling on foo/2. Moreover, it tells FTCLP that the second argument is a number and hence, it will model it using the theory of integer or real linear arithmetic, depending on one of the horn-prover script options. The `_` symbol tells us that the argument should be modelled with Herbrand logic and use Prolog unification on it.

# Translation to SMTLIB2 format #

Our CLP notation can be translated in a straightforward manner to SMTLIB2 format.
E.g., the above program can be written in SMTLIB2 as follows:

```

(declare-rel prove ())
(declare-rel l (Int Int Int Int))
(declare-rel err (Int Int Int Int))
(declare-var x Int)
(declare-var y Int)
(declare-var i Int)
(declare-var j Int)
(declare-var x1 Int)
(declare-var y1 Int)
(rule (=> (and (not (= x 0)) (= x1 (- x 1)) (= y1 (- y 1)) (l x1 y1 i j)) (l x y i j)))
(rule (=> (and (= x 0) (err x y i j )) (l x y i j)))
(rule (=> (and (= i j) (> y 0)) (err x y i j)))
(rule (=> (and (= x i) (= y j) (l x y i j)) prove))
```

Just to be clear, a parser from SMTLIB2 is not currently implemented.

# View Counterexamples #

Since VCs hold iff its corresponding CLP program has no solutions, if a solution is found then it means that the program is unsafe and the solution corresponds to a counterexample. We print the sequence of executed clauses (separated by the dash **-** symbol) that led to a solution. A clause is denoted by p/n/k where p is the name of the relation functor, n is the arity of the relation, and k is the number of the clause. The number of the clauses are assigned based on the order in which the clauses appear in the program. This is an example of a counterexample:

```
Solution: prove/0/1-p/6/2-l/4/1-l/4/2-err/1/1 
```