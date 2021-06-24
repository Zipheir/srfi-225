## Abstract

The procedures of this SRFI allow callers to
manipulate an object that maps keys to values
without the caller needing to know exactly what the type
of the object is.  Such an object is called a *dictionary* in this SRFI.

## Rationale

Until recently there was only one universally available mechanism for managing
key-value pairs: alists.  Most Schemes also support hash tables, but
until R6RS there was no standard interface to them, and many
implementations do not provide that interface.

Now, however, the number of such mechanisms is
growing.  In addition to both R6RS and R7RS hash tables, there are
persistent ordered and hashed mappings from SRFI 146
and ordered key-value stores (often on a disk or a remote machine)
from SRFI 167.

It's inconvenient for users if SRFIs or other libraries have to
insist on accepting only a specific type of dictionary.
This SRFI exposes
a number of accessors, mutators, and other procedures that can be
called on any dictionary, provided that its type has been registered
with an implementation of this SRFI.

This in turn requires that the dictionary type provides a predicate
that can recognize it, plus at least these primitive operations:
determine a dictionary's current size; reference, update, or insert
an element of the dictionary depending on its current contents;
map over all the elements with
a function mapping the old value to a new one;
filter the elements based on their keys or values;
and process all the elements using a side-effecting procedure.

By using the procedures of this SRFI, a procedure can take a dictionary
as an argument and make flexible use of it without knowing its exact type.

Note that dictionaries must still be constructed using type-specific
constructors, as the required and optional arguments differ in each case.

## Specification

In order for the system to know that an object is a dictionary,
a predicate must be defined that recognizes that type of dictionary.
Then the predicate must be registered along with procedures that know
how to manipulate the dictionary.  At least the six basic dictionary procedures
(see below) must be registered, but more may be provided at registration time.

We call a specific key-value pair an *association*.  This is
why an alist, or association list, is called that; it is a list
of associations represented as pairs.

When a key argument is said to be the same as some key of the dictionary,
it means that they are the same in the sense of the dictionary's equality predicate.
It is assumed that no dictionary contains two keys that are the same in this sense.

