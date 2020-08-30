
JSON-LD-LOGIC: JSON representation of first-order logic formulas
=================================================================

*Draft version 0.7: looking for comments and recommendations*


tanel.tammet@gmail.com

with contributions from schulz@eprover.org and geoff@cs.miami.edu

Contents
--------

* [Goals and summary](#Goals-and-summary)
* [Initial conversion and proof examples](#Initial-conversion-and-proof-examples)
* [Two layers of JSON-LD-LOGIC](#Two-layers-of-JSON-LD-LOGIC)
* [Core fragment](#Core-fragment)
  * [Primitives in the core fragment](#Primitives-in-the-core-fragment)
  * [Terms and atoms in the core fragment](#Terms-and-atoms-in-the-core-fragment)
  * [Formulas in the core fragment](#Formulas-in-the-core-fragment)
  * [Equality predicate](#Equality-predicate)
  * [Formula list](#Formula-list)
  * [Metainformation and roles in the core fragment](#Metainformation-and-roles-in-the-core-fragment)
  * [Included files](#Included-files)
* [Full JSON-LD-LOGIC](#Full-JSON-LD-LOGIC)
  * [JSON objects aka maps](#JSON-objects-aka-maps)
  * [Ordinary and distinct symbols](#Ordinary-and-distinct-symbols)
  * [Datatypes and typed symbols](#Datatypes-and-typed-symbols)
  * [Missing id and blank nodes](#Missing-id-and-blank-nodes)
  * [Introducing logic to JSON-LD](#Introducing-logic-to-JSON-LD)  
  * [ans and question](#ans-and-question)
  * [Uniqueness of Skolem functions](#Uniqueness-of-Skolem-functions)
  * [Convenience connectives and predicates](#Convenience-connectives-and-predicates)
  * [Multiple values and the context and namespaces and the base type](#Multiple-values-and-the-context-and-namespaces-and-the-base-type)
  * [Nested objects aka maps](#Nested-objects-aka-maps)
  * [Graphs and quads with the narc predicate](#Graphs-and-quads-with-the-narc-predicate)
  * [Numbers and arithmetic](#Numbers-and-arithmetic)
  * [Lists and list functions](#Lists-and-list-functions)
  * [Distinct symbols as strings](#Distinct-symbols-as-strings)


Goals and summary
------------------

JSON-LD-LOGIC is a simple JSON syntax for writing
first order logic [FOL](https://en.wikipedia.org/wiki/First-order_logic "FOL") 
formulas in a machine-processable manner suitable for automated reasoners.

The main goals of JSON-LD-LOGIC are:
* Providing a simple format for the programmatic management of logical problems and proofs.
* Compatibility with the [TPTP](http://tptp.org "TPTP") language used by 
  most high-end full FOL reasoners, as described in the  
  [TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml#ProblemPresentation).
* Compatibility with [JSON-LD](https://json-ld.org/), 
  see the [W3C recommendation](https://www.w3.org/TR/json-ld11/). 

The conversion and proof examples in this document have been created using 
the [gkc](https://github.com/tammet/gkc/)
reasoner and toolkit implementing both JSON-LD-LOGIC and TPTP languages.
Gkc is able to find and present proofs, answer questions,
clausify JSON-LD-LOGIC expressions and convert them to or from the TPTP language. 
Check out a [web playground](http://logictools.org/json.html) for JSON-LD-LOGIC, 
running gkc in the browser using WASM. 

A simple unsatisfiable example using core JSON-LD-LOGIC stating some facts, rules and finally a question in the negated form:

    [
      ["sibling","john","mike"],
      ["sibling","john","pete"],
      [["sibling","?:X","?:Y"], "=>", ["sibling","?:Y","?:X"]],
      [[["sibling","?:X","?:Y"], "&",  ["sibling","?:Y","?:Z"]], "=>", ["sibling","?:X","?:Z"]],
      ["~sibling","mike","pete"]
    ]

Another example using full JSON-LD-LOGIC demonstrating the combination of a more conventional logic syntax
with the JSON-LD syntax, using both explicit quantifiers and free variables, 
convenience operators and indicating a question to be answered:

    [
      {
        "@context": {    
          "@vocab":"http://foo.org/"
        },
        "@id":"pete", 
        "@type": "male",
        "father":"john",
        "son": ["mark","michael"],
        "@logic": [
          "forall",["X","Y"],
           [[{"@id":"X","son":"Y"}, "&", {"@id":"X","@type":"male"}], 
            "=>", 
            {"@id":"Y","father":"X"}]
        ]
      },   

      [
       "if", 
         {"@id":"?:X","http://foo.org/father":"?:Y"}, 
         {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
       "then", 
         {"@id":"?:X","grandfather":"?:Z"}        
      ],

      {"@question": {"@id":"?:X","grandfather":"john"}}
    ]


The main features of the syntax are:

* Terms, atoms and logical formulas are represented as JSON lists with predicate/function symbols 
  in the first position (prefix form) like `["father","john","pete"]` or
  `["forall",["X"],[["is_father","X"],"=>",["exists",["Y"],["father","X","Y"]]`.
* JSON-LD semantics in RDF is represented by the`"$arc"` predicate for triplets
  like `["$arc","pete","father","john"]` and an
  `"$narc"` predicate for named triplets aka quads like `["$narc","pete","father","john","eveknows"]`.
* JSON objects aka maps like  
  `{"son":"pete", "age":30, "@logic": [{"@id":"?:X","son":"?:Y"},"=>",{"@id":"?:Y","parent":"?:X"}}` 
  are interpreted as logical statements and use the special `"@logic"` key for inserting full *FOL*.
* JSON strings can represent ordinary constant/function/predicate symbols like `"foo"`,
  bound variables likes `"X"`, free variables like `"?:X"`, blank nodes like `"_:b0"` 
  and distinct symbols like `"#:bar"`, the latter three using special JSON-LD-style *prefixes*. 
* Arithmetic, a list type, string operations on distinct symbols and the semantics for `null` are defined.  
* JSON lists in JSON-LD like `{"@list":["a",4]}` are translated to nested typed terms
  using the `"$list"` and `"$nil"` functions: `["$list","a",["$list",4,"$nil"]]`.

The semantics of most JSFOL constructions stems directly from the semantics of the 
corresponding TPTP constructions. The semantics of *lists*, *null*, 
*if ... then ...* etc not present in TPTP are presented explicitly in the current document
via translations to the TPTP constructions.

Despite following the RDF semantics of JSON-LD, JSON-LD-LOGIC does not include 
RDF or RDFS axioms like the transitivity of `subClassOf`. Such axioms can
be written in the JSON-LD-LOGIC or TPTP language and explicitly added to the formula to be proved.
The [TPTP axiom collection](http://www.tptp.org/cgi-bin/SeeTPTP?Category=Axioms)
contains versions of RDF, RDFS and OWL axioms as `SWB*.ax` files. For an example, see the
[RDFS axioms](http://www.tptp.org/cgi-bin/SeeTPTP?Category=Axioms&File=SWB003+0.ax).
Before using these, one should modify their function and constant names to the
values preferred: in particular, `iext` should be renamed `$arc`. It is also possible
to avoid these axioms altogether and devise axioms suitable for the specific task.
The examples in the current document are self-contained and do not use any
axioms in addition to the ones explicitly present in the examples.

The current version of the document does not cover all the advanced features of the
W3C JSON-LD recommendation like @container, @direction, @index, @prefix, @propagate,
@protected, @language, @reverse. The document does cover core features like
@id, @context, @base, @graph, @list, @type (the latter only outside @context: see the
discussion in the *Datatypes and typed symbols* chapter).



Initial conversion and proof examples
---------------------------------------

JSON-LD-LOGIC defines direct JSON <-> TPTP conversion for the first order fragment
(FOF and CNF sublanguages) of TPTP along with a limited and extended subset of 
the typed sublanguage TFF with arithmetic.

Due to the goals of the current document it contains a number of JSON-LD-LOGIC 
examples together with their translation to the TPTP language and a proof found.

The first example above can be converted to TPTP as:

    fof(frm_1,axiom,sibling(john,mike)).
    fof(frm_2,axiom,sibling(john,pete)).
    fof(frm_3,axiom,(! [X,Y] : (sibling(X,Y) => sibling(Y,X)))).
    fof(frm_4,axiom,(! [X,Y,Z] : ((sibling(X,Y) & sibling(Y,Z)) => sibling(X,Z)))).
    fof(frm_5,axiom,~sibling(mike,pete)).

The TPTP syntax wraps formulas into the `fof(name,role,formula).`
construction terminated by the period. 
The exclamation mark `!` denotes *universal quantifier* (for all)
and the question mark `?` denotes *existential quantifier* (exists).
Variables have to start with the uppercase letter, non-variable symbols
with a lowercase letter, underscore `_` or dollar `$`. The logical
operator `~` denotes *negation*,`&` denotes *conjunction* (and), `|` denotes *disjunction* , `=>` denotes *implication*, 
`<=>` denotes *equivalence*. *Equality* is denoted by the infix `=` predicate along with the negation of equality `!=`.
The TPTP use of quotes and double quotes surrounding some symbols will be described later.

We obtain the following refutation proof in json:

    {"result": "proof found",

    "answers": [
    {
    "proof":
    [
    [1, ["in", "frm_4"], [["-sibling","?:X","?:Y"], ["-sibling","?:Z","?:X"], 
                          ["sibling","?:Z","?:Y"]]],
    [2, ["in", "frm_1"], [["sibling","john","mike"]]],
    [3, ["mp", 1, 2], [["-sibling","?:X","john"], ["sibling","?:X","mike"]]],
    [4, ["in", "frm_3"], [["-sibling","?:X","?:Y"], ["sibling","?:Y","?:X"]]],
    [5, ["in", "frm_2"], [["sibling","john","pete"]]],
    [6, ["mp", 4, 5], [["sibling","pete","john"]]],
    [7, ["mp", 3, 6], [["sibling","pete","mike"]]],
    [8, ["in", "frm_5"], [["-sibling","mike","pete"]]],
    [9, ["mp", 7, 4, 8], false]
    ]}
    ]}

The proofs in this document are json objects with two keys:

* `result` indicates whether the proof was found,
* `answers` contains a list of all answers and proofs for these answers. 
  Here we did not ask to find a concrete person, so we have only 
  a single answer containing just a `proof` key indicating a list with the
  numbered steps of the proof found:
  each step has a form `[formula id, [derivation rule, source 1, ..., source N], formula]` 
  where the *formula* either stems from an input fact / rule or is derived
  from the indicated sources. A source may be represented as a formula id or a list with 
  the formula id as a first element and the rest indicating an exact position in the source formula.

In this document all the formulas in the proofs are *clauses*: lists of positive or negative
atoms treated as a disjunction (or). Negation is prefixed as a minus sign `-` to the predicate:
`["-sibling","mike","pete"]` means the same as `["not" ["sibling","mike","pete"]]`.
Strings prefixed by `?:` like `?:X` are *free variables* implicitly assumed to be quantified by *forall*.
`["in","frm_N" ]` means that the clause stems from the *N*th fact/rule given as input.
Observe that one input rule may generate a number of different clauses and
use *Skolemization* to create new function and constant symbols.

`["mp", 1, 2]`  means that this clause was derived by the 
[resolution rule](https://en.wikipedia.org/wiki/Resolution_(logic)) (i.e. generalized modus ponens)
from the earlier sources 1 and 2 present in the proof. More concretely, the first literals of both were cut off and
the remaining parts were concatenated under the unifying substitution. 
`["mp", 7, 4, 8]` means that the clause was first 
derived from clauses 7 and 4 and then the clause 8 was used for an additional
simplifying resolution or equality conversion step. There are also other derivation operators like equality replacement,
simplification etc, denoted differently from `"mp"`.

The second example above can be converted to TPTP as:
    
    fof(frm_1,axiom,
      ((! [X,Y] : 
         (($arc(X,'http://foo.org/son',Y) & 
           $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)) 
          => 
          $arc(Y,'http://foo.org/father',X))) & 

         ($arc(pete,'http://foo.org/son',michael) & 
           ($arc(pete,'http://foo.org/son',mark) & 
            ($arc(pete,'http://foo.org/father',john) & 
             $arc(pete,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)))))).

    fof(frm_2,axiom,
      (! [X,Y,Z] : 
         (($arc(Y,'http://foo.org/father',Z) & $arc(X,'http://foo.org/father',Y)) 
          => 
          $arc(X,grandfather,Z)))).

    fof(frm_3,negated_conjecture,(! [X] : ($arc(X,grandfather,john) => $ans(X)))).

giving the following refutation proof in json:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","michael"]],
    "proof":
    [
    [1, ["in", "frm_1"], [["-$arc","?:X","http://foo.org/son","?:Y"], 
                          ["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"],
                          ["$arc","?:Y","http://foo.org/father","?:X"]]],
    [2, ["in", "frm_1"], [["$arc","pete","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"]]],
    [3, ["mp", [1,1], 2], [["-$arc","pete","http://foo.org/son","?:X"], 
                           ["$arc","?:X","http://foo.org/father","pete"]]],
    [4, ["in", "frm_1"], [["$arc","pete","http://foo.org/son","michael"]]],
    [5, ["mp", 3, 4], [["$arc","michael","http://foo.org/father","pete"]]],
    [6, ["in", "frm_2"], [["-$arc","?:X","http://foo.org/father","?:Y"], 
                          ["-$arc","?:Z","http://foo.org/father","?:X"], 
                          ["$arc","?:Z","grandfather","?:Y"]]],
    [7, ["in", "frm_1"], [["$arc","pete","http://foo.org/father","john"]]],
    [8, ["mp", 6, 7], [["-$arc","?:X","http://foo.org/father","pete"], 
                       ["$arc","?:X","grandfather","john"]]],
    [9, ["mp", 5, 8], [["$arc","michael","grandfather","john"]]],
    [10, ["in", "frm_3"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [11, ["mp", 9, 10], [["$ans","michael"]]]
    ]}
    ]}


Two layers of JSON-LD-LOGIC
-----------------------------

`Core fragment` specifies minimal JSON syntax for writing logic formulas 
which can be translated to the TPTP FOF and CNF sublanguages. All the TPTP FOF and CNF 
formulas are directly convertible to the core JSON-LD-LOGIC and vice versa. 
Both of these TPTP sublanguages cover standard first order logic with functions and 
the equality predicate.

`Full language` adds compatibility with JSON-LD along with support for
an arithmetic-plus-distinct-symbols part of the TPTP typed sublanguage TFF plus
an additional list type, several additional pre-defined functions, 
convenience functions and operators. Full JSON-LD-LOGIC can be 
converted to the core fragment, except for the typed part using arithmetic, lists
and distinct symbols: in the core fragment the functions and predicates on
typed values are left as is, i.e. not evaluated.

We will first present the core fragment and thereafter the full language.


Core fragment
-------------

The core fragment is essentially a JSON representation of the combined TPTP
FOF (first order formula) and CNF (clause normal form) sublanguages by
allowing the introduction of free variables in any formulas and not
requiring the formulas to be embedded in special `fof` or `cnf` terms.

Arithmetic, distinct symbols, lists, null and JSON-LD constructions are
not included in the core fragment.

Core fragment constructions can be viewed as falling into one of five categories,
from bottom to top:

* `Primitives` are constants, function/predicate symbols and variables 
   represented by JSON strings.
* `Terms and atoms` are built of primitives and terms as nested JSON lists. 
   A `literal` is either an atom or a negated atom.
* `Formulas` are built of literals using logical constructors like "&", "exists", etc
   using nested JSON lists. JSON objects aka maps are used for adding meta-information to formulas.
* `Formula lists` are JSON lists of fully formed, independent standalone formulas.


### Primitives in the core fragment


* JSON strings have multiple uses covered in the following list:

    * Special predefined strings in TPTP are used for constructing formulae.
      The full list is:
      `"~", "|", "&",  "<=>", "=>", "<=", "<~>", "~|", "~&", "@", 
      "forall", "exists"`.
      Pre-defined equality and inequality predicates are `"="` and `"!="`

    * Strings bound by a quantifier in JSON stand for corresponding variables in
      FOL and TPTP. A bound variable *must* start with an upper case letter.     
      In the following example all the variables `X1, X2, U` 
      are existentially quantified:        

      `["exists",["X1","X2","U"],["p","X1","X2","U"]]`
   
    * Strings starting with a question mark and colon like  `"?:X1"`
      and *not bound by a quantifier* 
      stand for free variables in FOL and are assumed to be universally quantified
      at the top level of a formula in TPTP, except in the formulas with the role 
      *conjecture*, as explained later. Examples: 

      `"?:X", "?:Y1", "?:Some car"`    

    * Strings prefixed by `"~"` and located in the first position of a list in a
      formula context construct a negated atom led by a symbol
      constructed from the rest of the string. 
      For example, the arguments of the following "&" are equivalent:

      `[["~p", 1], "&", ["~", ["p",1]]`
      
    * Strings starting with a hash sign and colon like  `"#:foo"` are distinct
      symbols used in the full language and should not be used in the core fragment.

    * Strings starting with an underscore sign and colon like  `"_:bar"` are 
      blank nodes of JSON-LD and should not be used in the core fragment.   
      
    * Strings without any of the previously described prefixes `?:`, `#:`, `_:`, `~` 
      stand for ordinary predicate, function or constant symbols, unless
      they are one of the predefined strings in the full language 
      like `"&", "$sum", "$is_real", "+"` etc. Examples:

      `"p", "Foo", "'bar' 23", "_here", "++something", "1 to 3"`

* `true` and `false` in JSON stand for the corresponding logical values
  in TPTP: `$true` and `$false`.

* Other primitive JSON datatypes - numbers and null - are a part
  of the full language and should not be be used in the core fragment.

* An empty JSON list `[]` stands for a symbol `$nil` in the full language
  and should not be used in the core fragment.

* Objects (aka maps) are a part of the full language and should be used
  only for providing meta-information for top level formulas in the core fragment. 
   
   
### Terms and atoms in the core fragment


Terms and atoms are represented as JSON lists, using the prefix form analogous to
lisp s-expressions: the first element of a list stands for a predicate or a function symbol.

Example:

    ["foo","bar","?:Y","a",["f",["f","b"]]]

stands for a TPTP term or atom 

    foo(bar,Y,a,f(f(b)))

where `Y` is a free variable.

Important aspects of the TPTP conversion:

* The TPTP FOF sublanguage does not allow free variables, whereas the clausal CNF sublanguage allows 
  only free variables. JSON-LD-LOGIC merges the FOF and CNF sublanguages into one, allowing
  free variables in both simple clauses and arbitrary formulas.

* TPTP requires symbols to either 

  * (a) consist of alphanumerics, underscore `_` and dollar `$` and not start with a capital letter,

  * (b) or to be surrounded by single quotes like `'Here is "John"'`. 
  
  JSON-LD-LOGIC treats all non-special strings not prefixed by `?:`, `#:` or `_:`
  and not bound by a quantifier as ordinary symbols. Thus the JSON-LD-LOGIC symbols which do not
  satisfy the TPTP requirement (a) are converted to TPTP by surrounding them with single quotes
  and adding a backslash character in front of all internal quote characters. The TPTP symbols surrounded
  by quotes are converted to JSON-LD-LOGIC by removing the surrounding quotes and adding 
  a backslash character in front of internal double quotes.

JSON-LD-LOGIC does not require that predicate or function symbols have a unique arity, although
applications may pose such restrictions.

The first element of an atom or a function term should be a string, except for
atoms constructed by infix equality `"="` or inequality `"!="`. 

### Formulas in the core fragment


Formulas are built from booleans, literals and formulas using ordinary logical connectives,
used in the infix form in the core fragment.

The following connectives are predefined, whereas `"~"`
may occur as a first element of a two-element list and all the other connectives 
may only occur between elements of a list (infix form). A list may not contain different 
connectives: instead of

  `["a","&","b","=>","c"]`

  one must use
  
  `[["a","&","b"],"=>","c"]`

The binary connectives are left associative. For example,

  `["a","=>","b","=>","c"]`

is interpreted as

  `[["a","=>","b"],"=>","c"]` 

The connectives are:  

* `"~"` stands for negation and takes a single argument. Note that as we have said before,
    `["~p",1]` stands for `["~",["p",1]]`
     
* Based on the TPTP language, the following connectives are available, all of them
   used only in the infix form:

    * infix `"|"` for disjunction,
    * infix `"&"` for conjunction, 
    * infix `"<=>"` for equivalence, 
    * infix `"=>"` for implication, 
    * infix `"<="` for reverse implication, 
    * infix `"<~>"` for non-equivalence (XOR), 
    * infix `"~|"` for negated disjunction (NOR), 
    * infix `"~&"` for negated conjunction (NAND), 
    * infix `"@"` for application, used mainly in the higher-order context in TPTP.

* The quantifiers `"exists"` and `"forall"` 
  are represented as a list of three elements: quantifier, list of variables, formula.
  Examples:

  `["exists",["X"],["p","X"]]`

  `["forall",["X","Y2"],["p","Y2","X"]]`

There is a simplified alternative syntax for clauses (disjunctions of literals):
a list `[L1, ... ,LN]` occurring in the  top-level formula formula list (described next)
is treated as a disjunction of elements `[L1,"|", ... ,"|",LN]` if

* `L1` is a list,

* no element `Li` in the list is a connective or a positive or negative equality predicate.



### Equality predicate


Predicates `"="` and `"!="` stand for equality and inequality and can occur
only in the middle of a three-element list (infix form), like

  `["a","=","b"]`
  `["?:X","!=",["foo","?:X"]]`

The following example uses equality:

    [
      [["father","john"],"=","pete"],
      [["father","mike"],"=","pete"],
      [["mother","john"],"=","eve"],
      [["mother","mike"],"=","eve"],
      [["father","pete"],"=","mark"],
      [["mother","eve"],"=","mary"],
      ["grandfather",["father",["father","?:0"]],"?:0"],
      ["grandfather",["father",["mother","?:0"]],"?:0"],
      ["~grandfather","mark","?:0"]
    ]

Conversion to the TPTP form is

    fof(frm_1,axiom,(father(john) = pete)).
    fof(frm_2,axiom,(father(mike) = pete)).
    fof(frm_3,axiom,(mother(john) = eve)).
    fof(frm_4,axiom,(mother(mike) = eve)).
    fof(frm_5,axiom,(father(pete) = mark)).
    fof(frm_6,axiom,(mother(eve) = mary)).
    fof(frm_7,axiom,(! [X0] : grandfather(father(father(X0)),X0))).
    fof(frm_8,axiom,(! [X0] : grandfather(father(mother(X0)),X0))).
    fof(frm_9,axiom,(! [X0] : ~grandfather(mark,X0))).

The proof of unsatisfiability we get uses equality. 
Notice that the clauses in the proof do not strictly correspond
to the core fragment, since (a) equality is written in prefix form
as allowed only in the full language and (b) minus sign is
used instead of tilde to indicate negation, also allowed
in the full language.

    {"result": "proof found",

    "answers": [
    {
    "proof":
    [
    [1, ["in", "frm_2"], [["=",["father","mike"],"pete"]]],
    [2, ["in", "frm_7"], [["grandfather",["father",["father","?:X"]],"?:X"]]],
    [3, ["=", 1, [2,0,2]], [["grandfather",["father","pete"],"mike"]]],
    [4, ["in", "frm_5"], [["=",["father","pete"],"mark"]]],
    [5, ["simp", 3, 4], [["grandfather","mark","mike"]]],
    [6, ["in", "frm_9"], [["-grandfather","mark","?:X"]]],
    [7, ["mp", 5, 6], false]
    ]}
    ]}



### Formula list


The JSON-LD-LOGIC core fragment document must be a list of formulas like
the following example:
 
    [      
      ["sibling","john","mike"],
      ["sibling","john","pete"],
      [["sibling","?:X","?:Y"], "=>", ["sibling","?:Y","?:X"]],
      [["is_father","?:X"], "<=>", ["exists",["Y"],["father","?:X","Y"]]]      
    ]

The elements of the formula list are translated as a conjunction
of the elements, with the following differences from a simple conjunction:

Each formula in the list containing free variables
is translated by binding the free variables in this formula by
a "forall" quantifier at the top level of this formula, except for
the formulas with the *conjecture* role (to be described later).
The previous example is thus equivalent to the following *single formula*:

    [
      ["sibling","john","mike"], "&", 
      ["sibling","john","pete"], "&",
      ["forall",["X","Y"], [["sibling","X","Y"], "=>", ["sibling","Y","X"]]], "&",
      ["forall",["X"], [["is_father","X"], "<=>", ["exists",["Y"],["father","X","Y"]]]]      
    ]
 
Notice that 

* The existentially quantified `"Y"` variable in the last formula is dependent on
  the leading `"X"` variable of the same formula. The last formula can be converted
  to a clause normal form with the help of *Skolemization* introducing a new blank node
  `'_:sk1'` as a function symbol like

  `cnf(frm_4,axiom,(~is_father(X0) | father(X0,'_:sk1'(X0)))).`

  `cnf(frm_4,axiom,(is_father(X0) | ~father(X0,X1))).`

* The free variables `"?:X"` in the last two formulas of the example before conversion 
  are distinct from each other.
  
   
### Metainformation and roles in the core fragment


Metainformation is represented using JSON objects and can be attached to formulas in
top level formula lists. Example:

    {
     "@name":"example formula",    
     "@role", "axiom",
     "@logic": ["p",1,"a"] 
    }
 
 
The following predefined keys correspond to the TPTP positional metainformation values:

* `"@name"` value is a formula name without a logical meaning,
  useful for increasing the readability of proofs.

* `"@role"` value should be normally either "axiom", "assumption", "conjecture" or a
  "negated_conjecture". It is used for annotating formulas for their intended use.
  A longer description and more options are given below.  

* `"@logic"` for the JSON-LD-LOGIC formula with the logical content as described before.

All of these key/value pairs are optional. If `"@logic"` is not present, the object should
be ignored by the application.

In case a formula list contains a JSON object, then

* If `"@name"` is not present, the system may optionally construct a new name.

* If `"@role"` is not present, the corresponding value is assumed to be "axiom". 

The special `"conjecture"` role value in TPTP forces the content to be negated when asking
for unsatisfiability of a formula list. 

We bring two equivalent example formulas, where in the first case "conjecture" is used with
the positive `["p","a"]` and in the second "negated_conjecture" with the negative `["~p","a"]`:

    [
      ["p","a"],
      ["p","b"],
      {"@role": "conjecture", "@logic": ["p","a"]}
    ]

represents a provable formula `(p(a) & p(b)) => p(a)`, the negation of which
is equivalent to an unsatisfiable `p(a) & p(b) & ~p(a)`.

The following example

    [
      ["p","a"],
      ["p","b"],
      {"@role": "negated_conjecture", "@logic": ["~p","a"]}
    ]

also represents an unsatisfiable formula `p(a) & p(b) & ~p(a)`.

As an exception the *free variables* are not implicitly bound by a universal quantifier
(*for all*) in the formula with a *conjecture* role: instead, they are implicitly
bound by the existential quantifier (*exists*): this corresponds better to the intuitive 
understanding of free variables in the conjecture, especially in the light
of the `$ans`predicate and the `@question` key in the full language. For example,

    [
      ["p","a"],
      ["p","b"],
      {"@role": "conjecture", "@logic": ["p","?:X"]}
    ]

is equivalent to

    [
      ["p","a"],
      ["p","b"],
      {"@role": "conjecture", "@logic": ["exists",["X"],["p","X"]]}
    ]

which is equivalent to

    [
      ["p","a"],
      ["p","b"],
      {"@role": "negated_conjecture", "@logic": ["forall",["X"],["~p","X"]]}
    ]


For other *role* values we cite "The Formulae Section" of 
[TPTP technical manual](http://tptp.org/TPTP/TR/TPTPTR.shtml): 
the role gives the user semantics of the formula, one of `axiom, hypothesis, definition,
assumption, lemma, theorem, corollary, conjecture, negated_conjecture, plain, type, and unknown.

Other key values except these of the predefined keys above will have a JSON-LD-defined
meaning in the full language, but not in the core fragment. 
Additional metainformation may be attached to formulas using keys
with `@` prefix and not defined by JSON-LD, like `"@comment"`: such keys are ignored by the 
full language, but may be given a specific meaning by applications.


### Included files

A document may include other files/documents, treated by appending the included formula list and the formula
list of the main document. Analogously to TPTP, JSON-LD-LOGIC uses a special object/map with the 
form `{"@include":filename}` in the top formula list, interpreted as an *include command*:

    [
      {"@include":"foo.js"},
      ...     
    ]

will cause the file "foo.js" to be included. The file is searched for in places specified by TPTP as follows: 
include files with relative path names are expected to be found either under the directory of the current file,
or if not found there then under the directory specified in the $TPTP environment variable.

The languages supported by the include command are implementation specific: other languages than JSON-LD-LOGIC
or TPTP might be allowed to be imported.



Full JSON-LD-LOGIC
------------------

The main additional features of the full language above the core fragment are:

* JSON-LD objects aka maps are interpreted as logic and can be intermingled with the
  core fragment constructions. 

* The semantics in RDF is represented by a special $arc predicate for triplets
  like `["$arc","pete","father","john"]` and an
  $narc predicate for named triplets aka quads, like `["$narc","pete","father","john","eveknows"]`.

* Numeric arithmetic, distinct symbols, string operations and a list type are provided.  

* JSON lists in JSON-LD like `{"@list":["a",4]}` are translated to nested typed terms
  using the `"$list"` and `"$nil"` functions, like `["$list","a",["$list",4,$nil]]`.

* Several convenience operators are introduced.


### JSON objects aka maps

JSON objects are interpreted according to the 
(RDF)[https://en.wikipedia.org/wiki/Resource_Description_Framework] 
semantics of JSON-LD, with modifications for the interpretation of 
JSON-LD lists and `null` as described later in this section.

The core principle is that each RDF triple `subject, property, value` is 
converted to an atom `["$arc",subject,property,value]` indicating that
there is an *arc* with the label *property* from the *subject* to *value*.

The `"$arc"` symbol has no special semantics except that it is used for
translating the triplets.

Consider the following JSON-LD example:

    [
      {"@id":"pete", "father":"john", "age":40},
      {"@id":"mark", "father":"pete", "age":10}
    ]

This formula list is a valid JSON-LD-LOGIC list and can be converted to TPTP as

    fof(frm_1,axiom,($arc(pete,age,40) & $arc(pete,father,john))).
    fof(frm_2,axiom,($arc(mark,age,10) & $arc(mark,father,pete))).

We say *can be converted* to indicate that the exact form of the conversion
is implementation-specific, but the logical semantics should be exactly
as indicated in the TPTP conversion.


### Ordinary and distinct symbols

Importantly, both the *subject*, *property* and *value* strings 
like "john" and "pete" in the example above are treated as ordinary
symbols in logic, i.e. "john" and "pete" are not automatically considered
to be distinct: they both could be potentially equal to some "John Pete Smith",
for example.

JSON-LD-LOGIC uses a special prefix `#:` to indicate that a symbol
is not equal to any other syntactically different symbols with the `#:`
prefix and not equal to any numbers or lists.

For example, both `["#:pete","=","#:john"]` and `["#:pete","=",2]`
must be evaluated to *false*, while `["pete","=","#:john"]` and `["pete","=",2]`
are not evaluated to *false*.

The *distinct symbols* can be seen as an extension of the *string* type; 
several string functions defined by JSON-LD-LOGIC can be
computed on distinct symbols.

The distinct symbols of JSON-LD-LOGIC are translated to the distinct symbols of the
TPTP language with the same semantics as described above. The distinct
symbols of TPTP are surrounded by double quotes.

The following example

    [
      {"@id":"#:pete", "father":"#:john", "age":40},
      {"@id":"mark", "father":"#:pete", "age":10}
    ]

 can be converted to TPTP containing the distinct symbols `"pete"` and `"john"`
 as well as ordinary symbols `mark` , `father` and `age`:

    fof(frm_1,axiom,($arc("pete",age,40) & $arc("pete",father,"john"))).
    fof(frm_2,axiom,($arc(mark,age,10) & $arc(mark,father,"pete"))).

We note that the TPTP language uses the single quotes like `'John Smith'`
to denote ordinary symbols containing characters which are neither alphanumeric nor
`_` or `$`.

The motivation for treating JSON strings as ordinary symbols by default and using
a special prefix for distinct symbols (strings, if so called), and not vice versa,
stems from the following observations:

* The vast majority of symbols occurring in the TPTP library with tens of thousands
  of problems are ordinary symbols, with distinct symbols being a rarity.

* The difference between ordinary and distinct symbols in logic only appears in
  the presence of the equality predicate or other typed objects in the context of
  special string/arithmetic/list operations: otherwise there is no difference
  in the logical meaning.


### Datatypes and typed symbols

JSON-LD-LOGIC does not currently specify the interpretation of `@type` used as a datatype
in a JSON-LD context like 

    "@context":{
      "foo":{
        "@id":"http://bar.org",
        "@type":"http://www.w3.org/2001/XMLSchema#dateTime"
      }
    }

The special types of strings like
dates etc could be interpreted, for example, by prepending `^^typename` to a 
typed string like 

    "2011-04-09T20:00:00Z^^http://www.w3.org/2001/XMLSchema#dateTime"

and considering such typed strings to be unequal to syntactically
different typed symbols. Alternatively, such types could be encoded using
a special datatype-assigning function symbol like

    ["$datatype", "2011-04-09T20:00:00Z", "http://www.w3.org/2001/XMLSchema#dateTime"]

The suitable way of converting typed strings to TPTP may be determined in the
later revisions.

JSON-LD-LOGIC does, however, treat numeric JSON values as well as lists
indicated by the `"@list"` key (described later) and distinct symbols analogous
to strings as built-in types.

The special URI type in JSON-LD written as `"@type": "@id"` is ignored by
JSON-LD-LOGIC, since it interprets JSON strings as ordinary logical symbols by default.

### Missing id and blank nodes

In case a JSON object does not contain an `"@id"` key, a new *blank node* symbol 
not occurring anywhere else in the document must be automatically generated 
for the *object*.

The following example 
    
    [
      {"father":"john", "age":40},
      {"father":"pete", "age":10}
    ]

can be converted to TPTP as

    fof(frm_1,axiom,($arc('_:crtd_1',age,40) & $arc('_:crtd_1',father,john))).
    fof(frm_2,axiom,($arc('_:crtd_2',age,10) & $arc('_:crtd_2',father,pete))).

where `_:crtd_1` and `_:crtd_2` are new symbols not occurring anywhere else,
corresponding to the *blank nodes* of JSON-LD and RDF. The blank nodes
behave similarly to the outermost existentially quantified variables, converted to
the *skolem constants* by the clausification algorithms. 

The following JSON-LD example containing *blank nodes* 
    
    [
      {"@id":"_:pete", "father":"john"},
      {"@id":"_:pete", "age":10}
    ]

can be converted to TPTP as

    fof(frm_1,axiom,$arc('_:pete',father,john)).
    fof(frm_2,axiom,$arc('_:pete',age,10)).

The interpretation of blank nodes is the same as in JSON-LD: 
syntactically identical blank nodes in different documents/files should be treated
as different symbols. This could be achieved by renaming all the blank nodes
in the documents to unique values in the scope of a document/file whenever several
documents/files are merged in some way.


## Introducing logic to JSON-LD 

The value of the `"@logic"` key in an object/map is treated as a logical formula,
using either the core fragment described before or the full JSON-LD-LOGIC language.

Both the value of the `"@logic"` key and values of ordinary property keys may
contain free variables. The free variables occurring in the value of the `"@logic"` key 
and/or elsewhere in the same top-level list element have the whole element as its scope.

The following example demonstrates a simple use of `"@logic"` along with the
convenience operator `if ... then ...` and the `"@question"`
key described in the next section:

    [
      {"@id":"pete", "father":"john"},
      {"@id":"mark", "father":"pete"},
      {"@name":"gfrule",
      "@logic": ["if", 
                    ["$arc","?:X","father","?:Y"],
                    ["$arc","?:Y","father","?:Z"], 
                  "then", 
                    ["$arc","?:X","grandfather","?:Z"]]
      },
      {"@question": ["$arc","?:X","grandfather","john"]}
    ]

The example can be converted to TPTP as 

    fof(frm_1,axiom,$arc(pete,father,john)).
    fof(frm_2,axiom,$arc(mark,father,pete)).
    fof(gfrule,axiom,(! [X,Y,Z] : 
                       (($arc(Y,father,Z) & $arc(X,father,Y)) 
                       => 
                       $arc(X,grandfather,Z)))).
    fof(frm_4,negated_conjecture,
          (! [X] : ($arc(X,grandfather,john) => $ans(X)))).

and given a proof as

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","mark"]],
    "proof":
    [
    [1, ["in", "gfrule"], [["-$arc","?:X","father","?:Y"], 
                           ["-$arc","?:Z","father","?:X"], 
                           ["$arc","?:Z","grandfather","?:Y"]]],
    [2, ["in", "frm_1"], [["$arc","pete","father","john"]]],
    [3, ["mp", 1, 2], [["-$arc","?:X","father","pete"], ["$arc","?:X","grandfather","john"]]],
    [4, ["in", "frm_2"], [["$arc","mark","father","pete"]]],
    [5, ["mp", 3, 4], [["$arc","mark","grandfather","john"]]],
    [6, ["in", "frm_4"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [7, ["mp", 5, 6], [["$ans","mark"]]]
    ]}
    ]}

The following example using objects/maps inside the value of the `"@logic"` key
along with the free variables has an exactly same translation and 
proof as the previous example:

    [
    {"@id":"pete", "father":"john"},
    {"@id":"mark", "father":"pete"},
    {"@name":"gfrule",
     "@logic": ["if", 
                  {"@id":"?:X","father":"?:Y"}, 
                  {"@id":"?:Y","father":"?:Z"}, 
                "then", 
                  {"@id":"?:X","grandfather":"?:Z"}]
    },
    {"@question": {"@id":"?:X","grandfather":"john"}}
    ]

The use of free variables in the last example illustrates that their
scope is limited to the top level list element: `"?:X"` in
`{"@id":"?:X","father":"?:Y"}` is different from `"?:X"` in
 `{"@id":"?:X","grandfather":"john"}`.


### ans and question

A conventional way to find out interesting substitutions for variables -
the answers we are looking for - is to use the special `$ans` predicate
with the meaning that deriving a clause containing only atoms with this predicate
is equivalent to deriving an empty clause, i.e. false, meaning the proof is
found. The arguments of `$ans` are then the answers we look for.
The presence of several `$ans` atoms in the clause indicates an
indefinite answer.

JSON-LD-LOGIC uses `"$ans"` as such a predicate.

Example:

    [
      ["father","john","pete"],
      ["father","pete","mark"],
      ["forall", ["X","Y","Z"], [[["father","X","Y"], "&", ["father","Y","Z"]], 
                                 "=>", 
                                ["grandfather","X","Z"]]],
      ["forall", ["X"], [["grandfather","john","X"],"=>",["$ans","X"]]]
    ]

gives a proof containing an answer:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","mark"]],
    "proof":
    [
    [1, ["in", "frm_3"], [["-father","?:X","?:Y"], 
                          ["-father","?:Z","?:X"], 
                          ["grandfather","?:Z","?:Y"]]],
    [2, ["in", "frm_2"], [["father","pete","mark"]]],
    [3, ["mp", 1, 2], [["-father","?:X","pete"], ["grandfather","?:X","mark"]]],
    [4, ["in", "frm_1"], [["father","john","pete"]]],
    [5, ["mp", 3, 4], [["grandfather","john","mark"]]],
    [6, ["in", "frm_4"], [["-grandfather","john","?:X"], ["$ans","?:X"]]],
    [7, ["mp", 5, 6], [["$ans","mark"]]]
    ]}
    ]}

It may be possible to indicate to the reasoner that several different
answers are looked for. The ways to do this are implementation-specific.

JSON-LD-LOGIC introduces a convenience key `"@question"` automatically
forming the implication with the `"$ans"` predicate with all the free
variables in its value as arguments.

The following example

    [
      ["father","john","pete"],
      ["father","pete","mark"],
      ["forall", ["X","Y","Z"], [[["father","X","Y"], "&", ["father","Y","Z"]], 
                                "=>", 
                                ["grandfather","X","Z"]]],
      {"@question": ["grandfather","john","?:X"]}
    ]

is thus equivalent to the previous example with the explicit `"$ans"`, plus
it gives the formula the `negated_conjecture` role telling the reasoner that
this particular formula is a goal to be focused on: this information often
leads to a much faster proof search. The conversion to TPTP is thus:


    fof(frm_1,axiom,father(john,pete)).
    fof(frm_2,axiom,father(pete,mark)).
    fof(frm_3,axiom,(! [X,Y,Z] : ((father(X,Y) & father(Y,Z)) => grandfather(X,Z)))).
    fof(frm_4,negated_conjecture,(! [X] : (grandfather(john,X) => $ans(X)))).


### Uniqueness of Skolem functions

While not strictly a part of the language, conversion to the
[skolem normal form](https://en.wikipedia.org/wiki/Skolem_normal_form) 
replacing existentially quantified variables with new *skolem constants* and
*skolem functions* commonly used during conversion to the 
[conjuctive normal form](https://en.wikipedia.org/wiki/Conjunctive_normal_form)
(*clausification procedure*) must guarantee uniqueness of generated symbols 
when the output is a JSON-LD-LOGIC document intended to be used together
with other JSON-LD or JSON-LD-LOGIC documents. Similarly, uniqueness should be
guaranteed for new unique predicates and other unique symbols possibly introduced
during clausification.

A suitable way for guaranteering uniqueness is using new *blank nodes* like
`_:sk_N`as such symbols. This guarantees that when JSON-LD-LOGIC expressions
are skolemized or clausified to the JSON-LD-LOGIC form, different resulting
documents can be merged using the JSON-LD/RDF *blank node semantics* without
a conflict of generated symbols in different documents.


### Convenience connectives and predicates

The following "convenience" connectives and constants 
`"-", "not", "and", "or", "if" ... "then" ..., null`
are translated to the core fragment constructions as follows.

The minus sign `-` can be used instead of the tilde `~` both as a logical negation and
as a prefix of a string, indicating that the string is a predicate which has to be
negated.

The `"not"` connective can be used as a negation connective instead of the tilde `"~"`.

The `"and"` and `"or"` connectives can be used instead of the `"&"` and `"|"` correspondingly,
whereas they can be used both in the infix and prefix form, the latter form taking an 
arbitrary number of arguments like this:

  `["and", ["p",1], ["foo","?:X", "a], ["bar"]]`


The `"and"` with no arguments is equivalent to `true` and the `"or"` with no arguments to `false`.

The mixfix operator `if..then..` can be used. The list of formulas before `then` is treated
as a conjunction premiss of the implication and the list after `then` as a disjunction
consequent of the implication.  Example:

  `["if", "a", "b", "c", "then", "d", "e"]`

  is translated as 

  `[["a","&","b","&","c"], "=>", ["d","|","e"]]`  

The equality predicate `"="` is also allowed in the prefix form, i.e. as a first
element of a list. It may be prefixed with  `-` or  `~` to indicate inequality.

### null

The `null` symbol of JSON is not given a semantics as a value in JSON-LD.

JSON-LD-LOGIC deviates from this lack of interpretation by treating `null` 
analogously to the SQL `null` representing a missing or unknown value. 
The translation mechanism of eliminating a `null` inside some formula
`F` is as follows:

* Create a new variable `V` not occurring in `F`. I.e. `V` is a new string.
* Replace an atom `[...null....]` where the eliminated `null` occurs by a formula

   `["exists", [V], [....V....]]`

where this particular occurrence of `null` is replaced by the variable `V'.

Example:

   `["father","john",null]` 

is translated as 

   `["exists", ["X"], ["father","john","X"]]`. 

Example:

   `[["is_father","?:X"], "<=>", ["father","?:X",null]]` 

is translated as 

   `[["is_father","?:X"], "<=>", ["exists", ["Y"], ["father","?:X","Y"]]]`
    

### Multiple values and the context and namespaces and the base type

Multiple values like given in `"son": ["mark","michael"]` in the
following example are interpreted according to the JSON-LD RDF
interpretation as leading to multiple triplets. The following example

    [
      {"a":["b","c"]}     
    ]

is converted as

    fof(frm_1,axiom,(
      $arc('_:crtd_1',a,c) &
      $arc('_:crtd_1',a,b))).

An important feature of JSON-LD is the use of the `"@context"` key to specify,
for example, namespaces, symbol mapping and types.

JSON-LD-LOGIC expands strings in the object/map by the JSON-LD `"@context"` 
expansion rules. The following example

    [
      {
        "@context": {
          "@vocab": "http://foo.org/",
          "b": "c",
          "g": {"@id": "h"}
        },
        "a": "b",
        "b": "c",
        "c": "d",
        "http://e.bar": "f",
        "g":"h"
      }  
    ]

can be converted to

    fof(frm_1,axiom,(
      $arc('_:crtd_1','http://foo.org/h',h) &
     ($arc('_:crtd_1','http://e.bar',f) &
     ($arc('_:crtd_1','http://foo.org/c',d) &
     ($arc('_:crtd_1','http://foo.org/c',c) &
      $arc('_:crtd_1','http://foo.org/a',b)))))).
  
Notice that the example uses inline `"@context"` value, not one read
from an URL: reading the `"@context"` value from the
URL would make the interpretation of the formula indeterminate.

The following example of using `"@base"` key inside a context

    [
      {
        "@context": {
          "@base": "http://foo.org/"       
        },
        "@id": "a",
        "b": "c"        
      }  
    ]

can be converted to

    fof(frm_1,axiom,$arc('http://foo.org/a',b,c)).

The question as for what is the default base URI for `"@id"` key values
is left open in JSON-LD-LOGIC. In the following we assume it is an empty
string. 

The `"@type"` key outside a `"@context"` value is converted to the RDF-specified string
`"http://www.w3.org/1999/02/22-rdf-syntax-ns#type"`.

Consider the following example using multiple values and the JSON-LD `"@context"` 
`"@base"` and  `"@type"` keys along with the JSON-LD-LOGIC `"@logic"` key and 
the convenience operator  `if ... then ...` and the `"@question"` key described later:

    [
    { "@context": {
        "@base": "http://people.org/",
        "@vocab":"http://bar.org/", 
        "father":"http://foo.org/father"
      },
      "@id":"pete", 
      "@type": "male",
      "father":"john",
      "son": ["mark","michael"],
      "@logic": ["forall",["X","Y"],
                  [[{"@id":"X","son":"Y"}, "&", {"@id":"X","@type":"male"}], 
                  "=>", 
                  {"@id":"Y","father":"X"}]
      ]
    },      
    ["if", 
       {"@id":"?:X","http://foo.org/father":"?:Y"},
       {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
      "then", 
      {"@id":"?:X","grandfather":"?:Z"}],
    {"@question": {"@id":"?:X","grandfather":"john"}}
    ]

The formula can be translated to TPTP as


    fof(frm_1,axiom,
        ((! [X,Y] : 
           (($arc(X,'http://bar.org/son',Y) & 
             $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)) 
             => 
             $arc(Y,'http://foo.org/father',X))) 
         & 
         ($arc('http://people.org/pete','http://bar.org/son',michael) & 
         ($arc('http://people.org/pete','http://bar.org/son',mark) & 
         ($arc('http://people.org/pete','http://foo.org/father',john) & 
         $arc('http://people.org/pete','http://www.w3.org/1999/02/22-rdf-syntax-ns#type',male)))))).
    fof(frm_2,axiom,
        (! [X,Y,Z] : 
          (($arc(Y,'http://foo.org/father',Z) & $arc(X,'http://foo.org/father',Y)) 
          => 
          $arc(X,grandfather,Z)))).
    fof(frm_3,negated_conjecture,(! [X] : ($arc(X,grandfather,john) => $ans(X)))).


Observe that unless a formula in the top level list is in the scope of the 
same `"@context"` key value, the namespaces of symbols should be defined either
with a copy of the `"@context"` key value or as absolute values, like done in
this example.

Proof of the example:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","michael"]],
    "proof":
    [
    [1, ["in", "frm_1"], [["-$arc","?:X","http://bar.org/son","?:Y"], 
                          ["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"],
                          ["$arc","?:Y","http://foo.org/father","?:X"]]],
    [2, ["in", "frm_1"], [["$arc","http://people.org/pete","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","male"]]],
    [3, ["mp", [1,1], 2], [["-$arc","http://people.org/pete","http://bar.org/son","?:X"], 
                           ["$arc","?:X","http://foo.org/father","http://people.org/pete"]]],
    [4, ["in", "frm_1"], [["$arc","http://people.org/pete","http://bar.org/son","michael"]]],
    [5, ["mp", 3, 4], [["$arc","michael","http://foo.org/father","http://people.org/pete"]]],
    [6, ["in", "frm_2"], [["-$arc","?:X","http://foo.org/father","?:Y"], 
                          ["-$arc","?:Z","http://foo.org/father","?:X"],
                          ["$arc","?:Z","grandfather","?:Y"]]],
    [7, ["in", "frm_1"], [["$arc","http://people.org/pete","http://foo.org/father","john"]]],
    [8, ["mp", 6, 7], [["-$arc","?:X","http://foo.org/father","http://people.org/pete"], 
                       ["$arc","?:X","grandfather","john"]]],
    [9, ["mp", 5, 8], [["$arc","michael","grandfather","john"]]],
    [10, ["in", "frm_3"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [11, ["mp", 9, 10], [["$ans","michael"]]]
    ]}
    ]}

### Nested objects aka maps

JSON-LD interprets JSON objects/maps as key values as new things with their
own id-s and properties.

JSON-LD-LOGIC interprets and converts such values correspondingly.

Nesting can be arbitrarily deep and nested objects/maps may contain the `@logic` 
key at any level

A simple example:

    [
      {"a":"b"},
      {"c":  
         {
          "d":"e",
          "f":"g"
         } 
      }   
    ]

is converted to

    fof(frm_1,axiom,$arc('_:crtd_1',a,b)).
    fof(frm_2,axiom,(
      ($arc('_:crtd_3',f,g) & 
       $arc('_:crtd_3',d,e)) & 
       $arc('_:crtd_2',c,'_:crtd_3'))).


Logic formulas may also contain nested objects. The following example indicates
that pete has a daughter and a son with the same unknown age:

    [
      ["exists",["X"],
        {"@id":"pete", 
         "son": {"age":"X"},
         "daughter": {"age":"X"} 
        }
      ]  
    ]

The example can be converted to

    fof(frm_1,axiom,
      (? [X] : 
        ($arc('_:crtd_2',age,X) & 
        ($arc(pete,daughter,'_:crtd_2') & 
        ($arc('_:crtd_1',age,X) & 
        $arc(pete,son,'_:crtd_1')))))).

Notice that in the last example we could have used a *blank node* instead of an
existentially quantified `X`: the clausified form of the example turns `X` into
a *Skolem constant* which should be represented by a *blank node*.

Next, let us define a type `fatheroftwins` illustrating the nesting of several
layers of lists and objects/maps. In particular, one of the internal objects/maps
also contains a `@logic` key:

    [
      [
        {"@id":"?:X","@type":"fatheroftwins"},
        "<=>",
        ["exists",
          ["C1","C2","A","M"],
          { 
            "@id":"?:X",
            "child": [
              {"@id":"C1","age":"A","father":"?:X","mother":"M"},
              {"@id":"C2","age":"A","father":"?:X","mother":"M"}
            ],
            "@logic":  ["C1","!=","C2"]                                  
          }
        ]    
      ]
    ] 

The example above can be converted as
    
    fof(frm_1,axiom,
      (! [X] : 
        ($arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',fatheroftwins) 
        <=> 
        (? [C1,C2,A,M] : 
          (~(C1 = C2) & 
           (($arc(C2,mother,M) & ($arc(C2,father,X) & $arc(C2,age,A))) & 
           ($arc(X,child,C2) & 
           (($arc(C1,mother,M) & ($arc(C1,father,X) & $arc(C1,age,A))) & 
           $arc(X,child,C1))))))))).

Since the values of variables in this example depend on the values of other variables,
we cannot express the same logic by using the blank nodes instead of bound variables:
the clausified version of the formula contains four *Skolem functions* with the `X`
variable as argument, each corresponding to one existentially quantified variable.

Next we will present a more complex example with a nested `"child"` value indicating that
the person we describe (with no id given) has two children with ages
10 and 2, but nothing more is known about them. We also know the person
has a father `john`. The rules state the son/daughter and mother/father
correspondence, define children with ages between 19 and 6 as schoolchildren
and define both the maternal and paternal grandfather. The question is to 
find a schoolchild with `john` as a grandfather. Notice the use of arithmetic
predicate `"$less"`. 

    [
    { "@context": {    
        "@vocab":"http://foo.org/",   
        "age":"hasage"
      },
      "father":"john",
      "child": [
        {"age": 10}, 
        {"age": 2} 
      ],      
      "@logic": ["and",    
        ["forall",["X","Y"],[{"@id":"X","child":"Y"}, 
                            "<=>", 
                            ["or",{"@id":"Y","father":"X"},{"@id":"Y","mother":"X"}]]],     
        ["forall",["X","Y"],[{"@id":"Y","child":"X"}, 
                            "<=>", 
                            ["or",{"@id":"Y","son":"X"},{"@id":"Y","daughter":"X"}]]]
      ]  
    },      
    [
      "if", 
        {"@id":"?:X","http://foo.org/hasage":"?:Y"}, 
        ["$less",6,"?:Y"], 
        ["$less","?:Y",19], 
      "then", 
        {"@id":"?:X", "@type": "schoolchild"}
    ],
    [
      "if", 
        {"@id":"?:X","http://foo.org/father":"?:Y"}, 
        {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
      "then",
        {"@id":"?:X","grandfather":"?:Z"}
    ],
    [
      "if", 
        {"@id":"?:X","http://foo.org/mother":"?:Y"}, 
        {"@id":"?:Y","http://foo.org/father":"?:Z"}, 
      "then", 
        {"@id":"?:X","grandfather":"?:Z"}
    ],
    {"@question": [{"@id":"?:X","grandfather":"john"}, 
                    "&", 
                   {"@id":"?:X","@type":"schoolchild"}]}
    ]

The conversion to TPTP creates new blank nodes `_:crtd_1` for
the person and `_:crtd_2` and `_:crtd_3` for the children:

    fof(frm_1,axiom,(((! [X,Y] : ($arc(X,'http://foo.org/child',Y) 
                                  <=> 
                                  ($arc(Y,'http://foo.org/father',X) | 
                                   $arc(Y,'http://foo.org/mother',X)))) & 
                     (! [X,Y] : ($arc(Y,'http://foo.org/child',X) 
                                 <=> ($arc(Y,'http://foo.org/son',X) | 
                                     $arc(Y,'http://foo.org/daughter',X))))) &
                     ($arc('_:crtd_3','http://foo.org/hasage',2) &
                     ($arc('_:crtd_1','http://foo.org/child','_:crtd_3') &
                     ($arc('_:crtd_2','http://foo.org/hasage',10) & 
                     ($arc('_:crtd_1','http://foo.org/child','_:crtd_2') & 
                     $arc('_:crtd_1','http://foo.org/father',john))))))).

    fof(frm_2,axiom,(! [X,Y] : ((($less(Y,19) &
                                  $less(6,Y)) & 
                                  $arc(X,'http://foo.org/hasage',Y)) 
                     =>
                     $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',schoolchild)))).
    fof(frm_3,axiom,(! [X,Y,Z] : (($arc(Y,'http://foo.org/father',Z) & 
                                  $arc(X,'http://foo.org/father',Y)) 
                                  => 
                                 $arc(X,grandfather,Z)))).
    fof(frm_4,axiom,(! [X,Y,Z] : (($arc(Y,'http://foo.org/father',Z) &
                                   $arc(X,'http://foo.org/mother',Y)) 
                                  => 
                                  $arc(X,grandfather,Z)))).
    fof(frm_5,negated_conjecture,
          (! [X] : (($arc(X,grandfather,john) & 
                     $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',schoolchild)) 
                     => 
                     $ans(X)))).

and the proof found gives a newly created blank node for one of the children as
an answer:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","_:crtd_2"]],
    "proof":
    [
    [1, ["in", "frm_2"], [["-$arc","?:X","http://foo.org/hasage","?:Y"], 
                          ["$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"], 
                          ["-$less",6,"?:Y"], ["-$less","?:Y",19]]],
    [2, ["in", "frm_1"], [["$arc","_:crtd_2","http://foo.org/hasage",10]]],
    [3, ["mp", 1, 2], [["$arc","_:crtd_2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"]]],
    [4, ["in", "frm_1"], [["-$arc","?:X","http://foo.org/child","?:Y"],
                          ["$arc","?:Y","http://foo.org/mother","?:X"], 
                          ["$arc","?:Y","http://foo.org/father","?:X"]]],
    [5, ["in", "frm_1"], [["$arc","_:crtd_1","http://foo.org/child","_:crtd_2"]]],
    [6, ["mp", 4, 5], [["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"], 
                       ["$arc","_:crtd_2","http://foo.org/mother","_:crtd_1"]]],
    [7, ["in", "frm_4"], [["-$arc","?:X","http://foo.org/father","?:Y"], 
                          ["-$arc","?:Z","http://foo.org/mother","?:X"], 
                          ["$arc","?:Z","grandfather","?:Y"]]],
    [8, ["in", "frm_1"], [["$arc","_:crtd_1","http://foo.org/father","john"]]],
    [9, ["mp", 7, 8], [["-$arc","?:X","http://foo.org/mother","_:crtd_1"],
                       ["$arc","?:X","grandfather","john"]]],
    [10, ["mp", [6,1], 9], [["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"],
                            ["$arc","_:crtd_2","grandfather","john"]]],
    [11, ["in", "frm_5"], [["-$arc","?:X","grandfather","john"],
                           ["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"],
                           ["$ans","?:X"]]],
    [12, ["mp", [10,1], 11], [["-$arc","_:crtd_2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"], 
                              ["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"],
                              ["$ans","_:crtd_2"]]],
    [13, ["mp", 3, 12], [["$arc","_:crtd_2","http://foo.org/father","_:crtd_1"], 
                         ["$ans","_:crtd_2"]]],
    [14, ["in", "frm_3"], [["-$arc","?:X","http://foo.org/father","?:Y"],
                           ["-$arc","?:Z","http://foo.org/father","?:X"],
                           ["$arc","?:Z","grandfather","?:Y"]]],
    [15, ["mp", 14, 8], [["-$arc","?:X","http://foo.org/father","_:crtd_1"],
                         ["$arc","?:X","grandfather","john"]]],
    [16, ["mp", 13, 15], [["$arc","_:crtd_2","grandfather","john"],
                          ["$ans","_:crtd_2"]]],
    [17, ["mp", 1, 2], [["$arc","_:crtd_2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","schoolchild"]]],
    [18, ["mp", 16, 11, 17], [["$ans","_:crtd_2"]]]
    ]}
    ]}



## Graphs and quads with the narc predicate

JSON-LD uses the `"@graph"` key for two objectives:

* If an object does not contain an `@id` key but contains `"@graph"`, the value of the latter is 
  simply a list of objects in the scope of the `"@context"` of the object.
  JSON-LD-LOGIC converts such lists to conjunctions, each element using
  the same expansion rules.

* If an object does contain an `@id` key, the conjunction elements are not interpreted as RDF triplets, 
  but *quads* with the `@id` key value as a fourth element: the `@id` value indicates which graph the
  arc belongs to. JSON-LD-LOGIC converts such lists using the four-argument `"$narc"`
  predicate (named arc) with the graph `@id` value as the last argument.
   
A grandfather example with two trivial named graphs and a rule merging the named graphs into one
unnamed graph:   

    [
    { 
      "@id":"bobknows",
      "@graph": [  
        {"@id":"pete", "father":"john"}
      ]
    },    
    { 
      "@id":"eveknows",
      "@graph": [  
        {"@id":"mark", "father":"pete"}
      ]
    }, 
    {
      "@name":"mergegraphs",
      "@logic": [["$narc","?:X","?:Y","?:Z","?:U"],"=>",["$arc","?:X","?:Y","?:Z"]]
    },  
    {
      "@name":"gfrule",
      "@logic": ["if", 
                    {"@id":"?:X","father":"?:Y"}, 
                    {"@id":"?:Y","father":"?:Z"}, 
                 "then", 
                    {"@id":"?:X","grandfather":"?:Z"}]
    },      
    {"@question": {"@id":"?:X","grandfather":"john"}}
    ]

Conversion to TPTP gives

    fof(frm_1,axiom,$narc(pete,father,john,bobknows)).
    fof(frm_2,axiom,$narc(mark,father,pete,eveknows)).
    fof(mergegraphs,axiom,(! [X,Y,Z,U] : ($narc(X,Y,Z,U) => $arc(X,Y,Z)))).
    fof(gfrule,axiom,(! [X,Y,Z] : (($arc(Y,father,Z) & $arc(X,father,Y))
                                   => 
                                   $arc(X,grandfather,Z)))).
    fof(frm_5,negated_conjecture,(! [X] : ($arc(X,grandfather,john) => $ans(X)))).

And we get the expected result

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","mark"]],
    "proof":
    [
    [1, ["in", "gfrule"], [["-$arc","?:X","father","?:Y"], 
                           ["-$arc","?:Z","father","?:X"],
                           ["$arc","?:Z","grandfather","?:Y"]]],
    [2, ["in", "mergegraphs"], [["-$narc","?:X","?:Y","?:Z","?:U"], 
                                ["$arc","?:X","?:Y","?:Z"]]],
    [3, ["in", "frm_1"], [["$narc","pete","father","john","bobknows"]]],
    [4, ["mp", 2, 3], [["$arc","pete","father","john"]]],
    [5, ["mp", 1, 4], [["-$arc","?:X","father","pete"],
                       ["$arc","?:X","grandfather","john"]]],
    [6, ["in", "frm_2"], [["$narc","mark","father","pete","eveknows"]]],
    [7, ["mp", 2, 6], [["$arc","mark","father","pete"]]],
    [8, ["mp", 5, 7], [["$arc","mark","grandfather","john"]]],
    [9, ["in", "frm_5"], [["-$arc","?:X","grandfather","john"], ["$ans","?:X"]]],
    [10, ["mp", 8, 9], [["$ans","mark"]]]
    ]}
    ]}


## Numbers and arithmetic

Although JSON-LD-LOGIC defines several functions and predicates on numbers, it
does not axiomatize the properties of these functions except direct evaluation
on numbers. Citing TPTP:

The extent to which ATP systems are able to work with the arithmetic predicates and
functions can vary, from a simple ability to evaluate ground terms, e.g., 
`$sum(2,3)` can be evaluated to 5, through an ability to instantiate variables 
in equations involving such functions, e.g., `$product(2,$uminus(X)) = $uminus($sum(X,2))` 
can instantiate `X` to 2, to extensive algebraic manipulation capability. 
The syntax does not axiomatize arithmetic theory, but may be used to write axioms of the theory. 

The same general principle holds for lists and distinct symbols interpreted as strings.

This said, the numbers and arithmetic functions and predicates are defined as follows:

* JSON numbers without a fractional part and exponent stand for integers in the typed
  fragment of TPTP. Different integers are assumed to be inequal.
  Examples:

  `23, -1, 0`

* JSON numbers with a fractional part stand for reals in the typed
  fragment of TPTP. Integers are separate from reals, thus `4` is different 
  from `4.0000`. Examples: 
  
  `23.45, 0.0, -4.0000000, 1.0E+2`  

* The following TPTP prefix form arithmetic predicates and functions on
  integers and reals are used with the same meaning as defined 
  in the [TPTP arithmetic system](http://www.tptp.org/TPTP/TR/TPTPTR.shtml#Arithmetic):
  
   * Type detection predicates `"$is_int"`, `"$is_real"`.
   
   * Comparison predicates `"$less"`, `"$lesseq"`, `"$greater"`, `"$greatereq"`.

   * Type conversion functions `"$to_int"`, `"$to_real"`.

   * Arithmetic functions on integers and reals:
      `"$sum", "$difference", "$product", 
      "$quotient", "$quotient_e",
      "$remainder_e", "$remainder_t", "$remainder_f", 
      "$floor", "$ceiling",
      "$uminus", "$truncate", "$round"`    

    Note: these comparison predicates and arithmetic functions take exactly two arguments and
    can thus occur only in the lists with the length three.  

    Example: `["$less",["$sum",1,["$to_int",2.1]],["$product",3,3]]`

* Additional convenience predicate is used: `"$is_number"` is true
  if and only if `"$is_int"` or `"$is_real"` is true.

* Additional infix convenience functions `"+", "-", "*", "/"` are
  used with the same meaning as `"$sum"`, `"$difference"`, `"$product"` and 
  `"$quotient"`, respectively.

  Example: `["$less",[1,"+",[1,"+",2]],[3,"*",3]]`

  Note: these arithmetic functions take exactly two arguments and
  can thus occur only in the lists with the length three.

The arithmetic predicates and functions can be applied to any types of objects, including
variables and functional terms: if the arguments are not numbers, the application is in most
cases simply left as is, without evaluation. However, an implementation may, for example,
detect that under given conditions two non-numeric terms cannot be equal and evaluate a formula
like `[["$greater","?:X",0], "=>", [["$sum",1,"?:X"],"=","?:X"]]` to *false*. JSON-LD-LOGIC
does neither require nor prohibit such evaluations.

## Lists and list functions

The RDF semantics of the JSON-LD list value construction using the `"@list"`key 
like in `"clients": {"@list":["a","b","c"]}` requires the construction of a number of
triplets with `rdf:first`, `rdf:rest` and `rdf:nil` properties.

JSON-LD-LOGIC deviates from this semantics since it can use function terms instead of
triplets.

A list is converted using the special `$list` function appending a first argument to
the second argument and the `$nil` constant for an empty list.

Example:

    [{"foo": {"@list":["a",["b","c"],[],"d"]}}]

is converted to

    fof(frm_1,axiom,$arc('_:crtd_1',foo,
      $list(a,$list($list(b,$list(c,$nil)),$list($nil,$list(d,$nil)))))).

A JSON-LD list value using the `"@list"`key must not contain a JSON object/map like `{"a":1}`.

Terms constructed using `$list` or `$nil` are interpreted as having a *list type*: 

* A *list type object* is inequal to any number or a distinct symbol.

* Syntactically different *list type objects* A and B are unequal if at any position the corresponding
  elements of A and B are unequal typed values: numbers, lists or distinct symbols.

For example, the atom `[["$list","a","$nil"],"=",["$list","b","$nil"]]` does not evaluate to *false* while
`[["$list","a",["$list","#:c",$nil"]],"=",["$list","?:X",["$list",1,$nil"]]` does evaluate to *false*.

This allows defining different functions on lists using the equality predicate.

JSON-LD-LOGIC defines a predicate and two functions on lists:

* `["$is_list",L]`  evaluates to *true* if A is a list and *false* is A is a number or a distinct symbol.

* `["$first",L]`  returns the first element of the list.

* `["$rest",L]`  returns the rest of the list, i.e. the result of removing the first element.

These functions can be applied to non-list arguments, where they are left as is and not 
evaluated.

Observe that since JSON-LD-LOGIC does not contain a theory of arithmetic, lists or strings,
an example formula `["exists",["X"],["$is_list","X"]]` is not guaranteed to be provable:
an implementation may prove it or not, depending on the particular theory and a proof search method
implemented. 

The following example defines functions for counting the length of the list and summing
the numeric elements. Then it uses these functions in a rule for deriving a `"goodcompany"` type
as an object with at least three customers and revenues over 100. Finally we ask to find an
object with the `"goodcompany"` type.

    [
    {
      "@id":"company1",
      "clients": {"@list":["a","b","c"]},
      "revenues": {"@list":[10,20,50,60]}
    },

    [["listcount","$nil"],"=",0],
    [["listcount",["$list","?:X","?:Y"]],"=",["+",1,["listcount","?:Y"]]],

    [["listsum","$nil"],"=",0],
    [["listsum",["$list","?:X","?:Y"]],"=",["+","?:X",["listsum","?:Y"]]],

    [
      "if",
        {"@id":"?:X","clients":"?:C","revenues":"?:R"},
        ["$greater",["listcount","?:C"],2], 
        ["$greater",["listsum","?:R"],100],
      "then", 
        {"@id":"?:X","@type":"goodcompany"} 
    ],

    {"@question": {"@id":"?:X","@type":"goodcompany"}}    
    ]

Conversion to TPTP:


    fof(frm_1,axiom,($arc(company1,revenues,
                                   $list(10,$list(20,$list(50,$list(60,$nil))))) & 
                     $arc(company1,clients,
                                   $list(a,$list(b,$list(c,$nil)))))).
    fof(frm_2,axiom,(listcount($nil) = 0)).
    fof(frm_3,axiom,(! [X,Y] : (listcount($list(X,Y)) = $sum(1,listcount(Y))))).
    fof(frm_4,axiom,(listsum($nil) = 0)).
    fof(frm_5,axiom,(! [X,Y] : (listsum($list(X,Y)) = $sum(X,listsum(Y))))).
    fof(frm_6,axiom,(! [X,R,C] : 
             ((($greater(listsum(R),100) & 
                $greater(listcount(C),2)) & 
                ($arc(X,revenues,R) & $arc(X,clients,C)))
            => 
             $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',goodcompany)))).
    fof(frm_7,negated_conjecture,(! [X] : 
          ($arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',goodcompany) 
          => 
          $ans(X)))).

Result:

    {"result": "proof found",

    "answers": [
    {
    "answer": [["$ans","company1"]],
    "proof":
    [
    [1, ["in", "frm_2"], [["=",["listcount","$nil"],0]]],
    [2, ["in", "frm_3"], [["=",["listcount",["$list","?:X","?:Y"]],[$sum,1,["listcount","?:Y"]]]]],
    [3, ["=", 1, [2,0,7]], [["=",["listcount",["$list","?:X","$nil"]],1]]],
    [4, ["=", 3, [2,0,7]], [["=",["listcount",["$list","?:X",["$list","?:Y","$nil"]]],2]]],
    [5, ["=", 4, [2,0,7]], [["=",["listcount",["$list","?:X",["$list","?:Y",["$list","?:Z","$nil"]]]],3]]],
    [6, ["in", "frm_6"], [["-$greater",["listsum","?:X"],100], ["-$greater",["listcount","?:Y"],2], 
                          ["-$arc","?:Z","revenues","?:X"], ["-$arc","?:Z","clients","?:Y"], 
                          ["$arc","?:Z","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","goodcompany"]]],
    [7, ["in", "frm_1"], [["$arc","company1","revenues",["$list",10,["$list",20,["$list",50,["$list",60,"$nil"]]]]]]],
    [8, ["mp", [6,2], 7], [["-$greater",["listsum",["$list",10,["$list",20,["$list",50,["$list",60,"$nil"]]]]],100],
                          ["$arc","company1","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","goodcompany"], 
                          ["-$greater",["listcount","?:X3"],2], 
                          ["-$arc","company1","clients","?:X3"]]],
    [9, ["in", "frm_4"], [["=",["listsum","$nil"],0]]],
    [10, ["in", "frm_5"], [["=",["listsum",["$list","?:X","?:Y"]],[$sum,"?:X",["listsum","?:Y"]]]]],
    [11, ["=", 9, [10,0,7]], [["=",["listsum",["$list","?:X","$nil"]],[$sum,"?:X",0]]]],
    [12, ["=", 11, [10,0,7]], [["=",["listsum",["$list","?:X",["$list","?:Y","$nil"]]],
                                    [$sum,"?:X",[$sum,"?:Y",0]]]]],
    [13, ["=", 12, [10,0,7]], [["=",["listsum",["$list","?:X",["$list","?:Y",["$list","?:Z","$nil"]]]],
                                    [$sum,"?:X",[$sum,"?:Y",[$sum,"?:Z",0]]]]]],
    [14, ["=", 13, [10,0,7]], [["=",["listsum",["$list","?:X",["$list","?:Y",["$list","?:Z",["$list","?:U","$nil"]]]]],
                                    [$sum,"?:X",[$sum,"?:Y",[$sum,"?:Z",[$sum,"?:U",0]]]]]]],
    [15, ["simp", 8, 14], [["$arc","company1","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","goodcompany"], 
                          ["-$greater",["listcount","?:X"],2],
                          ["-$arc","company1","clients","?:X"]]],
    [16, ["in", "frm_1"], [["$arc","company1","clients",["$list","a",["$list","b",["$list","c","$nil"]]]]]],
    [17, ["=", [5,1,"L"], [15,0,1], 16], [["$arc","company1","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","goodcompany"]]],
    [18, ["in", "frm_7"], [["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","goodcompany"], 
                          ["$ans","?:X"]]],
    [19, ["mp", 17, 18], [["$ans","company1"]]]
    ]}
    ]}



## Distinct symbols as strings 


Since distinct symbols (strings prefixed by `#:`) can be viewed as *string type objects*, JSON-LD-LOGIC 
defines a function and three predicates on distinct symbols:

* `["$strlen",S]` returns the integer length of a distinct symbol S as a string.

* `["$substr", A, B]` evaluates to *true* if a distinct symbol A is a substring of a distinct symbol B, 
   and *false* otherwise, provided that A and B are distinct symbols.

* `["$substrat", A, B, C]` evaluates to *true* if a distinct symbol A is a substring of a 
   distinct symbol B exactly at the integer position C (starting from 0), and *false* otherwise,
   provided that A and B are distinct symbols and C is an integer.

* `["$is_distinct", A]` evaluates to *true* if A is a distinct symbol and *false* if A is a number or a list.

The prefix `"#:"` part of a distinct symbol is not considered to be a part of the symbol interpreted
as a string: for example,`["$strlen","#:ab"]` should evaluate to 2.

The first three functions can be applied to non-distinct arguments, where they are left as is and not 
evaluated. The last predicate can be similarly applied to any arguments. For example, `["$is_distinct", "a"]`
is not evaluated, while `["$is_distinct", 1]` is evaluated to *false* and `["$is_distinct", "#:d"]` is evaluated to *true*.

Observe that since JSON-LD-LOGIC does not contain a theory of arithmetic, lists or strings,
an example formula `["exists",["X"],["$is_distinct","X"]]` is not guaranteed to be provable:
an implementation may prove it or not, depending on the particular theory and a proof search method
implemented. 

The following is an example of using distinct symbols: we define a rule saying that whenever sets of type values
of two objects contain different distinct elements, these objects must be different. Notice that ordinary
symbols `"smith1"` and `"smith2"` could be equal or unequal: syntactic difference of ordinary symbols does not
guarantee that they stand for different objects.


    [
    {
      "@id":"smith1",
      "@type":["#:person","baby"],
      "name":"John Smith"  
    },
    {
      "@id":"smith2",
      "@type":["#:dog","newborn"],  
      "name":"John Smith"  
    },
    ["if", 
      {"@id":"?:X","@type":"?:T1"},
      {"@id":"?:Y","@type":"?:T2"},
      ["?:T1","!=","?:T2"],
    "then", 
      ["?:X","!=","?:Y"]
    ],
    {"@question": ["smith1","!=","smith2"]}
    ]


Conversion to TPTP: 

    fof(frm_1,axiom,($arc(smith1,name,'John Smith') & 
                    ($arc(smith1,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',baby) & 
                    $arc(smith1,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',"person")))).
    fof(frm_2,axiom,($arc(smith2,name,'John Smith') & 
                    ($arc(smith2,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',newborn) &
                     $arc(smith2,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',"dog")))).
    fof(frm_3,axiom,(! [X,T1,Y,T2] : (((~(T1 = T2) & 
                                      $arc(Y,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',T2)) &
                                      $arc(X,'http://www.w3.org/1999/02/22-rdf-syntax-ns#type',T1)) 
                                     =>
                                      ~(X = Y)))).
    fof(frm_4,negated_conjecture,~~(smith1 = smith2)).


The proof:

    {"result": "proof found",

    "answers": [
    {
    "proof":
    [
    [1, ["in", "frm_3"], [["-$arc","?:X","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","?:Y"], 
                          ["-$arc","?:Z","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","?:U"],
                          ["-=","?:Z","?:X"], 
                          ["=","?:U","?:Y"]]],
    [2, ["in", "frm_4"], [["=","smith1","smith2"]]],
    [3, ["mp", [1,2], 2], [["-$arc","smith2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","?:X3"], 
                           ["-$arc","smith1","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","?:Y3"],
                           ["=","?:Y3","?:X3"]]],
    [4, ["simp", 3, 2], [["-$arc","smith2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","?:X"], 
                         ["-$arc","smith2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","?:Y"], 
                         ["=","?:Y","?:X"]]],
    [5, ["in", "frm_2"], [["$arc","smith2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","#:dog"]]],
    [6, ["mp", 4, 5], [["-$arc","smith2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","?:X"],
                       ["=","?:X","#:dog"]]],
    [7, ["in", "frm_1"], [["$arc","smith1","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","#:person"]]],
    [8, ["simp", 7, 2], [["$arc","smith2","http://www.w3.org/1999/02/22-rdf-syntax-ns#type","#:person"]]],
    [9, ["mp", 6, 8], false]
    ]}
    ]}

Finding the last proof is, in a sense, nontrivial. Since JSON-LD-LOGIC does not specify any axioms 
for distinct symbols, by default there are also no inequality axioms between distinct symbols
like `["#:person","!=","#:dog"]`. Finding a proof without these axioms requires that the
search strategy happens to generate evaluable literals like `["#:person","=","#:dog"]` in 
derived clauses. This may actually happen for some search strategies, but not for others.
Implementations may devise their own methods for axiomatizing inequality of distinct symbols
or using specialized strategies: this is neither required nor prohibited by JSON-LD-LOGIC.
