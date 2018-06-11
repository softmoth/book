# Imperative Programming {#imperative-programming-1}

Most of the code shown so far in this book, and indeed, most OCaml code in
general, is *pure*. Pure code works without mutating the program's internal
state, performing I/O, reading the clock, or in any other way interacting
with changeable parts of the world. Thus, a pure function behaves like a
mathematical function, always returning the same results when given the same
inputs, and never affecting the world except insofar as it returns the value
of its computation. *Imperative* code, on the other hand, operates by side
effects that modify a program's internal state or interact with the outside
world. An imperative function has a new effect, and potentially returns
different results, every time it's called. [imperative programming/benefits
of]{.idx}[pure code]{.idx}[programming/immutable vs.
imperative]{.idx}[programming/imperative programming]{.idx #PROGimper}

Pure code is the default in OCaml, and for good reason—it's generally
easier to reason about, less error prone and more composable. But imperative
code is of fundamental importance to any practical programming language,
because real-world tasks require that you interact with the outside world,
which is by its nature imperative. Imperative programming can also be
important for performance. While pure code is quite efficient in OCaml, there
are many algorithms that can only be implemented efficiently using imperative
techniques.

OCaml offers a happy compromise here, making it easy and natural to program
in a pure style, but also providing great support for imperative programming.
This chapter will walk you through OCaml's imperative features, and help you
use them to their fullest.

## Example: Imperative Dictionaries {#example-imperative-dictionaries}

We'll start with the implementation of a simple imperative dictionary, i.e.,
a mutable mapping from keys to values. This is really for illustration
purposes; both Core and the standard library provide imperative dictionaries,
and for most real-world tasks, you should use one of those implementations.
There's more advice on using Core's implementation in particular in
[Maps And Hash Tables](maps-and-hashtables.html#maps-and-hash-tables){data-type=xref}.
[dictionaries, imperative]{.idx #DICTimper}[Core standard library/imperative
dictionaries in]{.idx}[imperative programming/imperative
dictionaries]{.idx #IPimpdict}

The dictionary we'll describe now, like those in Core and the standard
library, will be implemented as a hash table. In particular, we'll use an
*open hashing* scheme, where the hash table will be an array of buckets, each
bucket containing a list of key/value pairs that have been hashed into that
bucket. [open hashing]{.idx}

Here's the interface we'll match, provided as an `mli`. The type `('a, 'b) t`
represents a dictionary with keys of type `'a` and data of type `'b`:

<link rel="import" href="code/imperative-programming/dictionary.mli" part=
"1" />

The `mli` also includes a collection of helper functions whose purpose and
behavior should be largely inferrable from their names and type signatures.
Notice that a number of the functions, in particular, ones like `add` that
modify the dictionary, return `unit`. This is typical of functions that act
by side effect.

We'll now walk through the implementation (contained in the corresponding
`ml` file) piece by piece, explaining different imperative constructs as they
come up.

Our first step is to define the type of a dictionary as a record with two
fields:

<link rel="import" href="code/imperative-programming/dictionary.ml" part=
"1" />

The first field, `length`, is declared as mutable. In OCaml, records are
immutable by default, but individual fields are mutable when marked as such.
The second field, `buckets`, is immutable but contains an array, which is
itself a mutable data structure. [fields/mutability of]{.idx}

Now we'll start putting together the basic functions for manipulating a
dictionary:

<link rel="import" href="code/imperative-programming/dictionary.ml" part=
"2" />

Note that `num_buckets` is a constant, which means our bucket array is of
fixed length. A practical implementation would need to be able to grow the
array as the number of elements in the dictionary increases, but we'll omit
this to simplify the presentation.

The function `hash_bucket` is used throughout the rest of the module to
choose the position in the array that a given key should be stored at. It is
implemented on top of `Hashtbl.hash`, which is a hash function provided by
the OCaml runtime that can be applied to values of any type. Thus, its own
type is polymorphic: `'a -> int`.

The other functions defined above are fairly straightforward:

`create`
: Creates an empty dictionary.

`length`
: Grabs the length from the corresponding record field, thus returning the
  number of entries stored in the dictionary.

`find`
: Looks for a matching key in the table and returns the corresponding value
  if found as an option.

Another important piece of imperative syntax shows up in `find`: we write
`array.(index)` to grab a value from an array. `find` also uses
`List.find_map`, which you can see the type of by typing it into the
toplevel:

<link rel="import" href="code/imperative-programming/examples.mlt" part=
"1" />

`List.find_map` iterates over the elements of the list, calling `f` on each
one until a `Some` is returned by `f`, at which point that value is returned.
If `f` returns `None` on all values, then `None` is returned.

Now let's look at the implementation of `iter`:

<link rel="import" href="code/imperative-programming/dictionary.ml" part=
"3" />

`iter` is designed to walk over all the entries in the dictionary. In
particular, `iter t ~f` will call `f` for each key/value pair in dictionary
`t`. Note that `f` must return `unit`, since it is expected to work by side
effect rather than by returning a value, and the overall `iter` function
returns `unit` as well.

The code for `iter` uses two forms of iteration: a `for` loop to walk over
the array of buckets; and within that loop a call to `List.iter` to walk over
the values in a given bucket. We could have done the outer loop with a
recursive function instead of a `for` loop, but `for` loops are syntactically
convenient, and are more familiar and idiomatic in imperative contexts.

The following code is for adding and removing mappings from the dictionary:

<link rel="import" href="code/imperative-programming/dictionary.ml" part=
"4" />

This preceding code is made more complicated by the fact that we need to
detect whether we are overwriting or removing an existing binding, so we can
decide whether `t.length` needs to be changed. The helper function
`bucket_has_key` is used for this purpose.

Another piece of syntax shows up in both `add` and `remove`: the use of the
`<-` operator to update elements of an array (`array.(i) <- expr`) and for
updating a record field (`record.field <- expression`).

We also use `;`, the sequencing operator, to express a sequence of imperative
actions. We could have done the same using `let` bindings:

<link rel="import" href="code/imperative-programming/dictionary2.ml" part=
"1" />

but `;` is more concise and idiomatic. More generally,

<link rel="import" href="code/imperative-programming/semicolon.syntax" />

is equivalent to

<link rel="import" href="code/imperative-programming/let-unit.syntax" />

When a sequence expression `expr1; expr2` is evaluated, `expr1` is evaluated
first, and then `expr2`. The expression `expr1` should have type `unit`
(though this is a warning rather than a hard restriction. The
`-strict-sequence` compiler flag makes this a hard restriction, which is
generally a good idea), and the value of `expr2` is returned as the value of
the entire sequence. For example, the sequence
`print_string "hello world"; 1 + 2` first prints the string `"hello world"`,
then returns the integer `3`.

Note also that we do all of the side-effecting operations at the very end of
each function. This is good practice because it minimizes the chance that
such operations will be interrupted with an exception, leaving the data
structure in an inconsistent state.
<a data-type="indexterm" data-startref="DICTimper">&nbsp;</a><a data-type="indexterm" data-startref="IPimpdict">&nbsp;</a>

## Primitive Mutable Data {#primitive-mutable-data}

Now that we've looked at a complete example, let's take a more systematic
look at imperative programming in OCaml. We encountered two different forms
of mutable data above: records with mutable fields and arrays. We'll now
discuss these in more detail, along with the other primitive forms of mutable
data that are available in OCaml. [array-like data]{.idx}[data
structures/primitive mutable data]{.idx}[mutable data]{.idx}[primitive
mutable data/array-like data]{.idx}[imperative programming/primitive mutable
data]{.idx}

### Array-Like Data {#array-like-data}

OCaml supports a number of array-like data structures; i.e., mutable
integer-indexed containers that provide constant-time access to their
elements. We'll discuss several of them in this section.

#### Ordinary arrays {#ordinary-arrays}

The `array` type is used for general-purpose polymorphic arrays. The 
`Array` module has a variety of utility functions for interacting with
arrays, including a number of mutating operations. These include `Array.set`,
for setting an individual element, and `Array.blit`, for efficiently copying
values from one range of indices to another. [values/copying with
Array.blit]{.idx}[elements/setting with Array.set]{.idx}[Array
module/Array.blit]{.idx}[Array module/Array.set]{.idx}

Arrays also come with special syntax for retrieving an element from an array:

<link rel="import" href="code/imperative-programming/array-get.syntax" />

and for setting an element in an array:

<link rel="import" href="code/imperative-programming/array-set.syntax" />

Out-of-bounds accesses for arrays (and indeed for all the array-like data
structures) will lead to an exception being thrown.

Array literals are written using `[|` and `|]` as delimiters. Thus,
`[| 1; 2; 3 |]` is a literal integer array.

#### Strings {#strings}

Strings are essentially byte arrays which are often used for textual data.
The main advantage of using a `string` in place of a `Char.t array` (a
`Char.t` is an 8-bit character) is that the former is considerably more
space-efficient; an array uses one word—8 bytes on a 64-bit machine—to
store a single entry, whereas strings use 1 byte per character. [byte
arrays]{.idx}[strings/vs. Char.t arrays]{.idx}

Strings also come with their own syntax for getting and setting values:

<link rel="import" href="code/imperative-programming/string.syntax" />

And string literals are bounded by quotes. There's also a module `String`
where you'll find useful functions for working with strings.

#### Bigarrays {#bigarrays}

A `Bigarray.t` is a handle to a block of memory stored outside of the OCaml
heap. These are mostly useful for interacting with C or Fortran libraries,
and are discussed in
[Memory Representation Of Values](runtime-memory-layout.html#memory-representation-of-values){data-type=xref}.
Bigarrays too have their own getting and setting syntax: [bigarrays]{.idx}

<link rel="import" href="code/imperative-programming/bigarray.syntax" />


### Mutable Record and Object Fields and Ref Cells {#mutable-record-and-object-fields-and-ref-cells}

As we've seen, records are immutable by default, but individual record fields
can be declared as mutable. These mutable fields can be set using the 
`<-` operator, i.e., `record.field <- expr`. [fields/mutability of]{.idx}

As we'll see in [Objects](objects.html#objects){data-type=xref}, fields of
an object can similarly be declared as mutable, and can then be modified in
much the same way as record fields. [primitive mutable data/record/object
fields and ref cells]{.idx}

#### Ref cells {#ref-cells}

Variables in OCaml are never mutable—they can refer to mutable data, but
what the variable points to can't be changed. Sometimes, though, you want to
do exactly what you would do with a mutable variable in another language:
define a single, mutable value. In OCaml this is typically achieved using a
`ref`, which is essentially a container with a single mutable polymorphic
field. [ref cells]{.idx}

The definition for the `ref` type is as follows:

<link rel="import" href="code/imperative-programming/ref.mlt" part="1" />

The standard library defines the following operators for working with `ref`s.

`ref expr`
: Constructs a reference cell containing the value defined by the expression
  `expr`.

`!refcell`
: Returns the contents of the reference cell.

`refcell := expr`
: Replaces the contents of the reference cell.

You can see these in action:

<link rel="import" href="code/imperative-programming/ref.mlt" part="3" />

The preceding are just ordinary OCaml functions, which could be defined as
follows:

<link rel="import" href="code/imperative-programming/ref.mlt" part="2" />


### Foreign Functions {#foreign-functions}

Another source of imperative operations in OCaml is resources that come from
interfacing with external libraries through OCaml's foreign function
interface (FFI). The FFI opens OCaml up to imperative constructs that are
exported by system calls or other external libraries. Many of these come
built in, like access to the `write` system call or to the `clock`, while
others come from user libraries, like LAPACK bindings. OCaml's FFI is
discussed in more detail in
[Foreign Function Interface](foreign-function-interface.html#foreign-function-interface){data-type=xref}.
[libraries/interfacing with external]{.idx}[external libraries/interfacing
with]{.idx}[LAPACK bindings]{.idx}[foreign function interface
(FFI)/imperative operations and]{.idx}[primitive mutable data/foreign
functions]{.idx}


## for and while Loops {#for-and-while-loops-1}

OCaml provides support for traditional imperative looping constructs, in
particular, `for` and `while` loops. Neither of these constructs is strictly
necessary, since they can be simulated with recursive functions. Nonetheless,
explicit `for` and `while` loops are both more concise and more idiomatic
when programming imperatively. [looping constructs]{.idx}[while
loops]{.idx}[for loops]{.idx}

The `for` loop is the simpler of the two. Indeed, we've already seen the
`for` loop in action—the `iter` function in `Dictionary` is built using it.
Here's a simple example of `for`:

<link rel="import" href="code/imperative-programming/for.mlt" part="1" />

As you can see, the upper and lower bounds are inclusive. We can also use
`downto` to iterate in the other direction:

<link rel="import" href="code/imperative-programming/for.mlt" part="2" />

Note that the loop variable of a `for` loop, `i` in this case, is immutable
in the scope of the loop and is also local to the loop, i.e., it can't be
referenced outside of the loop.

OCaml also supports `while` loops, which include a condition and a body. The
loop first evaluates the condition, and then, if it evaluates to true,
evaluates the body and starts the loop again. Here's a simple example of a
function for reversing an array in place:

<link rel="import" href="code/imperative-programming/for.mlt" part="3" />

In the preceding example, we used `incr` and `decr`, which are built-in
functions for incrementing and decrementing an `int ref` by one,
respectively.

## Example: Doubly Linked Lists {#example-doubly-linked-lists}

Another common imperative data structure is the doubly linked list. Doubly
linked lists can be traversed in both directions, and elements can be added
and removed from the list in constant time. Core defines a doubly linked list
(the module is called `Doubly_linked`), but we'll define our own linked list
library as an illustration. [lists/doubly-linked lists]{.idx}[doubly-linked
lists]{.idx}[imperative programming/doubly-linked lists]{.idx #IPdoublink}

Here's the `mli` of the module we'll build:

<link rel="import" href="code/imperative-programming/dlist.mli" />

Note that there are two types defined here: `'a t`, the type of a list; and
`'a element`, the type of an element. Elements act as pointers to the
interior of a list and allow us to navigate the list and give us a point at
which to apply mutating operations.

Now let's look at the implementation. We'll start by defining `'a element`
and `'a t`:

<link rel="import" href="code/imperative-programming/dlist.ml" part="1" />

An `'a element` is a record containing the value to be stored in that node as
well as optional (and mutable) fields pointing to the previous and next
elements. At the beginning of the list, the `prev` field is `None`, and at
the end of the list, the `next` field is `None`.

The type of the list itself, `'a t`, is a mutable reference to an optional
`element`. This reference is `None` if the list is empty, and `Some`
otherwise.

Now we can define a few basic functions that operate on lists and elements:

<link rel="import" href="code/imperative-programming/dlist.ml" part="2" />

These all follow relatively straightforwardly from our type definitions.

::: {data-type=note}
### Cyclic Data Structures

Doubly linked lists are a cyclic data structure, meaning that it is possible
to follow a nontrivial sequence of pointers that closes in on itself. In
general, building cyclic data structures requires the use of side effects.
This is done by constructing the data elements first, and then adding cycles
using assignment afterward. [let rec]{.idx}[data
structures/cyclic]{.idx}[cyclic data structures]{.idx}

There is an exception to this, though: you can construct fixed-size cyclic
data structures using `let rec`:

<link rel="import" href="code/imperative-programming/examples.mlt" part=
"2" />

This approach is quite limited, however. General-purpose cyclic data
structures require mutation.
:::


### Modifying the List {#modifying-the-list}

Now, we'll start considering operations that mutate the list, starting with
`insert_first`, which inserts an element at the front of the list:
[elements/inserting in lists]{.idx}

<link rel="import" href="code/imperative-programming/dlist.ml" part="3" />

`insert_first` first defines a new element `new_elt`, and then links it into
the list, finally setting the list itself to point to `new_elt`. Note that
the precedence of a `match` expression is very low, so to separate it from
the following assignment (`t := Some new_elt`), we surround the match with
`begin ... end`. We could have used parentheses for the same purpose. Without
some kind of bracketing, the final assignment would incorrectly become part
of the `None` case. [elements/defining new]{.idx}

We can use `insert_after` to insert elements later in the list.
`insert_after` takes as arguments both an `element` after which to insert the
new node and a value to insert:

<link rel="import" href="code/imperative-programming/dlist.ml" part="4" />

Finally, we need a `remove` function:

<link rel="import" href="code/imperative-programming/dlist.ml" part="5" />

Note that the preceding code is careful to change the `prev` pointer of the
following element and the `next` pointer of the previous element, if they
exist. If there's no previous element, then the list pointer itself is
updated. In any case, the next and previous pointers of the element itself
are set to `None`.

These functions are more fragile than they may seem. In particular, misuse of
the interface may lead to corrupted data. For example, double-removing an
element will cause the main list reference to be set to `None`, thus emptying
the list. Similar problems arise from removing an element from a list it
doesn't belong to.

This shouldn't be a big surprise. Complex imperative data structures can be
quite tricky, considerably trickier than their pure equivalents. The issues
described previously can be dealt with by more careful error detection, and
such error correction is taken care of in modules like Core's
`Doubly_linked`. You should use imperative data structures from a
well-designed library when you can. And when you can't, you should make sure
to put great care into your error handling. [imperative programming/drawbacks
of]{.idx}[Doubly-linked module]{.idx}[error handling/and imperative data
structures]{.idx}

### Iteration Functions {#iteration-functions}

When defining containers like lists, dictionaries, and trees, you'll
typically want to define a set of iteration functions like `iter`, `map`, and
`fold`, which let you concisely express common iteration patterns.
[functions/iteration functions]{.idx}[iteration functions]{.idx}

`Dlist` has two such iterators: `iter`, the goal of which is to call a
`unit`-producing function on every element of the list, in order; and
`find_el`, which runs a provided test function on each values stored in the
list, returning the first `element` that passes the test. Both `iter` and
`find_el` are implemented using simple recursive loops that use `next` to
walk from element to element and `value` to extract the element from a given
node:

<link rel="import" href="code/imperative-programming/dlist.ml" part="6" />

This completes our implementation, but there's still considerably more work
to be done to make a really usable doubly linked list. As mentioned earlier,
you're probably better off using something like Core's `Doubly_linked` module
that has a more complete interface and has more of the tricky corner cases
worked out. Nonetheless, this example should serve to demonstrate some of the
techniques you can use to build nontrivial imperative data structure in
OCaml, as well as some of the
pitfalls.<a data-type="indexterm" data-startref="IPdoublink">&nbsp;</a>


## Laziness and Other Benign Effects {#laziness-and-other-benign-effects}

There are many instances where you basically want to program in a pure style,
but you want to make limited use of side effects to improve the performance
of your code. Such side effects are sometimes called *benign effects*, and
they are a useful way of leveraging OCaml's imperative features while still
maintaining most of the benefits of pure programming. [lazy
keyword]{.idx}[side effects]{.idx}[laziness]{.idx}[benign
effects/laziness]{.idx}[imperative programming/benign effects and]{.idx}

One of the simplest benign effects is *laziness*. A lazy value is one that is
not computed until it is actually needed. In OCaml, lazy values are created
using the `lazy` keyword, which can be used to convert any expression of type
`s` into a lazy value of type `s Lazy.t`. The evaluation of that expression
is delayed until forced with `Lazy.force`:

<link rel="import" href="code/imperative-programming/lazy.mlt" part="1" />

You can see from the `print` statement that the actual computation was
performed only once, and only after `force` had been called.

To better understand how laziness works, let's walk through the
implementation of our own lazy type. We'll start by declaring types to
represent a lazy value:

<link rel="import" href="code/imperative-programming/lazy.mlt" part="2" />

A `lazy_state` represents the possible states of a lazy value. A lazy value
is `Delayed` before it has been run, where `Delayed` holds a function for
computing the value in question. A lazy value is in the `Value` state when it
has been forced and the computation ended normally. The `Exn` case is for
when the lazy value has been forced, but the computation ended with an
exception. A lazy value is simply a `ref` containing a `lazy_state`, where
the `ref` makes it possible to change from being in the `Delayed` state to
being in the `Value` or `Exn` states.

We can create a lazy value from a thunk, i.e., a function that takes a unit
argument. Wrapping an expression in a thunk is another way to suspend the
computation of an expression: [thunks]{.idx}

<link rel="import" href="code/imperative-programming/lazy.mlt" part="3" />

Now we just need a way to force a lazy value. The following function does
just that:

<link rel="import" href="code/imperative-programming/lazy.mlt" part="4" />

Which we can use in the same way we used `Lazy.force`:

<link rel="import" href="code/imperative-programming/lazy.mlt" part="5" />

The main user-visible difference between our implementation of laziness and
the built-in version is syntax. Rather than writing
`create_lazy (fun () -> sqrt 16.)`, we can (with the built-in `lazy`) just
write `lazy (sqrt 16.)`.

### Memoization and Dynamic Programming {#memoization-and-dynamic-programming}

Another benign effect is *memoization*. A memoized function remembers the
result of previous invocations of the function so that they can be returned
without further computation when the same arguments are presented again.
[memoization/of function]{.idx}[benign effects/memoization]{.idx #BEmem}

Here's a function that takes as an argument an arbitrary single-argument
function and returns a memoized version of that function. Here we'll use
Core's `Hashtbl` module, rather than our toy `Dictionary`:

<link rel="import" href="code/imperative-programming/memo.mlt" part="1" />

The preceding code is a bit tricky. `memoize` takes as its argument a
function `f` and then allocates a polymorphic hash table (called
`memo_table`), and returns a new function which is the memoized version of
`f`. When called, this new function uses `Hashtbl.find_or_add` to try to find
a value in the `memo_table`, and if it fails, to call `f` and store the
result. Note that `memo_table` is referred to by the function, and so won't
be collected until the function returned by `memoize` is itself collected.
[memoization/benefits and drawbacks of]{.idx}

Memoization can be useful whenever you have a function that is expensive to
recompute and you don't mind caching old values indefinitely. One important
caution: a memoized function by its nature leaks memory. As long as you hold
on to the memoized function, you're holding every result it has returned thus
far.

Memoization is also useful for efficiently implementing some recursive
algorithms. One good example is the algorithm for computing the
*edit distance* (also called the Levenshtein distance) between two strings.
The edit distance is the number of single-character changes (including letter
switches, insertions, and deletions) required to
<span class="keep-together">convert</span> one string to the other. This kind
of distance metric can be useful for a variety of approximate string-matching
problems, like spellcheckers. [string matching]{.idx}[Levenshtein
distance]{.idx}[edit distance]{.idx}

Consider the following code for computing the edit distance. Understanding
the algorithm isn't important here, but you should pay attention to the
structure of the recursive calls: [memoization/example of]{.idx}

<link rel="import" href="code/imperative-programming/memo.mlt" part="2" />

The thing to note is that if you call `edit_distance "OCaml" "ocaml"`, then
that will in turn dispatch the following calls:

<figure style="float: 0">
  <img src="images/imperative-programming/edit_distance.png"/>
</figure>


And these calls will in turn dispatch other calls:

<figure style="float: 0">
  <img src="images/imperative-programming/edit_distance2.png"/>
</figure>


As you can see, some of these calls are repeats. For example, there are two
different calls to `edit_distance "OCam" "oca"`. The number of redundant
calls grows exponentially with the size of the strings, meaning that our
implementation of `edit_distance` is brutally slow for large strings. We can
see this by writing a small timing function, using the `Mtime` package.

<link rel="import" href="code/imperative-programming/memo.mlt" part="3" />

And now we can use this to try out some examples:

<link rel="import" href="code/imperative-programming/memo.mlt" part="4" />

Just those few extra characters made it thousands of times slower!

Memoization would be a huge help here, but to fix the problem, we need to
memoize the calls that `edit_distance` makes to itself. Such recursive
memoization is closely related to a common algorithmic technique called
*dynamic programming*, except that with dynamic programming, you do the
necessary sub-computations bottom-up, in anticipation of needing them. With
recursive memoization, you go top-down, only doing a sub-computation when you
discover that you need it. [memoization/recursive]{.idx}[dynamic
programming]{.idx}

To see how to do this, let's step away from `edit_distance` and instead
consider a much simpler example: computing the *n*th element of the Fibonacci
sequence. The Fibonacci sequence by definition starts out with two `1`s, with
every subsequent element being the sum of the previous two. The classic
recursive definition of Fibonacci is as follows:

<link rel="import" href="code/imperative-programming/fib.mlt" part="1" />

This is, however, exponentially slow, for the same reason that
`edit_distance` was slow: we end up making many redundant calls to `fib`. It
shows up quite dramatically in the performance:

<link rel="import" href="code/imperative-programming/fib.mlt" part="2" />

As you can see, `fib 40` takes thousands of times longer to compute than
`fib 20`.

So, how can we use memoization to make this faster? The tricky bit is that we
need to insert the memoization before the recursive calls within `fib`. We
can't just define `fib` in the ordinary way and memoize it after the fact and
expect the first call to `fib` to be improved.

<link rel="import" href="code/imperative-programming/fib.mlt" part="3" />

In order to make `fib` fast, our first step will be to rewrite `fib` in a way
that unwinds the recursion. The following version expects as its first
argument a function (called `fib`) that will be called in lieu of the usual
recursive call.

<link rel="import" href="code/imperative-programming/fib.mlt" part="4" />

We can now turn this back into an ordinary Fibonacci function by tying the
recursive knot:

<link rel="import" href="code/imperative-programming/fib.mlt" part="5" />

We can even write a polymorphic function that we'll call `make_rec` that can
tie the recursive knot for any function of this form:

<link rel="import" href="code/imperative-programming/fib.mlt" part="6" />

This is a pretty strange piece of code, and it may take a few moments of
thought to figure out what's going on. Like `fib_norec`, the function
`f_norec` passed into `make_rec` is a function that isn't recursive but takes
as an argument a function that it will call. What `make_rec` does is to
essentially feed `f_norec` to itself, thus making it a true recursive
function.

This is clever enough, but all we've really done is find a new way to
implement the same old slow Fibonacci function. To make it faster, we need a
variant of `make_rec` that inserts memoization when it ties the recursive
knot. We'll call that function `memo_rec`:

<link rel="import" href="code/imperative-programming/fib.mlt" part="7" />

Note that `memo_rec` has the same signature as `make_rec`.

We're using the reference here as a way of tying the recursive knot without
using a `let rec`, which for reasons we'll describe later wouldn't work here.

Using `memo_rec`, we can now build an efficient version of `fib`:

<link rel="import" href="code/imperative-programming/fib.mlt" part="8" />

And as you can see, the exponential time complexity is now gone.

The memory behavior here is important. If you look back at the definition of
`memo_rec`, you'll see that the call `memo_rec fib_norec` does not trigger a
call to `memoize`. Only when `fib` is called and thereby the final argument
to `memo_rec` is presented does `memoize` get called. The result of that call
falls out of scope when the `fib` call returns, and so calling `memo_rec` on
a function does not create a memory leak—the memoization table is collected
after the computation completes.

We can use `memo_rec` as part of a single declaration that makes this look
like it's little more than a special form of `let rec`:

<link rel="import" href="code/imperative-programming/fib.mlt" part="9" />

Memoization is overkill for implementing Fibonacci, and indeed, the `fib`
defined above is not especially efficient, allocating space linear in the
number passed in to `fib`. It's easy enough to write a Fibonacci function
that takes a constant amount of space.

But memoization is a good approach for optimizing `edit_distance`, and we can
apply the same approach we used on `fib` here. We will need to change
`edit_distance` to take a pair of strings as a single argument, since
`memo_rec` only works on single-argument functions. (We can always recover
the original interface with a wrapper function.) With just that change and
the addition of the `memo_rec` call, we can get a memoized version of
`edit_distance`:

<link rel="import" href="code/imperative-programming/memo.mlt" part="6" />

This new version of `edit_distance` is much more efficient than the one we
started with; the following call is many thousands of times faster than it
was without memoization:

<link rel="import" href="code/imperative-programming/memo.mlt" part="7" />

::: {.allow_break data-type=note}
#### Limitations of let rec

You might wonder why we didn't tie the recursive knot in `memo_rec` using
`let rec`, as we did for `make_rec` earlier. Here's code that tries to do
just that: [let rec]{.idx}

<link rel="import" href="code/imperative-programming/letrec.mlt" part=
"1" />

OCaml rejects the definition because OCaml, as a strict language, has limits
on what it can put on the righthand side of a `let rec`. In particular,
imagine how the following code snippet would be compiled:

<link rel="import" href="code/imperative-programming/let_rec.ml" />

Note that `x` is an ordinary value, not a function. As such, it's not clear
how this definition should be handled by the compiler. You could imagine it
compiling down to an infinite loop, but `x` is of type `int`, and there's no
`int` that corresponds to an infinite loop. As such, this construct is
effectively impossible to compile.

To avoid such impossible cases, the compiler only allows three possible
constructs to show up on the righthand side of a `let rec`: a function
definition, a constructor, or the lazy keyword. This excludes some reasonable
things, like our definition of `memo_rec`, but it also blocks things that
don't make sense, like our definition of `x`.

It's worth noting that these restrictions don't show up in a lazy language
like Haskell. Indeed, we can make something like our definition of `x` work
if we use OCaml's laziness:

<link rel="import" href="code/imperative-programming/letrec.mlt" part=
"2" />

Of course, actually trying to compute this will fail. OCaml's `lazy` throws
an exception when a lazy value tries to force itself as part of its own
evaluation.

<link rel="import" href="code/imperative-programming/letrec.mlt" part=
"3" />

But we can also create useful recursive definitions with `lazy`. In
particular, we can use laziness to make our definition of `memo_rec` work
without explicit mutation:

<link rel="import" href="code/imperative-programming/letrec.mlt" part=
"5" />

Laziness is more constrained than explicit mutation, and so in some cases can
lead to code whose behavior is easier to think about.
<a data-type="indexterm" data-startref="BEmem">&nbsp;</a>
:::



## Input and Output {#input-and-output}

Imperative programming is about more than modifying in-memory data
structures. Any function that doesn't boil down to a deterministic
transformation from its arguments to its return value is imperative in
nature. That includes not only things that mutate your program's data, but
also operations that interact with the world outside of your program. An
important example of this kind of interaction is I/O, i.e., operations for
reading or writing data to things like files, terminal input and output, and
network sockets. [I/O (input/output) operations/terminal
I/O]{.idx}[imperative programming/input and output]{.idx #IPinpout}

There are multiple I/O libraries in OCaml. In this section we'll discuss
OCaml's buffered I/O library that can be used through the `In_channel` and
`Out_channel` modules in Core. Other I/O primitives are also available
through the `Unix` module in Core as well as `Async`, the asynchronous I/O
library that is covered in
[Concurrent Programming With Async](concurrent-programming.html#concurrent-programming-with-async){data-type=xref}.
Most of the functionality in Core's `In_channel` and `Out_channel` (and in
Core's `Unix` module) derives from the standard library, but we'll use Core's
interfaces here.

### Terminal I/O {#terminal-io}

OCaml's buffered I/O library is organized around two types: `in_channel`, for
channels you read from, and `out_channel`, for channels you write to. The
`In_channel` and `Out_channel` modules only have direct support for channels
corresponding to files and terminals; other kinds of channels can be created
through the `Unix` module. [Out_channel
module/Out_channel.stderr]{.idx}[Out_channel
module/Out_channel.stdout]{.idx}[In_channel module]{.idx}

We'll start our discussion of I/O by focusing on the terminal. Following the
UNIX model, communication with the terminal is organized around three
channels, which correspond to the three standard file descriptors in Unix:

`In_channel.stdin`
: The "standard input" channel. By default, input comes from the terminal,
  which handles keyboard input.

`Out_channel.stdout`
: The "standard output" channel. By default, output written to `stdout`
  appears on the user terminal.

`Out_channel.stderr`
: The "standard error" channel. This is similar to `stdout` but is intended
  for error messages.

The values `stdin`, `stdout`, and `stderr` are useful enough that they are
also available in the global namespace directly, without having to go through
the `In_channel` and `Out_channel` modules.

Let's see this in action in a simple interactive application. The following
program, `time_converter`, prompts the user for a time zone, and then prints
out the current time in that time zone. Here, we use Core's `Zone` module for
looking up a time zone, and the `Time` module for computing the current time
and printing it out in the time zone in question:

<link rel="import" href="code/imperative-programming/time_converter/time_converter.ml" />

We can build this program using `jbuilder` and run it. You'll see that it
prompts you for input, as follows:

<link rel="import" href="code/imperative-programming/time_converter/time_converter.rawsh" />

You can then type in the name of a time zone and hit Return, and it will
print out the current time in the time zone in question:

<link rel="import" href="code/imperative-programming/time_converter2.rawsh" />

We called `Out_channel.flush` on `stdout` because `out_channel`s are
buffered, which is to say that OCaml doesn't immediately do a write every
time you call `output_string`. Instead, writes are buffered until either
enough has been written to trigger the flushing of the buffers, or until a
flush is explicitly requested. This greatly increases the efficiency of the
writing process by reducing the number of system calls.

Note that `In_channel.input_line` returns a `string option`, with `None`
indicating that the input stream has ended (i.e., an end-of-file condition).
`Out_channel.output_string` is used to print the final output, and
`Out_channel.flush` is called to flush that output to the screen. The final
flush is not technically required, since the program ends after that
instruction, at which point all remaining output will be flushed anyway, but
the explicit flush is nonetheless good practice.

### Formatted Output with printf {#formatted-output-with-printf}

Generating output with functions like `Out_channel.output_string` is simple
and easy to understand, but can be a bit verbose. OCaml also supports
formatted output using the `printf` function, which is modeled after 
`printf` in the C standard library. `printf` takes a *format string* that
describes what to print and how to format it, as well as arguments to be
printed, as determined by the formatting directives embedded in the format
string. So, for example, we can write: [strings/format strings]{.idx}[format
strings]{.idx}[printf function]{.idx}[I/O (input/output) operations/formatted
output]{.idx}

<link rel="import" href="code/imperative-programming/printf.mlt" part=
"1" />

Unlike C's `printf`, the `printf` in OCaml is type-safe. In particular, if we
provide an argument whose type doesn't match what's presented in the format
string, we'll get a type error:

<link rel="import" href="code/imperative-programming/printf.mlt" part=
"2" />

<aside data-type="sidebar">
<h5>Understanding Format Strings</h5>

The format strings used by `printf` turn out to be quite different from
ordinary strings. This difference ties to the fact that OCaml format strings,
unlike their equivalent in C, are type-safe. In particular, the compiler
checks that the types referred to by the format string match the types of the
rest of the arguments passed to `printf`.

To check this, OCaml needs to analyze the contents of the format string at
compile time, which means the format string needs to be available as a string
literal at compile time. Indeed, if you try to pass an ordinary string to
`printf`, the compiler will complain:

<link rel="import" href="code/imperative-programming/printf.mlt" part=
"3" />

If OCaml infers that a given string literal is a format string, then it
parses it at compile time as such, choosing its type in accordance with the
formatting directives it finds. Thus, if we add a type annotation indicating
that the string we're defining is actually a format string, it will be
interpreted as such. (Here, we open the CamlinternalFormatBasics so that the
representation of the format string that's printed out won't fill the whole
page.)

<link rel="import" href="code/imperative-programming/printf.mlt" part=
"4" />

And accordingly, we can pass it to `printf`:

<link rel="import" href="code/imperative-programming/printf.mlt" part=
"5" />

If this looks different from everything else you've seen so far, that's
because it is. This is really a special case in the type system. Most of the
time, you don't need to know about this special handling of format
strings—you can just use `printf` and not worry about the details. But it's
useful to keep the broad outlines of the story in the back of your head.

</aside>

Now let's see how we can rewrite our time conversion program to be a little
more concise using `printf`:

<link rel="import" href="code/imperative-programming/time_converter2.ml" />

In the preceding example, we've used only two formatting directives: 
`%s`, for including a string, and `%!` which causes `printf` to flush the
channel.

`printf`'s formatting directives offer a significant amount of control,
allowing you to specify things like: [binary numbers, formatting with
printf]{.idx}[hex numbers, formatting with printf]{.idx}[decimals, formatting
with printf]{.idx}[alignment, formatting with printf]{.idx}

- Alignment and padding

- Escaping rules for strings

- Whether numbers should be formatted in decimal, hex, or binary

- Precision of float conversions

There are also `printf`-style functions that target outputs other than
`stdout`, including: [sprintf function]{.idx}[fprintf function]{.idx}[eprintf
function]{.idx}

- `eprintf`, which prints to `stderr`

- `fprintf`, which prints to an arbitrary `out_channel`

- `sprintf`, which returns a formatted string

All of this, and a good deal more, is described in the API documentation for
the `Printf` module in the OCaml Manual.

### File I/O {#file-io}

Another common use of `in_channel`s and `out_channel`s is for working with
files. Here are a couple of functions—one that creates a file full of
numbers, and the other that reads in such a file and returns the sum of those
numbers: [files/file I/O]{.idx}[I/O (input/output) operations/file I/O]{.idx}

<link rel="import" href="code/imperative-programming/file.mlt" part="1" />

For both of these functions, we followed the same basic sequence: we first
create the channel, then use the channel, and finally close the channel. The
closing of the channel is important, since without it, we won't release
resources associated with the file back to the operating system.

One problem with the preceding code is that if it throws an exception in the
middle of its work, it won't actually close the file. If we try to read a
file that doesn't actually contain numbers, we'll see such an error:

<link rel="import" href="code/imperative-programming/file.mlt" part="2" />

And if we do this over and over in a loop, we'll eventually run out of file
descriptors:

<link rel="import" href="code/imperative-programming/file.mlt" part="3" />

And now, you'll need to restart your toplevel if you want to open any more
files!

To avoid this, we need to make sure that our code cleans up after itself. We
can do this using the `protect` function described in
[Error Handling](error-handling.html#error-handling){data-type=xref}, as
follows:

<link rel="import" href="code/imperative-programming/file2.mlt" part=
"1" />

And now, the file descriptor leak is gone:

<link rel="import" href="code/imperative-programming/file2.mlt" part=
"2" />

This is really an example of a more general issue with imperative programming
and exceptions. If you're changing the internal state of your program and
you're interrupted by an exception, you need to consider quite carefully if
it's safe to continue working from your current state.

`In_channel` has functions that automate the handling of some of these
details. For example, `In_channel.with_file` takes a filename and a function
for processing data from an `in_channel` and takes care of the bookkeeping
associated with opening and closing the file. We can rewrite `sum_file` using
this function, as shown here:

<link rel="import" href="code/imperative-programming/file2.mlt" part=
"3" />

Another misfeature of our implementation of `sum_file` is that we read the
entire file into memory before processing it. For a large file, it's more
efficient to process a line at a time. You can use the
`In_channel.fold_lines` function to do just that:

<link rel="import" href="code/imperative-programming/file2.mlt" part=
"4" />

This is just a taste of the functionality of `In_channel` and `Out_channel`.
To get a fuller understanding, you should review the API documentation for
those modules.<a data-type="indexterm" data-startref="IPinpout">&nbsp;</a>


## Order of Evaluation {#order-of-evaluation}

The order in which expressions are evaluated is an important part of the
definition of a programming language, and it is particularly important when
programming imperatively. Most programming languages you're likely to have
encountered are *strict*, and OCaml is too. In a strict language, when you
bind an identifier to the result of some expression, the expression is
evaluated before the variable is bound. Similarly, if you call a function on
a set of arguments, those arguments are evaluated before they are passed to
the function. [strict evaluation]{.idx}[expressions, order of
evaluation]{.idx}[evaluation, order of]{.idx}[order of
evaluation]{.idx}[imperative programming/order of evaluation]{.idx}

Consider the following simple example. Here, we have a collection of angles,
and we want to determine if any of them have a negative `sin`. The following
snippet of code would answer that question:

<link rel="import" href="code/imperative-programming/order.mlt" part=
"1" />

In some sense, we don't really need to compute the `sin 128.` because
`sin 75.` is negative, so we could know the answer before even computing
`sin 128.`.

It doesn't have to be this way. Using the `lazy` keyword, we can write the
original computation so that `sin 128.` won't ever be computed:

<link rel="import" href="code/imperative-programming/order.mlt" part=
"2" />

We can confirm that fact by a few well-placed `printf`s:

<link rel="import" href="code/imperative-programming/order.mlt" part=
"3" />

OCaml is strict by default for a good reason: lazy evaluation and imperative
programming generally don't mix well because laziness makes it harder to
reason about when a given side effect is going to occur. Understanding the
order of side effects is essential to reasoning about the behavior of an
imperative program.

Because OCaml is strict, we know that expressions that are bound by a
sequence of `let` bindings will be evaluated in the order that they're
defined. But what about the evaluation order within a single expression?
Officially, the answer is that evaluation order within an expression is
undefined. In practice, OCaml has only one compiler, and that behavior is a
kind of *de facto* standard. Unfortunately, the evaluation order in this case
is often the opposite of what one might expect.

Consider the following example:

<link rel="import" href="code/imperative-programming/order.mlt" part=
"4" />

Here, you can see that the subexpression that came last was actually
evaluated first! This is generally the case for many different kinds of
expressions. If you want to make sure of the evaluation order of different
subexpressions, you should express them as a series of `let` bindings.

## Side Effects and Weak Polymorphism {#side-effects-and-weak-polymorphism}

Consider the following simple, imperative function: [polymorphism/weak
polymorphism]{.idx}[weak polymorphism]{.idx}[side effects]{.idx}[ imperative
programming/side effects/weak polymorphism ]{.idx #IPsideweak}

<link rel="import" href="code/imperative-programming/weak.mlt" part="1" />

`remember` simply caches the first value that's passed to it, returning that
value on every call. That's because `cache` is created and initialized once
and is shared across invocations of `remember`.

`remember` is not a terribly useful function, but it raises an interesting
question: what is its type?

On its first call, `remember` returns the same value it's passed, which means
its input type and return type should match. Accordingly, `remember` should
have type `t -> t` for some type `t`. There's nothing about `remember` that
ties the choice of `t` to any particular type, so you might expect OCaml to
generalize, replacing `t` with a polymorphic type variable. It's this kind of
generalization that gives us polymorphic types in the first place. The
identity function, as an example, gets a polymorphic type in this way:

<link rel="import" href="code/imperative-programming/weak.mlt" part="2" />

As you can see, the polymorphic type of `identity` lets it operate on values
with different types.

This is not what happens with `remember`, though. As you can see from the
above examples, the type that OCaml infers for `remember` looks almost, but
not quite, like the type of the identity function. Here it is again:

<link rel="import" href="code/imperative-programming/remember_type.ml" />

The underscore in the type variable `'_a` tells us that the variable is only
*weakly polymorphic*, which is to say that it can be used with any *single*
type. That makes sense because, unlike `identity`, `remember` always returns
the value it was passed on its first invocation, which means its return value
must always have the same type. [type variables]{.idx}

OCaml will convert a weakly polymorphic variable to a concrete type as soon
as it gets a clue as to what concrete type it is to be used as:

<link rel="import" href="code/imperative-programming/weak.mlt" part="3" />

Note that the type of `remember` was settled by the definition of
`remember_three`, even though `remember_three` was never called!

### The Value Restriction {#the-value-restriction}

So, when does the compiler infer weakly polymorphic types? As we've seen, we
need weakly polymorphic types when a value of unknown type is stored in a
persistent mutable cell. Because the type system isn't precise enough to
determine all cases where this might happen, OCaml uses a rough rule to flag
cases that don't introduce any persistent mutable cells, and to only infer
polymorphic types in those cases. This rule is called
*the value restriction*. [value restriction]{.idx}

The core of the value restriction is the observation that some kinds of
expressions, which we'll refer to as *simple values*, by their nature can't
introduce persistent mutable cells, including:

- Constants (i.e., things like integer and floating-point literals)

- Constructors that only contain other simple values

- Function declarations, i.e., expressions that begin with `fun` or
  `function`, or the equivalent let binding, `let f x = ...`

- `let` bindings of the form `let`*`var`*`=`*`expr1`*`in`*`expr2`*, where
  both *`expr1`* and *`expr2`* are simple values

Thus, the following expression is a simple value, and as a result, the types
of values contained within it are allowed to be polymorphic:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"1" />

But, if we write down an expression that isn't a simple value by the
preceding definition, we'll get different results. For example, consider what
happens if we try to memoize the function defined previously.

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"2" />

The memoized version of the function does in fact need to be restricted to a
single type because it uses mutable state behind the scenes to cache values
returned by previous invocations of the function. But OCaml would make the
same determination even if the function in question did no such thing.
Consider this example:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"3" />

It would be safe to infer a fully polymorphic variable here, but because
OCaml's type system doesn't distinguish between pure and impure functions, it
can't separate those two cases.

The value restriction doesn't require that there is no mutable state, only
that there is no *persistent* mutable state that could share values between
uses of the same function. Thus, a function that produces a fresh reference
every time it's called can have a fully polymorphic type:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"4" />

But a function that has a mutable cache that persists across calls, like
`memoize`, can only be weakly polymorphic.

### Partial Application and the Value Restriction {#partial-application-and-the-value-restriction}

Most of the time, when the value restriction kicks in, it's for a good
reason, i.e., it's because the value in question can actually only safely be
used with a single type. But sometimes, the value restriction kicks in when
you don't want it. The most common such case is partially applied functions.
A partially applied function, like any function application, is not a simple
value, and as such, functions created by partial application are sometimes
less general than you might expect. [partial application]{.idx}

Consider the `List.init` function, which is used for creating lists where
each element is created by calling a function on the index of that element:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"5" />

Imagine we wanted to create a specialized version of `List.init` that always
created lists of length 10. We could do that using partial application, as
follows:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"6" />

As you can see, we now infer a weakly polymorphic type for the resulting
function. That's because there's nothing that guarantees that `List.init`
isn't creating a persistent `ref` somewhere inside of it that would be shared
across multiple calls to `list_init_10`. We can eliminate this possibility,
and at the same time get the compiler to infer a polymorphic type, by
avoiding partial application:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"7" />

This transformation is referred to as *eta expansion* and is often useful to
resolve problems that arise from the value restriction.

### Relaxing the Value Restriction {#relaxing-the-value-restriction}

OCaml is actually a little better at inferring polymorphic types than was
suggested previously. The value restriction as we described it is basically a
syntactic check: you can do a few operations that count as simple values, and
anything that's a simple value can be generalized.

But OCaml actually has a relaxed version of the value restriction that can
make use of type information to allow polymorphic types for things that are
not simple values.

For example, we saw that a function application, even a simple application of
the identity function, is not a simple value and thus can turn a polymorphic
value into a weakly polymorphic one:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"8" />

But that's not always the case. When the type of the returned value is
immutable, then OCaml can typically infer a fully polymorphic type:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"9" />

On the other hand, if the returned type is mutable, then the result will be
weakly polymorphic:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"10" />

A more important example of this comes up when defining abstract data types.
Consider the following simple data structure for an immutable list type that
supports constant-time concatenation:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"11" />

The details of the implementation don't matter so much, but it's important to
note that a `Concat_list.t` is unquestionably an immutable value. However,
when it comes to the value restriction, OCaml treats it as if it were
mutable:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"12" />

The issue here is that the signature, by virtue of being abstract, has
obscured the fact that `Concat_list.t` is in fact an immutable data type. We
can resolve this in one of two ways: either by making the type concrete
(i.e., exposing the implementation in the `mli`), which is often not
desirable; or by marking the type variable in question as *covariant*. We'll
learn more about covariance and contravariance in
[Objects](objects.html#objects){data-type=xref}, but for now, you can
think of it as an annotation that can be put in the interface of a pure data
structure. [datatypes/covariant]{.idx}

In particular, if we replace `type 'a t` in the interface with `type +'a t`,
that will make it explicit in the interface that the data structure doesn't
contain any persistent references to values of type `'a`, at which point,
OCaml can infer polymorphic types for expressions of this type that are not
simple values:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"13" />

Now, we can apply the identity function to `Concat_list.empty` without
without losing any polymorphism:

<link rel="import" href="code/imperative-programming/value_restriction.mlt" part=
"14" />


## Summary {#summary}

This chapter has covered quite a lot of ground, including: [imperative
programming/overview of]{.idx}

- Discussing the building blocks of mutable data structures as well as the
  basic imperative constructs like `for` loops, `while` loops, and the
  sequencing operator `;`

- Walking through the implementation of a couple of classic imperative data
  structures

- Discussing so-called benign effects like memoization and laziness

- Covering OCaml's API for blocking I/O

- Discussing how language-level issues like order of evaluation and weak
  polymorphism interact with OCaml's imperative features

The scope and sophistication of the material here is an indication of the
importance of OCaml's imperative features. The fact that OCaml defaults to
immutability shouldn't obscure the fact that imperative programming is a
fundamental part of building any serious application, and that if you want to
be an effective OCaml programmer, you need to understand OCaml's approach to
imperative
programming.<a data-type="indexterm" data-startref="PROGimper">&nbsp;</a>

