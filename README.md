# Symbols as WeakMap keys

Stage 1

**Coauthors/champions**:

- Robin Ricard (@rricard)
- Rick Button (@rickbutton)
- Daniel Ehrenberg (@littledan)

[Spec text](https://arai-a.github.io/ecma262-compare/?pr=2038)

---

This proposal aims to solve a problem space introduced by the [Record & Tuple Proposal][rtp]; how can we reference and access non-primitive values in a primitive?

tl;dr We see Symbols, dereferenced through WeakMaps, as the most reasonable way forward to reference Objects from Records and Tuples, given all the constraints raised in the discussion so far.

There are some open questions as to how this should they work exactly, and also valid ergonomics/ecosystem coordination issues, which we hope to resolve/validate in the course of the TC39 stage process. We'll start with an understanding of the problem space, including why Records and Tuples are a good first step without this feature. Then, we'll examine various possible solutions, with their pros and cons, 

----

[Records & Tuples][rtp] can't contain objects, functions, or methods and will throw a `TypeError` when someone attempts to do it:

```js
const server = #{
    port: 8080,
    handler: function (req) { /* ... */ }, // TypeError!
};
```

This limitation exists because the one of the **key goals** of the [Record & Tuple Proposal][rtp]  is to have deep immutability guarantees and structural equality _by default_.

The userland solutions mentioned below provide multiple methods of side-stepping this limitation, and `Record and Tuple` is viable and useful without additional language support for boxing objects. This proposal attempts to describe solutions that complement the usage of these userland solutions with `Record and Tuple`, but is not a prerequisite to landing `Record and Tuple` in the language.

## Userland solutions

You can already escape the aforementioned constraints without additional language features.

Instead of directly storing objects in a record or tuple, you can instead store some application domain specific information that can be used somewhere else to perform the desired action. For example, instead of embedding an `execute` function in an object like this example:

```js
const object = {
    execute() {
        console.log("foo");
    },
};
object.execute();
```

You can instead store an `action` that gets consumed by another function when needed:
```js
const record = #{
    action: "foo",
};

function execute(record) {
    if (record.action === "foo") {
        console.log("foo");
    }
}

execute(record);
```

As another example, if you want to store an object containing primitives in a record, you can store the individual properties themselves instead with the spread operator:

Instead of:
```js
const object = { foo: "bar" };
const record = #{ object: object }; // TypeError
```

Try this:
```js
const object = { foo: "bar" };
const record = #{ object: #{ ...object } };
```

#### Centralizing references to objects

If you can't convert everything to primitives, Records and Tuples directly, then a separate mapping, implemented in JavaScript, could be used to explain how some primitives should be interpreted in terms of objects. You can think of this as generalizing the interpretation of the action strings above.

For example, a simple Array of references could be passed around, in parallel to a Record, and certain numbers in the Record could be treated as indices into that Array when needed.

```js
const server = (() => {
    const handlerRef = 0;
    function handler(req) { /* ... */ }
    const structure = #{
        port: 8080,
        handler: handlerRef,
    };
    const references = [];
    references[handlerRef] = handler;
    return {
        structure,
        references,
    };
})();
server.references[server.structure.handler]({ /* ... */ });
```

To access, explicit reference to the Array needs to be made. These references could be encapsulated in a bookkeeper class, to assist with dereferencing:

```js
class RefBookkeeper {
    #references = [];
    ref(obj) { 
        const idx = this.#references.length;
        this.#references[idx] = obj;
        return idx;
    }
    deref(sym) { return this.#references[sym]; }
}

// Usage
const server = (() => {
    const references = new RefBookkeeper();
    const structure = #{
        port: 8080,
        handler: references.ref(function handler(req) { /* ... */ }),
    };
    return {
        structure,
        references,
    };
})();
server.references.deref(server.structure.handler)({ /* ... */ });
```

You might want to reuse the same `RefBookkeeper` object across multiple server instances, so you don't have as much to keep track of for each instance:

```js
globalThis.refs = new RefBookkeeper();

// Usage
const server = #{
    port: 8080,
    handler: refs.ref(function handler(req) { /* ... */ }),
};
refs.deref(server.handler)({ /* ... */ });
```

See also [a larger worked example](https://github.com/rricard/proposal-refcollection/issues/5#issuecomment-619104173) demonstrating the kinds of code patterns where you'd want to be able to use a RefBookkeeper across several functions.

However, this pattern would lead to some memory management issues: As the program runs, more and more things might get added to `refs`, the Array just gets longer and longer, and all of the entries in the Array have to be held in memory just in case someone calls `deref` to get them, even if no one is actually going to do that. But really, we should avoid referencing things that are no longer needed anymore, so the garbage collector can reclaim memory. There are two ways to do this:
- If we reuse one global `RefBookkeeper` everywhere, then we need to manually delete unused entries when we're done with them, with an extra `delete` method added to the class, allowing the index to be reused. (We can't use FinalizationRegistry for this, since Numbers are never really dead.)
- Otherwise, we could avoid using a global RefBookkeeper, and instead use smaller local ones, which we pass around in parallel to the Record/Tuple, as in the previous examples.

Both of these are a bit unfortunate; it would be nice if you didn't have to worry about deleting entries from `RefBookkeeper`, and if you didn't have to pass it around in parallel with Records and Tuples. Further, it's not really optimal ergonomics to have to call `refs.deref(idx)` all the time, and to have to remember which numbers serve as an index into the RefBookkeeper and which are just meant to be numbers.

# Boxing objects to place in Records and Tuples while avoiding bookeeping

Overall, with the patterns described above, Records and Tuples can work with JavaScript objects in a variety of ways, such that many different problems can be solved with them directly. We don't believe that any additions are *needed* to make Records and Tuples significantly useful. However, to improve ergonomics further, this proposal exists to discuss various mechanisms to make simple, flexible references to JavaScript objects.

### The `box` primitive type

This potential solution introduces a new primitive type called `box`. These primitives can be constructed with `Box(obj)`, and the object can be retrieved with `box.deref()`, which would return `obj`.

#### Examples

```js
const obj = { hello: "world" };
const box = Box(obj);
assert(typeof box === "box", "boxes are a new primitive type");
assert(obj !== box, "boxes are not their boxed object");
assert(obj.hello === box.deref().hello, "boxes can deref props");
assert(obj === box.deref(), "boxes can deref the full object");
```

```js
const server = #{
    port: 8080,
    handler: Box(function handler(req) { /* ... */ }),
};
server.handler.deref()({ /* ... */ });
```

#### Semantics

##### Boxing the same object gives the same box

```js
const obj = {};
const box1 = Box(obj);
const box2 = Box(obj);
assert(box1 === box2);
```

##### Boxes remove the need for bookkeeping

There's no need to pass around any kind of `RefBookkeeper` object; just call `.deref()` directly on the box that's in the Record or Tuple.

JS's built-in garbage collector understands boxes, so you don't have to worry about keeping track of a separate reference to the object, or nulling that reference out when it's no longer needed. As long as you can access a box, calling `.deref()` on the box will give you access to the object it was created with.

##### Edge case: Calling `.deref()` on a Box when passed to a separate global object (Realm)

When working in the context of multiple JS global objects (e.g., with a same-origin iframe, or the Node.js vm module), if a box is passed from one global object to another, then calling the `.deref()` method in the context of that other global object would lead to behavior as if the box is unrecognized (either returning `undefined` or throwing; bikeshedding welcome). 

It's possible to get around this by passing `Box.prototype.deref()` to that other global object. This method "grants the right" to dereference boxes created in the context of that global object. In this way, different global objects are a little bit isolated from each other, when it comes to `Box`.

##### Formalization of Box semantics

Data model
- Each Box has an associated [[Id]]
- Each Realm has an associated weak mapping [[Id]] -> Boxed object

API
- Box(obj)
    - Create a new [[Id]], and write [[Id]] -> obj into the Realm's mapping
    - Return a Box value with an associated [[Id]]
- Box.prototype.deref
    - If this Realm's mapping contains an entry for this.[[Id]]
        - Return the associated object
    - Else, return undefined

###### Sharing and isolating with Boxes

`Box.prototype.deref` is the only way to get at that Realm's mapping of [[Id]] to objects. Two different Realms cannot see into each other's Boxes, unless one Realm gives the other Realm a reference to its `Box.prototype.deref` method. A Realm can pull off its `deref`, delete it, etc, to control access to the mapping.

If using membrane-based isolation within a Realm, `Box()` + `Box.prototype.deref` bypasses the membrane. To preserve the membrane, the feature must be disabled early, by
```js
delete Box.prototype.deref
```
Some committee members want all TC39 features to be available within a single (potentially frozen) Realm, membrane-isolated world. This goal makes the Box proposal inviable.

<details>
Our hope was that, the fact that object operations are used to access and call `Box.prototype.deref`, and this can be membrane-wrapped, would imply that the Box system would work great for membrane-based security--just run each membrane-enclosed piece of code in a separate Realm, and you can safely share Boxes, granting or denying access based on the access to the `Box` constructor or `deref` method.

However, in systems which want to use membrane-based isolation *within* the same Realm, the whole feature would have to be turned off. This is to prevent boxes from "piercing" through membranes: a box primitive could be passed from one side of the membrane to the other (the membrane has no chance to intervene since boxes are not objects), and then it can simply be dereferenced. This would constitute an unmediated cross-membrane communication channel, going against the goals of the membrane system.

All access to the contents of boxes goes through `Box.prototype.deref`. A Realm's `Box.prototype` is ambiently available within that Realm, since it's referenced from ToObject, as long as you have a box. Although it is possible to turn off this feature from JavaScript by removing that function, this seems unfortunate for two reasons:
- If the Realm is already deeply frozen, it's no longer possible to mutate `Box.prototype`
- It'd be unfortunate to lose the functionality!
</details>

### BoxMakers to avoid Realm-wide communication channels

> Boxmaker, boxmaker, plan me no plans.
>
> I’m in no rush. Maybe I’ve learned
>
> Playing with boxes a girl can get burned.
>
> -- [A musical](http://www.exelana.com/lyrics/MatchmakerMatchmaker.html), probably

Ultimately, what we want is, a way to construct and dereference boxes, without being attached to the Realm. It could be used something like this:

```js
const Box = new BoxMaker();

const obj = { hello: "world" };
const box = Box(obj);
assert(typeof box === "box", "boxes are a new primitive type");
assert(obj !== box, "boxes are not their boxed object");
assert(obj.hello === Box.deref(box).hello, "boxes can deref props");
assert(obj === Box.deref(box), "boxes can deref the full object");
```

```js
const server = #{
    port: 8080,
    handler: Box(function handler(req) { /* ... */ }),
};
Box.deref(server.handler)({ /* ... */ });
```

This is definitely a bit uglier--it's annoying to have to instantiate your own `Box` constructor, call out to it explicitly to call `deref` as opposed to using method chaining, and to coordinate all your code to use the same `Box`. Still, if you  reuse the same `Box` constructor across all your functions, they interact "for free" without any manual bookkeeping or passing around of auxiliary structures, as described with `RefBookkeeper` above.

The need for a `Box`/`BoxMaker` separation is a direct consequence of the goal to avoid membranes within the same Realm that can use this feature.

#### Another way to look at it: RefCollection

Our initial attempt to provide a solution for this problem space was [RefCollection](https://github.com/rricard/proposal-refcollection), which is basically the same as `BoxMaker`, except that it uses Symbols instead of Box primitives.

When you call the Symbol constructor, you get a new Symbol; ultimately it can serve just as well to reference the Object as a Box primitive does. The RefCollection API allocates the Symbol for you, just like `Box.ref` creates a box. They're quite similar, when it comes down to it!

RefCollection is a bit simpler since it doesn't create a new primitive type, just reusing what we have. However, it's not possible to polyfill RefCollection without something new in the language: we run into the exact problem described with RefBookkeeper above, where it can fill up with junk that's not relevant anymore, if the same RefCollection keeps getting reused. (Refer to [this example](https://github.com/rricard/proposal-refcollection/issues/5#issuecomment-619104173) to see how you might want to continue using a RefCollection over time.)

There was a bit of debate about the details of the API surface of RefCollection, e.g., as it would relate to reusable templates. This seems to point towards exposing a low-level primitive that could address these memory management issues, leaving JavaScript libraries and frameworks to develop the best API surface.

### `symbol` as `WeakMap` keys

We propose to make `WeakMap` accept `symbol` as keys. This would allow JavaScript libraries to implement their own RefCollection-like things which could be reusable (avoiding the need to pass around the mapping all over the place, using a single global one, and just passing around Records and Tuples) while not leaking memory over time.

```js
class RefBookkeeper {
    #references = new WeakMap();
    ref(obj) {
        // (Simplified; we may want to return an existing symbol if it's already there)
        const sym = Symbol();
        this.#references.set(sym, obj);
        return sym;
    }
    deref(sym) { return this.#references.get(sym); }
}
globalThis.refs = new RefBookkeeper();

// Usage
const server = #{
    port: 8080,
    handler: refs.ref(function handler(req) { /* ... */ }),
};
refs.deref(server.handler)({ /* ... */ });
```

See [earlier discussion](https://github.com/tc39/ecma262/issues/1194) on Symbols as WeakMap keys--it remains controversial and not currently supported.

Some questions to discuss about Symbols as WeakMap keys:
- Can registered symbols be used as WeakMap keys? Some TC39 delegates have argued strongly in either direction. We see both "allowing" and "disallowing" as acceptable options.
    - Allowing registered symbols doesn't seem so bad, since registered Symbols are analogous to Objects that are held alive for the lifetime of the Realm. In the context of a Realm that stays alive as long as there's JS running (e.g., on the Web, the Realm of a Worker), things like `Symbol.iterator` are analogous to primordials like `Object.prototype` and `Array.prototype`, and registered `Symbol.for()` symbols are analogous to properties of the global object, in terms of lifetime. Just because these will stay alive doesn't mean we disallow them as WeakMap keys.
    - Prohibiting registered symbols doesn't seem so bad, since it's already readily observable whether a Symbol is registered, and it's not very useful to include these as WeakMap keys. Therefore, it's hard to see what practical or consistency problems the prohibition would create, or why it would be surprising (if there's a meaningful error message).
- We could support Symbols in WeakRefs and FinalizationRegistry, or not. It's not clear what the use cases are, but it would seem consistent with adding them as WeakMap keys.

As starting points, we propose that all Symbols be allowed as WeakMap keys, WeakSet entries, and in WeakRefs and FinalizationRegistry.

## Summing up

We think that adding Symbols as WeakMap keys is a useful, minimal primitive enabling Records and Tuples to reference Objects while respecting the constraints imposed by the goal to support membrane-based isolation within a single Realm. At the same time, the userspace solutions seem sufficient for many/most use cases; we believe that Records and Tuples are very useful without any additional mechanism for referencing objects from primitives, and therefore makes sense to proceed with Records and Tuples independently of this proposal.

[rtp]: https://github.com/tc39/proposal-record-tuple
[rcp]: https://github.com/rricard/proposal-refcollection