Dictionaries are said in this SRFI to be *similar* if they are of the same
type and have the same [SRFI 128](http://srfi.schemers.org/srfi-128/srfi-128.html)
comparator.

## Lists as dictionaries

The exact set of pre-registered dictionaries depends on their
availability in a given implementation.  However, lists are
supported as dictionaries using the specification in this section.
If two keys are the same (in the sense of the specified equality predicate),
then all but the first are treated as if they did not exist.

If the car of a list is a symbol, then the list is assumed to be a property
list, alternating symbol keys with values.
Mutation operations actually mutate the property list whenever possible.
The equality predicate of this type of dictionary is `eq?`.

If a list is empty, or its car is a pair, then the list is assumed
to be an alist.  New values are added to the beginning of an alist
and the new alist is returned;
deletion does not mutate the alist, but returns an alist
that may or may not share storage with the original alist.
If an association has been updated, then both the new and the old
association may be processed by the whole-dictionary procedures.
The equality predicate of this type of dictionary is `equal?`.
The examples in this SRFI use alists.

In all other cases, lists are not treated as dictionaries
unless an appropriate dictionary type has been registered.

## Predicates

`(dictionary? `*obj*`)`

Returns `#t` if *obj* answers `#t` to some registered predicate,
and `#f` otherwise.

```
(define dict '((1 . 2) (3 . 4) (5 . 6)))
(dictionary? dict) => #t
```

`(dict-empty? `*dictionary*`)`

Returns `#t` if *dictionary* contains no associations
and `#f` if it does contain associations.

```
(dict-empty? '()) => #t
(dict-empty? dict) => #f
```

`(dict-contains? `*dictionary key*`)`
```
(dict-contains? dict 1) => #t
(dict-contains? dict 2) => #f
```

Returns `#t `if one of the keys of *dictionary* is the same as *key*
and `#f` otherwise.

## Lookup

`(dict-ref 	`*dictionary key* [*failure* [*success*] ]`)`

If *key* is the same as
some key of *dictionary*,
then invokes *success* on the corresponding value and returns its result.
If *key* is not a key of *dictionary*,
then invokes the thunk *failure* and returns its result.
The default value of *failure* signals an error;
the default value of *success* is the identity procedure.

```
(dict-ref dict 1 (lambda () '() list) => (1)  ; success wraps value in a list
(dict-ref dict 2 (lambda () '() list)) => ()  ; failure returns empty list
```

`(dict-ref/default `*dictionary key default*`)`

If *key* is the same as some key of *dictionary*
then returns the corresponding value.
If not, then returns *default*.

```
(dict-ref/default dict 1 #f) => 1
(dict-ref/default dict 1 #f) => #f
```

## Mutation

All these procedures are linear-update: that is, they may return a new
dictionary object (which may or may not share storage with the *dictionary*
argument), or the same dictionary object, mutated.  In either case,
it is an error to access the dictionary later through any other reference to it,
as that reference may have been invalidated.

`(dict-set! `*dictionary obj* ...`)`

Returns a dictionary that contains all the associations of *dictionary*
plus those specified by *objs*, which alternate between keys and values.
If a key to be added already exists in *dictionary*, the new value prevails.

```
; alists are changed non-destructively
(dict-set! dict 7 8) => ((7 . 8) (1 . 2) (3 . 4) (5 . 6))
(dict-set! dict 3 5) => ((1 . 2) (3 . 5) (5 . 6) ; may share last alist entry
```

`(dict-adjoin! `dictionary obj ...`)`

Returns a dictionary that contains all the associations of *dictionary*
plus those specified by *objs*, which alternate between keys and values.
If a key to be added already exists in *dictionary*, the old value prevails.

```
; alists are changed non-destructively
(dict-adjoin! dict 7 8) => ((7 . 8) (1 . 2) (3 . 4) (5 . 6))
(dict-adjoin! dict 3 5) => ((1 . 2) (3 . 5) (5 . 6) ; may share last alist entry
```

`(dict-delete! `*dictionary key* ...`)`

Returns a dictionary that contains all the associations of *dictionary* except those
whose keys are the same as one of the *keys*.

```
; alists are changed non-destructively
(dict-delete! dict 1 3) => ((7 . 8) (1 . 2) (3 . 4) (5 . 6))
(dict-delete! dict 3 5) => ((1 . 2) (3 . 4) (5 . 6) ; may share whole alist
```

`(dict-delete-all! `*dictionary keylist*`)`

Returns a dictionary with all the associations of *dictionary* except those whose
keys are the same as some member of *keylist*.

```
(dict-delete-all! dict '(1 3)) => ((5 . 6))
```

`(dict-replace! `*dictionary key value*`)`

Returns a dictionary that
contains all the associations of *dictionary* except as follows:
If *key* is the same as a key of *dictionary*,
then the association for that key is omitted and replaced by the association
defined by the pair *key* and *value*.
If there is no such key in *dictionary*,
then dictionary is returned unchanged.

```
(dict-replace! dict 1 3) => ((1 . 3) (3 . 4) (5 . 6))
(dict-replace! dict 2 3) => ((1 . 2) (3 . 4) (5 . 6))
```

`(dict-intern! `dictionary key failure`)`

Extracts the value associated with the key in *dictionary* that is the same as *key*,
and returns two values, *dictionary* and the value.
If *key* is not the same as any key in *dictionary*, *failure* is invoked on no arguments.

The procedure then returns two values,
a dictionary that contains all the associations of *dictionary*
and in addition a new association that maps *key* to the result of invoking *failure*, 
and the result of invoking *failure*.

```
(dict-intern! dict 1 (lambda () #f)) => ; 2 values
  ((1 . 2) (3 . 4) (5 . 6))
  3
(dict-intern! dict 2 (lambda () #f)) => ; 2 values
  ((2 . #f) (1 . 2) (3 . 4) (5 . 6))
  #f
```

`(dict-update! `*dictionary key updater* [*failure* [*success*] ]`)`

Retrieves the value of *key* as if by `dict-ref`,
invokes *updater* on it, and sets the value of *key* to be
the result of calling *updater* as if by `dict-set`,
but may do so more efficiently.  Returns the updated dictionary.
The default value of *failure* signals an error;
the default value of *success* is the identity procedure.

`(dict-update/default! `*dictionary key updater default*`)`

Retrieves the value of *key* as if by `dict-ref/default`,
invokes *updater* on it, and sets the value of *key* to be
the result of calling *updater* as if by `dict-set`,
but may do so more efficiently.  Returns the updated dictionary.

`(dict-pop! `*dictionary*`)`

Chooses an  association from *dictionary* and returns three values:
a dictionary that contains all associations of *dictionary* except the chosen one,
and the key and the value of the chosen association.
If the dictionary is ordered, the first association is chosen;
otherwise the chosen association is arbitrary.

If dictionary contains no associations and *failure* is supplied,
it is an error.

```
(dict-pop! dict) => # 3 values
  ((3 . 4) (5 . 6))
  1
  2
```

`(dict-map! `*proc dictionary*`)`

Returns a dictionary similar to *dictionary* that maps each key of *dictionary*
to the value that results
from invoking *proc* on the corresponding key and value of *dictionary*.

```
(dict-map! (lambda (k v) (cons v k)) dict) => ((2 . 1) (4 . 3) (6 . 5))
```

`(dict-filter! `*pred dictionary*`)`  

Returns a dictionary similar to *dictionary* that contains just the associations of *dictionary*
that satisfy *pred* when it is invoked on the key and value of the association.

```
(dict-filter (lambda (x) (= x 1)) dict) => ((1 . 2))
```

`(dict-remove! `*pred dictionary*`)`

Returns a dictionary that contains all the associations of *dictionary*
except those that satisfy *pred* when called on the key and value.

```
(dict-remove (lambda (x) (= x 1)) dict) => ((3 . 4) (5 . 6))
```

`(dict-search! `*dictionary key failure success*`)`

This procedure is a workhorse for dictionary lookup, insert, and delete.
The dictionary *dictionary* is searched
for an association whose key is the same as *key*
in the sense of *dictionary*'s comparator.
If one is not found, then the *failure* procedure is tail-called
with two continuation arguments, *insert* and *ignore*,
and is expected to tail-call one of them.

However, if such an association is found,
then the *success* procedure is tail-called
with the matching key of *dictionary*, the associated value,
and two continuation arguments, *update* and *remove*,
and is expected to tail-call one of them.

It is an error if the continuation arguments are invoked other than
in tail position in the *failure* and *success* procedures.
It is also an error if the *failure* and *success* procedures
return to their implicit continuation without invoking
one of their arguments.

The behaviors of the continuations are as follows
(where *obj* is any Scheme object):

 *  Invoking `(`*insert value obj*`)` returns a dictionary that
    contains all the associations of *dictionary*,
    and in addition a new association that maps *key* to *value*.

 *  Invoking `(`*ignore obj*`)` has no effects and returns *dictionary*
    unchanged.

 *  Invoking `(`*update new-key new-value obj*`)` returns a dictionary that
    contains all the associations of *dictionary*,
    except for the association whose key is the same as *key*,
    which is replaced or hidden by a new association
    that maps *new-key* to *new-value*.
    It is an error if *key* and *new-key* are not the same
    in the sense of the dictionary's equality predicate.

 *  Invoking `(`*remove obj*`)` returns a dictionary that
    contains all the associations of *dictionary*,
    except for the association with key key.

In all cases, *obj* is returned as a second value.

## The whole dictionary

`(dict-size `*dictionary*`)`

Returns an exact integer representing the number of associations in *dictionary*.

```
(dict-size dict) => 0
```

`(dict-for-each `*proc dictionary*`)`

Invokes *proc* on each key of *dictionary* and its corresponding value in that order.
This procedure is used for doing operations on the whole dictionary.
If the dictionary type is inherently ordered, associations are processed in the
inherent order; otherwise in an arbitrary order.
Returns an unspecified value.

```
(dict-for-each write dict) => unspecified
  ; writes "135" to current output
```

`(dict-count `*pred dictionary*`)`

Passes each association of dictionary as two arguments to *pred*
and returns the number of times
that *pred* returned true as an an exact integer.

```
(dict-count dict (lambda (k v) (even? k) => 0
```

`(dict-any `*pred dictionary*`)`

Passes each association of *dictionary* as two arguments to *pred*
and returns the value of the first call to *pred* that returns true,
after which no further calls are made.
If the dictionary type is inherently ordered, associations are processed in the
inherent order; otherwise in an arbitrary order.
If all calls return false, `dict-any` returns false.

```
(define (both-even? k v) (and (even? k) (even? v))
(dict-any both-even? '((2 . 4) (3 . 5))) => #t
(dict-any both-even? '((1 . 2) (3 . 4))) => #f
```

`(dict-every `*pred dictionary*`)`

Passes each association of *dictionary* as two arguments to *pred*
and returns `#f` after the first call to *pred* that returns false
after which no further calls are made.
If the dictionary type is inherently ordered, associations are processed in the
inherent order; otherwise in an arbitrary order.
If all calls return true, `dict-any` returns the value of the last call,
or `#t` if no calls are made.

```
(define (some-even? k v) (or (even? k) (even? v))
(dict-every some-even? '((2 . 3) (3 . 4))) => #t
(dict-every some-even? '((1 . 3) (3 . 4))) => #f
```

`(dict-keys `*dictionary*`)`

Returns a list of the keys of *dictionary*.
If the dictionary type is inherently ordered, associations are processed in the
inherent order; otherwise in an arbitrary order.
The order may change when new elements are added to *dictionary*.

```
(dict-keys dict) => (1 3 5)
```

`(dict-values `*dictionary*`)`

Returns a list of the values of *dictionary*.  The results returned
by `dict-keys` and `dict-values` are ordered consistently.

```
(dict-values dict) => (2 4 6)
```

`(dict-entries `*dictionary*`)`

Returns two values, the result of calling `dict-keys` and the
result of calling `dict-values` on *dictionary*.

```
(dict-entries dict) => ; 2 values
  (1 3 5)
  (2 4 6)
```

`(dict-fold `*proc knil dictionary*`)`

Invokes *proc* on each association of *dictionary* with three arguments:
the key of the association, the value of the association,
and an accumulated result of the previous invocation. 
For the first invocation, *knil* is used as the third argument.
Returns the result of the last invocation,
or *knil* if there was no invocation.

```
(dict-fold + 0 '((1 . 2) (3 . 4))) => 10
```

`(dict-map->list `*proc dictionary*`)`

Returns a list of values that result from invoking *proc*
on the keys and corresponding values of *dictionary*.

```
(dict-map->list - dict) => (-1 -1 -1)
```

`(dict->alist `*dictionary*`)`

Returns an alist whose keys and values are the keys and values of *dictionary*.

```
; plist to alist
(dict->alist '(1 2 3 4 5 6)) => ((1 . 2) (3 . 4) (5 . 6))
```

## Registering dictionary types

The following procedure registers new dictionary types.
It is an error to register a dictionary type whose
instances return `#t` to any predicate already registered.

`(register-dictionary! `*arg* ...`)`

Registers a new dictionary type, providing procedures that allow
manipulation of dictionaries of that type.
The *args* are alternately *procnames* and corresponding *procs*.

A *procname* argument is a symbol which is the same as one
of the procedures defined in this SRFI (other than
`register-dictionary!` itself), and a *proc* argument
is the specific procedure implementing it for this type.
These procedures only need to handle the full argument list
when defining `dict-ref` and `dict-update!`, as the
defaults have already been supplied by the framework.

Arguments for the six procedures `dictionary?`, `dict-size`,
`dict-search!`, `dict-map!`, `dict-filter!`, and `dict-for-each` are required.
The others are optional, but if provided can be more efficient
than the versions automatically provided by the implementation of this SRFI.

## Implementation

The sample implementation of this SRFI can be found in its repository.
The following list of dependencies is designed to ease registering
new dictionary types that may not have complete dictionary APIs:


 * `dict-empty?` depends on `dict-size`
 * `dict-contains?` depends on `dict-ref`
 * `dict-ref` depends on `dict-search!`
 * `dict-ref/default` depends on `dict-ref`
 * `dict-set!` depends on `dict-search!`
 * `dict-adjoin!` depends on `dict-search!`
 * `dict-delete!` depends on `dict-delete-all!`
 * `dict-delete-all!` depends on `dict-search!`
 * `dict-replace!` depends on `dict-search!`
 * `dict-intern!` depends on `dict-search!`
 * `dict-update!` depends on `dict-search!`
 * `dict-update/default!` depends on `dict-update!`
 * `dict-pop!` depends on `dict-for-each`, `dict-delete!` and `dict-empty?`
 * `dict-remove!` depends on `dict-filter!`
 * `dict-count` depends on `dict-fold`
 * `dict-any` depends on `dict-for-each`
 * `dict-every` depends on `dict-for-each`
 * `dict-keys` depends on `dict-fold`
 * `dict-values` depends on `dict-fold`
 * `dict-entries` depends on `dict-fold`
 * `dict-fold` depends on `dict-for-each`
 * `dict-map->list` depends on `dict-fold`
 * `dict->alist` depends on `dict-map->list`

For example, the first dependency means that if an implementation
being registered has something corresponding to `dict-ref` it need not
also supply `dict-contains?`, because its default implementation
uses `dict-ref`.  This might not be true in other implementations.