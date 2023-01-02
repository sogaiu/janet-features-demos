# Janet Features Demos

Some demos of [Janet features listed at the main
page](https://janet-lang.org/#Features), in order, on a Linux box.

Links to [official docs](https://janet-lang.org/docs/index.html)
should be provided along the way, highly recommended to check them out
to start the assimila^H^H^H^H^H^H^H^H familiarization process :)

Hint: if reading this on GitHub, consider using the Table of Contents
icon to navigate to different sections:

![GitHub Table of Contents Icon](github-table-of-contents-icon.png?raw=true "GitHub Table of Contents Icon")

The intent is to touch on the features listed at the main page.  This
is a work-in-progress.  Currently the list includes:

* Minimal setup - one binary and you are good to go!
* Builtin support for threads, networking, and an event loop
* First class closures
* Garbage collection
* First class green threads (continuations)
* Mutable and immutable arrays (array/tuple)
* Mutable and immutable hashtables (table/struct)
* Mutable and immutable strings (buffer/string)
* Macros
* Tail call optimization
* Direct interop with C via abstract types and C functions
* Dynamically load C libraries
* Lexical scoping
* REPL and interactive debugger
* Parsing Expression Grammars built in to the core library
* 500+ functions and macros in the core library
* Export your projects to standalone executables with a companion build tool, jpm
* Add to a project with just janet.c and janet.h

Note that this list is not quite the same as [the one in the
repository README](https://github.com/janet-lang/janet/#features).

Some additional items from the longer list:

* Configurable at build time - turn features on or off for a smaller or more featureful build
* Python-style generators (implemented as a plain macro)

## Minimal setup - one binary and you are good to go!

One way to experience this is to download one of the files from the
[GitHub releases page](https://github.com/janet-lang/janet/releases/).

At the time of this writing, for a Linux environment, I got
[janet-v1.25.1-linux-x64.tar.gz](https://github.com/janet-lang/janet/releases/download/v1.25.1/janet-v1.25.1-linux-x64.tar.gz),
uncompressed it, and was able to run the `janet` binary that lived in
the `bin` subdirectory:

```shell
wget --quiet https://github.com/janet-lang/janet/releases/download/v1.25.1/janet-v1.25.1-linux-x64.tar.gz
tar xf janet-v1.25.1-linux-x64.tar.gz
cd janet-v1.25.1-linux
./bin/janet
```

Personally, I prefer [a non-root local
install](https://janet-lang.org/docs/index.html#Non-root-install-(macOS-and-Unix-like))
of the master branch (later content in this document will be using the
local installation), but may be the earlier method is better if you
want a quick taste.  Some distributions carry packages, but I'm not
sure how up-to-date they are.

## Builtin support for threads, networking, and an event loop

### Threads

Here's a snippet from the [Multithreading
docs](https://janet-lang.org/docs/threads.html):

```janet
(ev/spawn-thread
  (print "New thread started!"))

(ev/do-thread
  (print "New thread started!"))
```

Trying that at the repl:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (ev/spawn-thread (print "Thread one"))
nil
repl:2:> Thread one

repl:3:> (ev/do-thread (print "Thread the second"))
Thread the second
nil
```

Not so clear in text perhaps -- I pressed the Enter key after the
`Thread one` text appeared for aesthetic purposes.

### Networking

Let's ask for some help from our snake friend:

```shell
mkdir /tmp/fun
cd /tmp/fun
python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Here's a snippet adapted from the [Networking docs](https://janet-lang.org/docs/networking.html):

```janet
(with [conn (net/connect "127.0.0.1" "8000" :stream)]
  (printf "Connected to %q!" conn)
  (:write conn "GET / HTTP/1.0\n\n")
  (print "Wrote to connection...")
  (def res (:read conn 1024))
  (pp res))
```

Trying that at the repl:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (with [conn (net/connect "127.0.0.1" "8000" :stream)]
repl:2:(>   (printf "Connected to %q!" conn)
repl:3:(>   (:write conn "GET / HTTP/1.0\n\n")
repl:4:(>   (print "Wrote to connection...")
repl:5:(>   (def res (:read conn 1024))
repl:6:(>   (pp res))
Connected to <core/stream 0x55B39596BF30>!
Wrote to connection...
@"HTTP/1.0 200 OK\r\nServer: SimpleHTTP/0.6 Python/3.10.6\r\nDate: Sat, 31 Dec 2022 01:37:29 GMT\r\nContent-type: text/html; charset=utf-8\r\nContent-Length: 607\r\n\r\n"
nil
```

[Servers are doable](https://janet-lang.org/docs/networking.html#Server) too.

### Event Loop

Here is some code from [The Event Loop docs](https://janet-lang.org/docs/event_loop.html):

```janet
(defn worker
  "Does some work."
  [name n]
  (for i 0 n
    (print name " working " i "...")
    (ev/sleep 0.5))
  (print name " is done!"))

# Start bob working in a new task with ev/call
(ev/call worker "bob" 10)

(ev/sleep 0.25)

# Start sally working in a new task with ev/go
(ev/go (fiber/new |(worker "sally" 20)))

(ev/sleep 11)
(print "Everyone should be done by now!")
```

Pasting the above to the repl a bit at a time.

First we make a function:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (defn worker
repl:2:(>   "Does some work."
repl:3:(>   [name n]
repl:4:(>   (for i 0 n
repl:5:((>     (print name " working " i "...")
repl:6:((>     (ev/sleep 0.5))
repl:7:(>   (print name " is done!"))
<function worker>
```

We pass the function to `ev/call` which gives us back a fiber.

```
repl:8:> (ev/call worker "bob" 10)
<fiber 0x55C44E732CC0>
```

To get it to start, we need to let it run by temporarily relinquishing
control via `ev/sleep`:

```
repl:9:> (ev/sleep 0.25)
bob working 0...
nil
```

Note the `bob working 0...`.

Now arrange for another:

```
repl:10:> (ev/go (fiber/new |(worker "sally" 20)))
<fiber 0x55C44E736470>
```

Relinquish control again:

```
repl:11:> (ev/sleep 11)
sally working 0...
bob working 1...
sally working 1...
bob working 2...
sally working 2...
bob working 3...
sally working 3...
bob working 4...
sally working 4...
bob working 5...
sally working 5...
bob working 6...
sally working 6...
bob working 7...
sally working 7...
bob working 8...
sally working 8...
bob working 9...
sally working 9...
bob is done!
sally working 10...
sally working 11...
sally working 12...
(print "Everyone should be done by now!")sally working 13...
sally working 14...
sally working 15...
sally working 16...
sally working 17...
sally working 18...
sally working 19...
sally is done!
nil
```

I was able to paste `(print "Everyone should be done by now!")` while
output was appearing from ongoing activity.  Notice how it appears
between the text `sally working 12...` and `sally working 13...`.

## First class closures

We'll make an "adder" maker:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (defn adder-maker [x] (fn [y] (+ x y)))
<function adder-maker>
```

Now use it to make an adder and call it:

```
repl:2:> (def eight-adder (adder-maker 8))
<function 0x5648F3BF95B0>
repl:3:> (eight-adder 1)
9
```

Finally, pass the adder to another function for use:


```
repl:4:> (defn use-adder-to-increment [an-adder value] (an-adder value))
<function use-adder-to-increment>
repl:5:> (use-adder-to-increment eight-adder 3)
11
```

## Garbage collection

Not sure how to demonstrate this...how about some
[evidence](https://github.com/janet-lang/janet/blob/master/src/core/gc.c)?

The [Janet's Memory Model
docs](https://janet-lang.org/capi/memory-model.html) have some details.

## First class green threads (continuations)

In Janet, I believe these are called "fibers".

Adapting some code from the beginning of the [Fibers
docs](https://janet-lang.org/docs/fibers/index.html):

```janet
(def f
  (fiber/new (fn []
               (yield 1)
               (yield 2)
               (yield 3)
               (yield 4)
               5)))

(fiber/status f)

(resume f)

(resume f)
(resume f)
(resume f)

(fiber/status f)

(resume f)

(fiber/status f)

(resume f)
```

Stepping through:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def f
repl:2:(>   (fiber/new (fn []
repl:3:(((>                (yield 1)
repl:4:(((>                (yield 2)
repl:5:(((>                (yield 3)
repl:6:(((>                (yield 4)
repl:7:(((>                5)))
<fiber 0x55D01BE6D050>
```

Can get the status of a fiber:

```
repl:7:> (fiber/status f)
:new
```

Resume the fiber ("resume" also means start):

```
repl:8:> (resume f)
1
```

Resume 3 more times:

```
repl:9:> (resume f)
2
repl:10:> (resume f)
3
repl:11:> (resume f)
4
```

Check the fiber's status:
```
repl:12:> (fiber/status f)
:pending
```

Resume it again:
```
repl:13:> (resume f)
5
```

Check the status again:
```
repl:14:> (fiber/status f)
:dead
```

What if `resume` is called now?
```
repl:15:> (resume f)
error: cannot resume fiber with status :dead
  in _thunk [repl] (tailcall) on line 15, column 1
```

Oops :)

See the [Fibers docs](https://janet-lang.org/docs/fibers/index.html)
and the [Fiber Module docs](https://janet-lang.org/api/fiber.html) for
more info.

## Mutable and immutable arrays (array/tuple)

There are two kinds of array-like data structures in Janet, mutable
(array) and immutable (tuple).

### Mutable Arrays aka Arrays

The [Arrays
docs](https://janet-lang.org/docs/data_structures/arrays.html) cover
mutable arrays.  In Janet, "array" refers to the mutable type of
array data structure.

Mutability is typically expressed using a leading `@` character:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def an-array @[1 3 8])
@[1 3 8]
```

Parentheses can be used as well, but this does not appear common:

```
repl:2:> (def another-array @(2 7 9))
@[2 7 9]
```

Let's add something:

```
repl:3:> (array/push an-array 2)
@[1 3 8 2]
```

Now get something back:

```
repl:4:> (get an-array 0)
1
```

...in a number of other ways:

```
repl:5:> (in an-array 0)
1
repl:6:> (an-array 0)
1
repl:7:> (0 an-array)
1
```

[`get`](https://janet-lang.org/api/index.html#get) and
[`in`](https://janet-lang.org/api/index.html#in) are ordinary
functions.

See the [Array
docs](https://janet-lang.org/docs/data_structures/arrays.html) and the
[Array Module docs](https://janet-lang.org/api/array.html) for more
info.

### Immutable Arrays aka Tuples

The [Tuples
docs](https://janet-lang.org/docs/data_structures/tuples.html) cover
imumutable arrays.  In Janet, "tuple" refers to an immutable type of
array data structure.  Note that this is not a "persistent" data
structure as in Clojure.

Let's make a tuple:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def a-tuple [2 7 9])
(2 7 9)
```

It's also possible to make a tuple "directly" using parentheses, but
quoting is necessary:

```
repl:2:> (def another-tuple '(3 8 0))
(3 8 0)
```

Retrieval can be done like:

```
repl:3:> (get a-tuple 0)
2
```

or in a number of other ways:


```
repl:4:> (in a-tuple 0)
2
repl:5:> (a-tuple 0)
2
repl:6:> (0 a-tuple)
2
```

[`get`](https://janet-lang.org/api/index.html#get) and
[`in`](https://janet-lang.org/api/index.html#in) are ordinary
functions.

See the [Tuples
docs](https://janet-lang.org/docs/data_structures/tuples.html) and the
[Tuple Module docs](https://janet-lang.org/api/tuple.html) for more
info.

## Mutable and immutable hashtables (table/struct)

There are two kinds of hash table / associative array-like data
structures in Janet, mutable (table) and immutable (struct).

### Tables

The [Tables
docs](https://janet-lang.org/docs/data_structures/tables.html) cover
mutable hash tables.  In Janet, "table" refers to the mutable type of
hash table data structure.

Mutability is typically expressed using a leading `@` character:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def a-table @{:a 1 :b 2})
@{:a 1 :b 2}
```
Let's add something:

```
repl:2:> (put a-table :c 3)
@{:a 1 :b 2 :c 3}
repl:3:> a-table
@{:a 1 :b 2 :c 3}
```

To get something out:

```
repl:4:> (get a-table :a)
1
```

This can also be accomplished by:

```
repl:5:> (in a-table :a)
1
repl:6:> (a-table :a)
1
```

Note that the following doesn't work in the same way as for arrays and
tuples:

```
repl:7:> (:a a-table)
nil
```

For a hint as to why this is, see the [Object-Oriented Programming
docs](https://janet-lang.org/docs/object_oriented.html).  In short,
`(:a a-table)` becomes `((get a-table :a) a-table)` which in turn
leads to `(1 a-table)`.

So if `1` were added to the table as a key with an associated value:

```
repl:8:> (put a-table 1 :surprise)
@{1 :surprise :a 1 :b 2 :c 3}
```

and another attempt is made, we would get:

```
repl:9:> (:a a-table)
:surprise
```

[`get`](https://janet-lang.org/api/index.html#get) and
[`in`](https://janet-lang.org/api/index.html#in) are ordinary
functions.

See the [Tables
docs](https://janet-lang.org/docs/data_structures/tables.html) and the
[Table Module docs](https://janet-lang.org/api/table.html) for more
info.

### Structs

The [Structs
docs](https://janet-lang.org/docs/data_structures/structs.html) cover
immutable hash tables.  In Janet, "struct" refers to an immutable type
of hash table data structure.  Note that this is not a "persistent"
data structure as in Clojure.

Let's make a struct:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def a-struct {:a 1})
{:a 1}
```

Retrieval can be done like this:

```
repl:2:> (get a-struct :a)
1
```

or by either of:

```
repl:3:> (in a-struct :a)
1
repl:4:> (a-struct :a)
1
```

Note that the following doesn't work in the same way as for arrays and
tuples:

```
repl:5:> (:a a-struct)
nil
```

For a hint as to why this is, see the [Object-Oriented Programming
docs](https://janet-lang.org/docs/object_oriented.html).  In short,
`(:a a-struct)` becomes `((get a-struct :a) a-struct)` which in turn
leads to `(1 a-struct)`.

If `a-struct` had been defined to contain a key of value `1` with an
associated value:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def a-struct {:a 1 1 :relax})
{1 :relax :a 1}
```

and an attempt similar to the earlier one is made, we should get:

```
repl:2:> (:a a-struct)
:relax
```

[`get`](https://janet-lang.org/api/index.html#get) and
[`in`](https://janet-lang.org/api/index.html#in) are ordinary
functions.

See the [Structs
docs](https://janet-lang.org/docs/data_structures/structs.html) and
the `struct`-related calls starting at [the docs for
struct](https://janet-lang.org/api/index.html#struct) for more info.

## Mutable and immutable strings (buffer/string)

There are two kinds of string-like data structures in Janet, mutable
(buffer) and immutable (string).

### Buffers

The [Buffers
docs](https://janet-lang.org/docs/data_structures/buffers.html) cover
mutable strings.  In Janet, "buffer" refers to a mutable type of
string data structure.

Mutability is typically expressed using a leading `@` character:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def a-buffer @"hello")
@"hello"
```

Let's add a character to the end of the buffer:

```
repl:2:> (put a-buffer (length a-buffer) 33)
@"hello!"
```

How about multiple characters at once?

```
repl:3:> (buffer/push-string a-buffer " world?")
@"hello! world?"
```

Retrieval of an individual character can be done by:

```
repl:4:> (get a-buffer 5)
33
repl:5:> (chr "1")
33
```

Or:

```
repl:6:> (in a-buffer 5)
33
repl:7:> (a-buffer 5)
33
repl:8:> (5 a-buffer)
33
```

A "sub" buffer (though it's a different buffer) can be retrieved by:

```
repl:9:> (buffer/slice a-buffer 7)
@"world?"
```

[`get`](https://janet-lang.org/api/index.html#get) and
[`in`](https://janet-lang.org/api/index.html#in) are ordinary
functions.

See the [Buffers
docs](https://janet-lang.org/docs/data_structures/buffers.html) and the
[Buffer Module docs](https://janet-lang.org/api/buffer.html) for more
info.

### Strings


The [Strings docs](https://janet-lang.org/docs/strings.html#Strings)
cover immutable strings.  In Janet, "string" refers to the immutable
type of string as in a number of other programming languages.

One can make a string by:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def a-string "sit")
"sit"
```

To get a character:

```
repl:2:> (get a-string 2)
116
repl:3:> (chr "t")
116
```

Or:

```
repl:4:> (in a-string 2)
116
repl:5:> (a-string 2)
116
repl:6:> (2 a-string)
116
```

Substrings can be retrieved by:

```
repl:7:> (string/slice a-string 1)
"it"
repl:8:> (string/slice a-string 1 2)
"i"
```

[`get`](https://janet-lang.org/api/index.html#get) and
[`in`](https://janet-lang.org/api/index.html#in) are ordinary
functions.

See the [Strings
docs](https://janet-lang.org/docs/strings.html#Strings) and the
[String Module docs](https://janet-lang.org/api/string.html) for more
info.

## Macros

Adapted from the [Macros
docs](https://janet-lang.org/docs/macros.html):

```janet
(defmacro my-defn
  "Defines a new function."
  [name args & body]
  ~(def ,name (fn ,name ,args ,;body)))
```

Let's try it out.

Define the macro first:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (defmacro my-defn
repl:2:(>   "Defines a new function."
repl:3:(>   [name args & body]
repl:4:(>   ~(def ,name (fn ,name ,args ,;body)))
<function my-defn>
```

Now call it:

```
repl:5:> (my-defn my-name [x] (+ x 1))
<function my-name>
repl:6:> (my-name 3)
4
```

Note that unlike Common Lisp, Scheme, or Clojure, Janet uses `~` for
quasi-quoting / syntax-quoting.  There is a corresponding special form
named `quasiquote`.

`;` is an abbreviation for using the `splice` special form.

`,` is an abbreviation for using the `unquote` special form.

See the [Special Forms
docs](https://janet-lang.org/docs/specials.html) for more details about
`~` (`quasiquote`), `;` (`splice`) , and `,` (`unquote`).

For more info on macros, see the [Macros
docs](https://janet-lang.org/docs/macros.html).

## Tail call optimization

As a placeholder, here is some evidence:

* https://github.com/janet-lang/janet/blob/a8a78d452506fdbfbf877379eba969e4efbef842/src/core/vm.c#L379
* https://github.com/janet-lang/janet/blob/a8a78d452506fdbfbf877379eba969e4efbef842/src/core/vm.c#L1008-L1052
* https://github.com/janet-lang/janet/blob/a8a78d452506fdbfbf877379eba969e4efbef842/src/core/vm.c#L270-L276
* https://github.com/janet-lang/janet/blob/a8a78d452506fdbfbf877379eba969e4efbef842/src/core/vm.c#L606-L616

## Direct interop with C via abstract types and C functions

See [The Janet C API docs](https://janet-lang.org/capi/index.html) for
more details.

## Dynamically load C libraries

There may be more than one way to interpret what this means.  Since
this item has been in the list longer than the recent addition of
Janet's FFI capability, perhaps it didn't use to refer to FFI.

### Non-FFI

I think this refers to Janet's native module capability.

See the [Writing a Module
section](https://janet-lang.org/capi/index.html#Writing-a-Module) of
[The Janet C API docs](https://janet-lang.org/capi/index.html), the
[Writing C Functions
docs](https://janet-lang.org/capi/writing-c-functions.html), and the
[Wrapping Types docs](https://janet-lang.org/capi/wrapping.html) for
details.

### FFI

See the [gtk
example](https://github.com/janet-lang/janet/blob/a8a78d452506fdbfbf877379eba969e4efbef842/examples/ffi/gtk.janet).

I had to tweak the location of `libgtk-3.so` in `gtk.janet` to be:

```
/usr/lib/x86_64-linux-gnu/libgtk-3.so
```

but after that, I had success via the invocation:

```
janet gtk.janet
```

See the [Foreign Function Interface
docs](https://janet-lang.org/docs/ffi.html) and the [FFI Module
docs](https://janet-lang.org/api/ffi.html) for more information.

## Lexical scoping

Does the following count as evidence?

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def a 2)
2
repl:2:> (defn b [] (def a 3) a)
<function b>
repl:3:> (b)
3
repl:4:> a
2
```

## REPL and interactive debugger

### REPL

Calling `janet` at the command line gives a repl:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> 1
1
```

There is a sometimes-helpful delimiter reminder feature:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (defn my-fn
repl:2:(>   [x]
repl:3:(>   (+ x 1
repl:4:((> )
repl:5:(> )
<function my-fn>
```

Note that in the repl's prompt after the last colon `:` character and
before the greater-than `>` character, there are one or more opening
delimiters in some of the above lines:

```
repl:3:(>   (+ x 1
repl:4:((> )
```

These are reminders that there are unmatched opening delimiters
waiting to be closed.

Simple completion is possible via the Tab key:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (do
do            doc           doc*          doc-format    doc-of
dofile
```

To get the above output, I pressed the Tab key after typing "(do".

There is a way to quickly see docstrings for various things:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> array/push


    cfunction
    src/core/array.c on line 179, column 1

    (array/push arr x)

    Insert an element in the end of an array. Modifies the input array and
    returns it.
```

After typing "array/push", I entered the sequence Ctrl-G (holding the
Ctrl key down and pressing and releasing the G key).

Some other key sequences such as Ctrl-A, Ctrl-E, etc. behave in similar
ways to what one might use in a shell such as bash.

### Interactive Debugger

One way to get access to the bytecode debugger is to use the `-d`
command line option to `janet`, then use the `debug` function:

```
$ janet -d
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (debug)
debug:
  in _thunk [repl] (tailcall) on line 1, column 1
entering debug[1] - (quit) to exit
debug[1]:1:> (quit)
nil
exiting debug[1]
nil
```

Putting a call to `debug` in a function and calling that function can
be handy too:

```
$ janet -d
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (defn my-fn [x] (debug) (+ x 1))
<function my-fn>
repl:2:> (my-fn 8)
debug:
  in my-fn [repl] on line 1, column 17
  in _thunk [repl] (tailcall) on line 2, column 1
entering debug[1] - (quit) to exit
```

We can get a bytecode listing via `.ppasm`:

```
debug[1]:1:> (.ppasm)

  signal:
  status:     debug
  function:   my-fn [repl]
  constants:  @[]
  slots:      @[8 <function my-fn> nil nil]

   lds 1                # line 1, column 1
   ldn 3                # line 1, column 17
 > sig 2 3 2
   addim 3 0 1          # line 1, column 25
   ret 3

nil
```

Execute one bytecode instruction via `.step`:
```
debug[1]:2:> (.step)
nil
debug[1]:3:> (.ppasm)

  signal:
  status:     debug
  function:   my-fn [repl]
  constants:  @[]
  slots:      @[8 <function my-fn> nil nil]

   lds 1                # line 1, column 1
   ldn 3                # line 1, column 17
   sig 2 3 2
 > addim 3 0 1          # line 1, column 25
   ret 3

nil
```

...and keep going to see the function return:

```
debug[1]:4:> (.step)
nil
debug[1]:5:> (.ppasm)

  signal:
  status:     debug
  function:   my-fn [repl]
  constants:  @[]
  slots:      @[8 <function my-fn> nil 9]

   lds 1                # line 1, column 1
   ldn 3                # line 1, column 17
   sig 2 3 2
   addim 3 0 1          # line 1, column 25
 > ret 3

nil
debug[1]:6:> (.step)
9
```

Note that the value `9` is the return value of the function call.

When we're done we can exit the debugger to return to the original
repl context:

```
debug[1]:7:> (quit)
nil
exiting debug[1]
```

See [this part of
boot.janet](https://github.com/janet-lang/janet/blob/a8a78d452506fdbfbf877379eba969e4efbef842/src/boot/boot.janet#L3353-L3507)
for which functions are available for the debugger and [The Janet
Abstract Machine
docs](https://janet-lang.org/docs/abstract_machine.html) for more on
Janet bytecode and bytecode interpreter.

Also of interest might be the [Debug Module
docs](https://janet-lang.org/api/debug.html).

## Parsing Expression Grammars built in to the core library

From the [Parsing Expression Grammars docs](https://janet-lang.org/docs/peg.html) is:

```janet
(def ip-address
 '{:dig (range "09")
   :0-4 (range "04")
   :0-5 (range "05")
   :byte (choice
           (sequence "25" :0-5)
           (sequence "2" :0-4 :dig)
           (sequence "1" :dig :dig)
           (between 1 2 :dig))
   :main (sequence :byte "." :byte "." :byte "." :byte)})
```

Note that this expression refers to a plain Janet struct.

Let's define it at the repl:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> (def ip-address
repl:2:(>  '{:dig (range "09")
repl:3:({>    :0-4 (range "04")
repl:4:({>    :0-5 (range "05")
repl:5:({>    :byte (choice
repl:6:({(>            (sequence "25" :0-5)
repl:7:({(>            (sequence "2" :0-4 :dig)
repl:8:({(>            (sequence "1" :dig :dig)
repl:9:({(>            (between 1 2 :dig))
repl:10:({>    :main (sequence :byte "." :byte "." :byte "." :byte)})
{:0-4 (range "04") :0-5 (range "05") :byte (choice (sequence "25" :0-5) (sequence "2" :0-4 :dig) (sequence "1" :dig :dig) (between 1 2 :dig)) :dig (range "09") :main (sequence :byte "." :byte "." :byte "." :byte)}
```

Typically one might use this via the function `peg/match`:

```
repl:11:> (peg/match ip-address "1")
nil
```

A `nil` return value means there was no match.

For a match:

```
repl:12:> (peg/match ip-address "127.0.0.1")
@[]
```

One gets back an array of captures.  In this case there was a match,
but nothing was captured, so the array is empty.

For an example of a capturing PEG expression, try:

```
repl:13:> (peg/match ~(capture 3) "127.0.0.1")
@["127"]
```

The first 3 characters were captured as a unit and returned as an
element of the "capture stack".

A slightly more complicated example is:

```
repl:14:> (def p ~(sequence (any :s) (capture (some :d)) (any :s) (capture (some :d))))
(sequence (any :s) (capture (some :d)) (any :s) (capture (some :d)))
repl:15:> (peg/match p "   12 89.")
@["12" "89"]
```

The PEG might be more readily perceived as:

```janet
(def p
  ~(sequence (any :s)
             (capture (some :d))
             (any :s)
             (capture (some :d))))
```

Above, initial whitespace is skipped (matched but not captured), a
sequence of digits is captured, more whitespace is skipped and then
finally a sequence of digts is captured again.

Note that there are 2 captures on the capture stack "12" and "89" and
the trailing period has not been captured.

Also note that the definition of `ip-address` from earlier can be
condensed (and the key-value pairs reordered) as follows:

```janet
(def ip-address
 '{:main (* :byte "." :byte "." :byte "." :byte)
   :byte (+ (* "25" :0-5)
            (* "2" :0-4 :d)
            (* "1" :d :d)
            (between 1 2 :d))
   :0-5 (range "05")
   :0-4 (range "04")})
```

`*` is an alias for `sequence`.

`+` is an alias for `choice`.

`:d` is defined in `default-peg-grammar`...along with a number of other things:

```
$ janet
Janet 1.26.0-dev-a8a78d45 linux/x64 - '(doc)' for help
repl:1:> default-peg-grammar
@{:A (if-not :a 1) :D (if-not :d 1) :H (if-not :h 1) :S (if-not :s 1) :W (if-not :w 1) :a (range "az" "AZ") :a* (any :a) :a+ (some :a) :d (range "09") :d* (any :d) :d+ (some :d) :h (range "09" "af" "AF") :h* (any :h) :h+ (some :h) :s (set " \t\r\n\0\f\v") :s* (any :s) :s+ (some :s) :w (range "az" "AZ" "09") :w* (any :w) :w+ (some :w)}
```

FWIW, [its definition in
`boot.janet`](https://github.com/janet-lang/janet/blob/a8a78d452506fdbfbf877379eba969e4efbef842/src/boot/boot.janet#L2221-L2243)
is prettier.

Especially while learning I recommend using the longer names as:

* some of the aliases might be confusing if you are used to regular
  expressions (I'm looking at you `*` and `+` -- though I take it
  these are based on those from Lua's LPEG), and

* if you return to looking at Janet code after not having looked in a while
  you might not immediately recall the aliases

See the [Parsing Expression Grammars
docs](https://janet-lang.org/docs/peg.html) and the [PEG
Module](https://janet-lang.org/api/peg.html) for official docs.

Highly recommended is pyrmont's excellent [How-To: Using PEGs in
Janet](https://articles.inqk.net/2020/09/19/how-to-use-pegs-in-janet.html)
article.

For simple examples and docs, follow individual links at [this listing
 of the PEG
 specials](https://github.com/sogaiu/margaret#specials-implementation-status)
 along with [some extracted real-world
 usage](https://github.com/sogaiu/margaret/tree/master/samples/pegs).

Yes, that last bit is a shameless plug :)

## 500+ functions and macros in the core library

See the [Core API docs](https://janet-lang.org/api/index.html), but
also [JanetDocs](https://janetdocs.com/), a community documentation
site.

## Export your projects to standalone executables with a companion build tool, jpm

Install [jpm](https://github.com/janet-lang/jpm) first.

Create a new project:

```shell
cd /tmp
jpm new-project yo
```

Answer some questions (if you want):

```shell
author? yours truly
description? yo
creating project directory for yo
```

Look inside:

```shell
cd yo
tree yo
yo
├── bin
├── CHANGELOG.md
├── LICENSE
├── project.janet
├── README.md
├── test
│   └── basic.janet
└── yo
    └── init.janet

3 directories, 6 files
```

Make an executable with jpm's `quickbin` subcommand:

```shell
jpm quickbin yo/init.janet executable-yo
generating executable c source executable-yo.c from yo/init.janet...
compiling executable-yo.c to build/executable-yo.o...
linking executable-yo...
```

Try it out:

```shell
./executable-yo
Hello!
```

Get curious:

```shell
cat yo/init.janet
```

```janet
(defn hello
  `Evaluates to "Hello!"`
  []
  "Hello!")

(defn main
  [& args]
  (print (hello)))
```

See the [jpm docs](https://janet-lang.org/docs/jpm.html), the [jpm
repository](https://github.com/janet-lang/jpm), and the generated
[JPM](https://janet-lang.org/api/jpm/index.html) reference for more info.


## Add to a project with just janet.c and janet.h

Some examples of doing this include:

* The [janet-lang.org website](https://github.com/janet-lang/janet-lang.org/)
* ahungry's [puny-gui](https://github.com/ahungry/puny-gui/) - a small cross-platform (native) GUI setup (GNU/Linux + Windows)
* ianthehenry's [bauble](https://github.com/ianthehenry/bauble) - for composing signed distance functions in a high-level language (Janet), compiling them to GLSL, and rendering them via WebGL
* MikeBeller's [janet-playground](https://github.com/MikeBeller/janet-playground) - A WebAssembly based playground for Janet
* s5bug's [sys-script](https://github.com/s5bug/sys-script) - Controlling the Nintendo Switch with Lisp
* sedpisoad's [super-janet-typist](https://github.com/sepisoad/super-janet-typist/) - a short typing game made with janet lisp
* [jaylib-wasm-demo](https://github.com/sogaiu/jaylib-wasm-demo) - A demo of using jaylib in a web browser

See [The Janet C API docs](https://janet-lang.org/capi/index.html) and
the [Embedding docs](https://janet-lang.org/capi/embedding.html) for
more details.

---

## Additional Items

* Configurable at build time - turn features on or off for a smaller or more featureful build

    See the [Configuration docs](https://janet-lang.org/capi/configuration.html).

* Python-style generators (implemented as a plain macro)

    See the [generate](https://janet-lang.org/api/index.html#generate) macro and at
    least [one example at JanetDocs](https://janetdocs.com/generate).


