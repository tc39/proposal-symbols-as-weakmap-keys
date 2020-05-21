# ECMAScript Proposal for Boxing Objects into Primitives


**Champions**:

- Robin Ricard (@rricard)
- Rick Button (@rickbutton)

**Authors**:

- Robin Ricard (@rricard)
- Rick Button (@rickbutton)
- Daniel Ehrenberg (@littledan)

---

This proposal aims to solve a problem space introduced by the [Record & Tuple Proposal][rtp]; how can we reference and access non-primitive values in a primitive?

[Records & Tuples][rtp] can't contain objects, functions, or methods and will throw a `TypeError` when someone attempts to do it:

```js
const server = #{
    port: 8080,
    handler: function (req) { /* ... */ }, // TypeError!
};
```

This limitation exists because the one of the **key goals** of the [Record & Tuple Proposal][rtp]  is to have deep immutability guarantees _by default_.

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
const server = #{
    port: 8080,
    handler: refs.ref(function handler(req) { /* ... */ }),
};
refs.deref(server.handler)({ /* ... */ });
```

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

###### Membrane/Ocap implications

This isolation enables membrane-based security in the context of Boxes. Since Boxes are primitives, they cannot be membrane-wrapped. But, the `Box.prototype.deref()` method exists separately--this *method* can be membrane-wrapped, in order to mediate before giving out the information contained in the box.

Conceptually, in object-capability (ocap) security terms: Box primitives don't directly point to the object. Instead, they are treated as keys in a mapping, owned by the `Box` constructor, which `Box.prototype.deref` uses. Note that a Realm may deny access to this built-in mapping by monkey-patching `Box.prototype.deref`, which is the only point of access.

### `symbol` as `WeakMap` keys

Another solution is to make `WeakMap` accept `symbol` as keys. This could enable improvements to the userland solution seen before and let us do global automatic bookkeeping of references:

```js
class RefBookkeeper {
    #references = new WeakMap();
    ref(obj) { 
        const sym = Symbol();
        this.#references.set(sym, obj);
        return sym;
    }
    deref(sym) { return this.#references.get(sym); }
}
globalThis.refs = new RefBookkeeper();

const server = #{
    port: 8080,
    handler: refs.ref(function handler(req) { /* ... */ }),
};
refs.deref(server.handler)({ /* ... */ });
```

See [earlier discussion](https://github.com/tc39/ecma262/issues/1194) on Symbols as WeakMap keys--it remains controversial and not currently supported. 

### RefCollection

Our initial attempt to provide a solution for this problem space was [RefCollection](https://github.com/rricard/proposal-refcollection) which sits in between the two aforementioned solutions.
- A RefCollection could be built with WeakMaps with Symbol keys, but it's a somewhat higher-level mechanism.
- On the other hand, you could think of Box as a built-in per-Realm RefCollection, that just happens to use a new primitive type which is parallel to Symbol.

Since you could think of these each as implemented in terms of the other, more or less, they're equivalent at some level: they all attempt to leverage a primitive's identity in order to let an object be accessed, mediated through a mapping that only holds that object alive if the unforgeable primitive exists.

## Summing up

We think that `Box` has the most ergonomic surface area of these three options, which is why this document puts that solution front and center, ahead of Symbols in WeakMaps and RefCollection. At the same time, the userspace solutions seem sufficient for many/most use cases; we believe that Records and Tuples are very useful without Box or any other built-in mechanism for referencing objects from primtiives, and therefore makes sense to proceed with Records and Tuples independently.

[rtp]: https://github.com/tc39/proposal-record-tuple
[rcp]: https://github.com/rricard/proposal-refcollection
