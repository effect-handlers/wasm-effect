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


## Syntax

### Exception definitions

````
e ::=
  | ex* @exception et
  | ex* @exception et im
````

### Exception types

````
et ::= q ft
q ::= . | @resumable
````

````
~. := @resumable
~@resumable := .
````

### Reference types

````
rt ::= ...
  | (@exn q (t*)^|q|)
  | (@cont ft) 
````

### Instructions
````
e ::= ...
  | @throw q x
  | @rethrow
  | @resume
  | @resume_throw z
  | @try q ft e* @catch e* @end
  | @handle q ft x e* @else e* @end
````

### Administrative instructions

````
e ::= ...
  | @catch^q_n\{e*\} e* @end
  | @throw a
  | @exn_n a v* a?
  | @cont_n a
````

## Typing

### Contexts

````
C ::= { ..., @exn et* }
````

### Exception types

`````

--------------------------------
C |- q (t1* -> (t2*)^|q|) @ok
`````

### Exception declarations

`````
C |- et @ok
-----------------------------
C |- (@exception et) @ok
`````


### Instructions

`````
C_@exn(x) = q (t1* -> t2*)
---------------------------------------------------
C |- @throw q x : (t3*)^|~q| t1* -> (t4*)^|~q| t2*
`````

`````

----------------------------------------
C |- @rethrow : (@exn q (t*)^|q|) -> t*
`````

`````

---------------------------------------------------
C |- @resume : t1* (@cont t1* -> t2*) -> t2*
`````

`````
C_@exn(x) = q (t3* -> t4*)
---------------------------------------------------
C |- @resume\_throw : t3* (@cont t1* -> t2*) -> t2*
`````

`````
ft = t1* -> t2*
C' = C @with @label = C_@label^|~q|
C',@label t2* |- e1* : ft
C',@label t2* |- e2* : (@exn q (t2*)^|q|) -> t2*
--------------------------------------------------
C |- @try q ft e1* @catch e2* @end : ft
`````

`````
ft = t1* -> t2*
C_@exn(x) = q (t3* -> t4*)
C,@label t2* |- e1* : t1* t3* (@cont t2* t*)^|~q| -> t2*
C,@label t2* |- e2* : t1* (@exn q (t*)^|q|) -> t2*
----------------------------------------------------------------------
C |- @handle q ft x e1* @else e2* @end : t1* (@exn q (t*)^|q|) -> t2*
`````

### Administrative instructions

`````
C' = C @with @label = C_@label^|~q|
S; C',@label t2* |- e1* : t1* -> t2*
S; C',@label t2* |- e2* : (@exn q (t2*)^|q|) -> t2*
-------------------------------------------------------
S; C |- @catch^q_n\{e2*\} e1* @end : t1* -> t2*
`````

`````
S_@exn(a) = q ft
-------------------------
S; C |- @throw a : ft
`````

`````
S_@exn(a1) = q (t1* -> t2*)
(C |- v : t1)*
(S; . |- S_@cont(a2) : (. -> t2*) => (. -> t*))^|q|
----------------------------------------------------
S; C |- @exn_n a1 v* a2^|q| : @exn q (t*)^|q|
`````

`````
S_@cont(a) = E
S; . |- E : (. -> t1*) => (. -> t2*)
---------------------------------------------
S; C |- @cont_n a : @cont (t1* -> t2*)
`````

## Execution

### Module instance

````
M ::= {..., @exn a*}
````

### Store
````
S ::= {..., @exn et*, @cont (E?)*}
````

### Lookup
````
F_@exn(x) := (F_@mod)_@exn(x)
````

### Branch contexts

````
B^0 ::= v* \_ e*
B^(i+1) ::= @label_n\{e*\} B^i @end | @catch^q_m\{e*\} B^(i+1) @end
````

### Throw contexts

````
E_q ::= v* \_ e* | @label_n\{e*\} E_q @end | @catch^{~q}_m\{e*\} E_q @end | @frame_n\{F\} E_q @end
````

### Reduction

````
F; @throw x  -->  F; @throw a
    F_@exn(x) = a
````

````
v^n (@try q ft e1* @catch e2* @end)  -->  @catch^q_m\{e2*\} (@label_m\{.\} v^n e1* @end) @end
    ft = t1^n -> t2^m
````

````
@catch^q_m\{e*\} v* @end  -->  v*
````

````
S; F; @catch^._m{e*} E_.[v^n (@throw a1)] @end  -->  S'; F; @label_m\{.\} (@exn_m a1 v^n) e* @end
    S_@exn(a1) = t1^n -> t2^m
````

````
S; F; @catch^{@resumable}_m {e*} E_@resumable[v^n (@throw a1)] @end  -->  S'; F; @label_m\{.\} (@exn_m a1 v^n a2) e* @end
    S_@exn(a1) = @resumable (t1^n -> t2^m)
    a2 = |S_@cont|
    S' = S @with @cont += E'
    E' = @catch^@resumable_m\{e*\} E_@resumable end
````

````
(@exn_m a1 v^n a2?) @rethrow  -->  v^n (@throw a1) ( (@cont_m a2) @resume )?
````

````
F; v1^n (@exn_m a1 v* a2?) @handle q ft x e1* @else e2* @end  -->  F; @label_k\{.\} v1^n v* (@cont_m a2)? e1* @end
    F_@exn(x) = a1
    ft = t1^n -> t2^k
````

````
F; v1^n (@exn_m a1 v* a2?) @handle q ft x e1* @else e2* @end  -->  F; @label_k\{.\} v1^n (@exn_m a1 v* a2?) e2* @end
    F_@exn(x) =/= a1
    ft = t1^n -> t2^k
````

````
S; v^n (@cont_n a) @resume  -->  S'; E_@resumable[v^n]
    S_cont(a) = E_@resumable
    S' = S @with @cont(a) = .
````

````
S; v^n (@cont_n a) @resume  -->  S; @trap
    S_@cont(a) = .
````

````
S; F; v^m (@cont_n a) (@resume_throw x)  —>  S; F; E[v^m (@throw a’)]
   S_@cont(a) = E
   F_@exn(x) = a'
   S_@exn(a') = q (t1^m -> .)
````

````
S; (@cont_n a) (@resume_throw x)  —>  S; @trap
   S_@cont(a) = .
````


# Conclusion
    
    
[BIB]
