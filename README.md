# Bind-`this` operator for JavaScript
ECMAScript Stage-0 Proposal. J. S. Choi, 2021.

* **[Formal specification][]**
* Babel plugin: Not yet

[formal specification]: http://jschoi.org/21/es-bind-operator/

This proposal is a **resurrection**
of the old [Stage-0 Bind Operator proposal][old bind].
It is also an alternative, **competing** proposal
to the [Stage-1 Extensions proposal][extensions].
For more information, see [§ Related proposals](#related-proposals).

[old bind]: https://github.com/tc39/proposal-bind-operator
[extensions]: https://github.com/tc39/proposal-extensions

## Description
(A [formal specification][] is available.)

Method binding `->` is a **left-associative** binary operator.
Its right-hand side is an **identifier** (like `f`)
or a parenthesized **expression** (like `(hof())`),
either of which must evaluate to a **function**.
Its left-hand side is some expression that evaluates to an **object**.
The `->` operator binds its left-hand side
to its right-hand side’s `this` value,
creating a **bound function** in the same manner
as [`Function.prototype.bind`][bind].

For example, `arr->fn` would equivalent to `fn.bind(arr)`
(except that its behavior does not change
if code elsewhere reassigns the global method `Function.prototype.bind`).

Likewise, `obj->(createMethod())` would be roughly
equivalent to `createMethod().bind(obj)`.

If the operator’s right-hand side does not evaluate to a function during runtime,
then the program throws a `TypeError`.

The bound functions created by the bind-`this` operator
are **indistinguishable** from the bound functions
that are already created by [`Function.prototype.bind`][bind].
Both are **exotic objects** that do not have a `prototype` property,
and which may be called like any typical function.

From this definition, `o->f(...args)`
is also **indistinguishable** from `f.call(o, ...args)`
(except that its behavior does not change
if code elsewhere reassigns the global method `Function.prototype.call`).

The `this`-bind operator has equal **[precedence][]** with
**member expressions**, call expressions, `new` expressions with arguments,
and optional chains.
Like those operators, the `this`-bind operator also may be short-circuited
by optional chains in its left-hand side.

[precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence

| Left-hand side                     | Example      | Grouping
| ---------------------------------- | ------------ | --------------
| Member expressions                 |`a.b->fn.c`   |`((a.b)->fn).c`
| Call expressions                   |`a()->fn()`   |`((a())->fn)()`
| Optional chains                    |`a?.b->fn`    |`(a?.b)->fn`
|`new` expressions with arguments    |`new C(a)->fn`|`(new C(a))->fn`
|`new` expressions without arguments*|`new a->fn`   |`new (a->fn)`

\* Like `.` and `?.`, the `this`-bind operator also have tighter precedence
than `new` expressions without arguments.
Of course, `new a->fn` is not a very useful expression,
just like how `new (fn.bind(a))` is not a very useful expression.

Similarly to the `.` and `?.` operators,
the `->` operator may be **padded by whitespace**.\
For example, `a -> m`\
is equivalent to `a->fn`,\
and `a -> (createFn())`\
is equivalent to `a->(createFn())`.

There are **no other** special rules.

## Why a bind-`this` operator
In short:

1. [`.bind`][bind], [`.call`][call], and [`.apply`][apply]
   are very useful and very common in JavaScript codebases.
2. But `.bind`, `.call`, and `.apply` are clunky and unergonomic.

[bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
[call]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call
[apply]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply

### `.bind`, `.call`, and `.apply` are very common
The dynamic `this` binding is a fundamental part of JavaScript design and practice today.
Because of this, developers frequently need to change the `this` binding.
`.bind`, `.call`, and `.apply` are arguably three of the most commonly used functions
in all JavaScript.

We can estimate `.bind`, `.call`, and `.apply`’s prevalences using [Node Gzemnid][].
Although [Gzemnid can be deceptive][], we are only seeking rough estimations.

[Node Gzemnid]: https://github.com/nodejs/Gzemnid
[Gzemnid can be deceptive]: https://github.com/nodejs/Gzemnid/blob/main/README.md#deception

First, we download the 2019-06-04 [pre-built Gzemnid dataset][]
for the top-1000 downloaded NPM packages.
We also need Gzemnid’s `search.topcode.sh` script in the same active directory,
which in turn requires the lz4 command suite.
`search.topcode.sh` will output lines of code from the top-1000 packages
that match the given regular expression.

[pre-built Gzemnid dataset]: https://gzemnid.nodejs.org/datasets/

```bash
./search.topcode.sh '\.call\b' | head
grep -aE  "\.call\b"
177726827	debug@4.1.1/src/common.js:101:					match = formatter.call(self, val);
177726827	debug@4.1.1/src/common.js:111:			createDebug.formatArgs.call(self, args);
154772106	kind-of@6.0.2/index.js:54:  type = toString.call(val);
139612972	readable-stream@3.4.0/errors-browser.js:26:      return _Base.call(this, getMessage(arg1, arg2, arg3)) || this;
139612972	readable-stream@3.4.0/lib/_stream_duplex.js:60:  Readable.call(this, options);
139612972	readable-stream@3.4.0/lib/_stream_duplex.js:61:  Writable.call(this, options);
139612972	readable-stream@3.4.0/lib/_stream_passthrough.js:34:  Transform.call(this, options);
139612972	readable-stream@3.4.0/lib/_stream_readable.js:183:  Stream.call(this);
139612972	readable-stream@3.4.0/lib/_stream_readable.js:786:  var res = Stream.prototype.on.call(this, ev, fn);
```

We use `awk` to count those matching lines of code
and compare their numbers for `bind`, `call`, `apply`,
and several other frequently used functions.

```bash
> ls
search.topcode.sh
slim.topcode.1000.txt.lz4
> ./search.topcode.sh '\.call\b' | awk 'END { print NR }'
500084
> ./search.topcode.sh '\.apply\b' | awk 'END { print NR }'
225315
> ./search.topcode.sh '\.bind\b' | awk 'END { print NR }'
170248
> ./search.topcode.sh '\b.map\b' | awk 'END { print NR }'
1016503
> ./search.topcode.sh '\bconsole.log\b' | awk 'END { print NR }'
271915
> ./search.topcode.sh '\.set\b' | awk 'END { print NR }'
168872
> ./search.topcode.sh '\.push\b' | awk 'END { print NR }'
70116
```

These results suggest that usage of `.call`, `.bind`, and `.apply`
are comparable to usage of other frequently used standard functions.
In this dataset, their combined usage even exceeds that of `console.log`.

Obviously, [this methodology has many pitfalls][Gzemnid can be deceptive],
but we are only looking for roughly estimated orders of magnitude
relative to other baseline functions.
Gzemnid counts each library’s codebase only once; it does not double-count dependencies.

In fact, this method definitely underestimates the prevalences
of `.bind`, `.call`, and `.apply`
by excluding the large JavaScript codebases of Node and Deno.
Node and Deno [copiously use bound functions for security][security-use-case]
hundreds or thousands of times.

[security-use-case]: https://github.com/js-choi/proposal-bind-this/blob/main/security-use-case.md

### `.bind`, `.call`, and `.apply` are clunky
JavaScript developers are used to using methods in a [noun–verb–noun word order][]
that resembles English and other [SVO human languages][]: `obj.fn(arg)`.

[SVO human languages]: https://en.wikipedia.org/wiki/Category:Subject–verb–object_languages

However, `.bind`, `.call`, and `.apply` flip this “natural” word order,
They flip the first noun and the verb,
and they interpose the verb’s `Function.prototype` method between them:
`obj.call(arg)`.

Consider the following real-life code using `.bind` or `.call`,
and compare them to versions that use the `->` operator.
The difference is especially evident when you read them aloud.

```js
// kind-of@6.0.2/index.js
type = toString.call(val);
type = val->toString();

// debug@4.1.1/src/common.js
match = formatter.call(self, val);
match = self->formatter(val);

createDebug.formatArgs.call(self, args);
self->(createDebug.formatArgs)(args);

// readable-stream@3.4.0/errors-browser.js
return _Base.call(this, getMessage(arg1, arg2, arg3)) || this;
return this->_Base(getMessage(arg1, arg2, arg3)) || this;

// readable-stream@3.4.0/lib/_stream_readable.js
var res = Stream.prototype.on.call(this, ev, fn);
var res = this->(Stream.prototype.on)(ev, fn);

var res = Stream.prototype.removeAllListeners.apply(this, arguments);
var res = this->(Stream.prototype.removeAllListeners)(...arguments);

// yargs@13.2.4/lib/middleware.js
Array.prototype.push.apply(globalMiddleware, callback)
globalMiddleware->(Array.prototype.push)(...callback)

// yargs@13.2.4/lib/command.js
[].push.apply(positionalKeys, parsed.aliases[key])

// pretty-format@24.8.0/build-es5/index.js
var code = fn.apply(colorConvert, arguments);
var code = colorConvert->fn(...arguments);

// q@1.5.1/q.js
return value.apply(thisp, args);
return thisp->value(...args);

// rxjs@6.5.2/src/internal/operators/every.ts
result = this.predicate.call(this.thisArg, value, this.index++, this.source);
result = this.thisArg->(this.predicate)(value, this.index++, this.source);

// bluebird@3.5.5/js/release/synchronous_inspection.js
return isPending.call(this._target());
return this._target()->isPending();

var matchesPredicate = tryCatch(item).call(boundTo, e);
var matchesPredicate = boundTo->(tryCatch(item))(e);

// graceful-fs@4.1.15/polyfills.js
return fs$read.call(fs, fd, buffer, offset, length, position, callback)
return fs->fs$read(fd, buffer, offset, length, position, callback)
```

[noun–verb–noun word order]: https://en.wikipedia.org/wiki/Subject–verb–object

## Non-goals
A goal of this proposal is **simplicity**.
Therefore, this proposal purposefully
does *not* address the following use cases.

**Tacit method extraction** with another operator
(like `arr&.slice` for `arr.slice.bind(arr.slice)` hypothetically)
would be nice to have,
but method extraction is already possible with this proposal.\
`const slice = arr->(arr.slice); slice(1, 3);`\
is not much wordier than\
`const slice = arr&.slice; slice(1, 3);`

**Extracting property accessors** (i.e., getters and setters)
is not a goal of this proposal.
Get/set accessors are **not like** methods.
Methods are **properties** (which happen to be functions).
Accessors themselves are **not** properties;
they are functions that activate when getting or setting properties.
Getters/setters have to be extracted using `Object.getOwnPropertyDescriptor`;
they are not handled in a special way.
This verbosity may be considered to be desirable [syntactic salt][]:
it makes the developer’s intention (to extract getters/setters – and not methods)
more explicit.

```js
const { get: $getSize } =
  Object.getOwnPropertyDescriptor(
    Set.prototype, 'size');

// The adversary’s code.
delete Set; delete Function;

// Our own trusted code, running later.
new Set([0, 1, 2])->$getSize();
```

**Function/expression application**,
in which deeply nested function calls and other expressions
are untangled into linear pipelines,
is important but not addressed by this proposal.
Instead, it is addressed by the **pipe operator**,
with which this proposal’s syntax works well.\
For example, we could untangle `h(await g(o->f(0, v)), 1)`\
into `v |> o->f(0, %) |> await g(%) |> h(%, 1)`.

[syntactic salt]: https://en.wikipedia.org/wiki/Syntactic_sugar#Syntactic_salt
[primordials.js]: https://github.com/nodejs/node/blob/master/lib/internal/per_context/primordials.js

## Related proposals

### Old bind operator
This proposal is a **resurrection**
of the old [Stage-0 Bind Operator proposal][old bind].
[TODO]

### Extensions
This proposal is an alternative, **competing** proposal
to the [Stage-1 Extensions proposal][extensions].
[TODO]

### Pipe operator
[TODO]
