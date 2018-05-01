<!--meta
[INCLUDE=style/acmart]

Title       : Algebraic Effect Handlers for WASM
Author      : Andreas Rossberg et al.
Affiliation : Dfinity
Email       : andreas@dfinity.com
TechReport  : True

Bibliography: wasm-effect.bib
Csl Style   : madoko-numeric
Cite Style  : numeric

[INCLUDE=wasm-style]
[INCLUDE=paper-style]
[INCLUDE=html-style]
[INCLUDE=latex-style]

~ HtmlOnly
[TITLE]
~

~ Abstract
Algebraic effect handlers are a powerful abstraction mechanism 
that can express many complex control-flow mechanisms. 
~

~ TexOnly
[TITLE]
~

~ Updates
v1, 2018-04-03: Initial version.
~
-->

# Introduction     { #sec-intro }

Algebraic effects [@Plotkin:effects] and their extension with
handlers [@Plotkin:handlers;@Pretnar:handlers], are a novel way to
describe many control-flow mechanisms in programming languages. In
general any free monad can be expressed as an effect handler and they
have been used to describe complex control structures such as iterators,
async-await, concurrency, parsers, state, exceptions, etc. (without
needing to extend the compiler or
language) [@Wu:hscope;@Dolan:conc;@Kammar:handlers;@Lindley:liberate;@Leijen:async].

Recently, there are various implementations of algebraic effects, either
embedded in other languages like Haskell [@Wu:hscope;@Kammar:handlers],
Scala [@Brachthauser:effekt], or C [@Leijen:algeffc], or built into a language, like
Eff [@Bauer:algeff], Links [@Lindley:liberate], Frank [@Frank],
Koka [@Leijen:algeff], and Multi-core OCaml [@Dolan:conc;@White:mleff].

# Formalization


Syntax

#### reference types

````
rt ::= ...
  | (exn t*)
  | (cont ft)

q ::= @exn | @eff
````

#### instructions
````
e ::= ...
  | throw x
  | rethrow
  | resume
  | abort
  | try_q ft e* catch e* end
  | handle_q ft x e* else e* end
````

#### administrative instructions
````
e ::= ...
  | catch^q_n e* e* end
  | throw a
  | swallow a
  | @exn_n a v* a?
  | @cont_n a
````

#### exception definitions
````
e ::=
  | ex* exception_q ft
  | ex* exception_q ft im
````

## Typing

#### contexts
````
C ::= { ..., exn (q ft)* }
````

`````
C_exn(x) = ft
---------------------
C |- @throw x : ft
`````

`````

-----------------------------------
C |- rethrow : (exn t*) -> t*
`````

`````

---------------------------------------------------
C |- resume : t1* (cont t1* -> t2*) -> t2*
`````

`````

--------------------------------------------
C |- abort : (cont t1* -> t2*) -> t2*
`````

`````
ft = t1* -> t2*
C^q,label t2* |- e1* : ft
C^q,label t2* |- e2* : (exn t2*) -> t2*
--------------------------------------------------
C |- try_q ft e1* catch e2* end : ft
@where
  C^exn = C
  C^eff = C @with label = .
`````

`````
ft = t1* -> t2*
C_exn(x) = q (t3* -> t4*)
q = exn /\ t'? = .  \/  q = eff /\ 
t'? = (cont t2* t*) C,label t2* |- e1* : t1* t3* 
t'? -> t2* C,label t2* |- e2* : t1* (exn t*) -> t2*
----------------------------------------------------------------------
C |- handle_q ft x e1* else e2* end : t1* (exn t*) -> t2*
`````

`````
S; C^q,label t2* |- e1* : t1* -> t2*
S; C^q,label t2* |- e2* : (exn t2*) -> t2*
-------------------------------------------------------
S; C |- catch^q_n{e2*} e1* end : t1* -> t2*
`````

`````
S_exn(a) = q ft
-------------------------
S; C |- throw a : ft
`````

`````
S_exn(a) = q ft
----------------------------------------
S; C |- swallow a : (exn t*) -> .
`````

`````
S_exn(a) = exn (t1* -> .)
(C |- v : t1)*
-------------------------------------
S; C |- exn_n a v* : exn t*
`````

`````
S_exn(a1) = eff (t1* -> t2*)
(C |- v : t1)*
S; . |- S_cont(a2) : (. -> t2*) => (. -> t*)
----------------------------------------------------
S; C |- exn_n a1 v* a2 : exn t*
`````

`````
S_cont(a) = E_eff
S; . |- E_eff : (. -> t1*) => (. -> t2*)
---------------------------------------------
S; C |- cont_n a : cont (t1* -> t2*)
`````

## Reduction

#### module instance

````
M ::= {..., exn a*}
````

#### store
````
S ::= {..., exn (q ft)*, cont (E?)*}
````

#### lookup
````
F_exn(x) := (F_mod)_exn(x)
````

#### branch contexts

````
B^0 ::= v* \_ e*
B^(i+1) ::= label_n{e*} B^i end | catch^q_m {e*} B^(i+1) end
````

#### throw contexts
````
E_q ::= v* \_ e* | label_n{e*} E_q end | catch^{(~q)}_m {e*} E_q end | frame_n{F} E_q end
````

````
F; throw x  -->  F; throw a
    F_exn(x) = a
````

````
v^n (try_q ft e1* catch e2* end)  -->  catch^q_m{e2*} (label_m{.} v^n e1* end) end
    ft = t1^n -> t2^m
````

````
catch^q_m{e*} v* end  -->  v*
````

````
S; F; catch^{exn}_m{e*} E_exn[v^n (throw a1)] end  -->  S'; F; label_m{.} (exn_m a1 v^n) e* end
    S_exn(a1) = exn (t1^n -> t2^m)
````

````
S; F; catch^{eff}_m {e*} E_eff[v^n (throw a1)] end  -->  S'; F; label_m{.} (exn_m a1 v^n a2) e* end
    S_exn(a1) = eff (t1^n -> t2^m)
    a2 = |S_cont|
    S' = S with cont += E'
    E' = catch^eff_m{e*} E_eff end
````

````
(exn_m a1 v^n a2?) rethrow  -->  v^n (throw a1) ( (cont_m a2) resume )?
````

````
F; v1^n (exn_m a1 v* a2?) handle_q ft x e1* else e2* end  -->  F; label_k{.} v1^n v* (cont_m a2)? e1* end
    F_exn(x) = a1
    ft = t1^n -> t2^k
````

````
F; v1^n (exn_m a1 v* a2?) handle_q ft x e1* else e2* end  -->  F; label_k{.} v1^n (exn_m a1 v* a2?) e2* end
    F_exn(x) =/= a1
    ft = t1^n -> t2^k
````

````
S; v^n (cont_n a) resume  -->  S'; E_eff[v^n]
    S_cont(a) = E_eff
    S' = S with cont(a) = .
````

````
S; v^n (cont_n a) resume  -->  S; trap
    S_cont(a) = .
````

````
S; (cont_n a) abort  -->  S'; catch^exn_n{swallow a} E_eff[(throw a')] end
    S_cont(a) = E_eff
    a' = |S_exn|
    S' = S @with exn += exn (. -> .) with cont(a) = .
````

````
S; (cont_n a) abort  -->  S; trap
    S_cont(a) = .
````

````
(exn_n a1 v^* a2?) (swallow a1)  -->  .
````

````
(exn_n a1 v^* a2?) (swallow a3)  -->  (exn_n a1 v^* a2?) rethrow
     a1 =/= a3
````


# Conclusion
    
    
[BIB]
