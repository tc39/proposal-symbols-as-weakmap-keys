# Boxing objects into primitive types proposal

This proposal aims to solve a problem space introduced by the [Record & Tuple Proposal][rtp]: how can we reference and access non-primitive types in a primitive type.

Right now, [Records & Tuples][rtp] can't refer to objects, functions or methods and will throw a `TypeError` when someone attempts to do it:

```js
const server = #{
    port: 8080,
    handler: function (req) { /* ... */ }, // TypeError!
};
```

This limitation exists because the [Record & Tuple Proposal][rtp] attempts to keep 2 invariants:

- They behave like primitive types
- They have deep immutability guarantees _by default_

Being immutable by default is important but could be overridable and that is what this proposal is about.

There are two main use cases for relaxing this default (as far as we know):

- Being able to store an object or anything that can't be represented as a primitive type alongside some deeply immutable data
- Being able to reference mutable state from an immutable representation of that state (like a virtual dom)

Being able to perform those actions opens up more use cases and possibilities for primitive types. The goal of this proposal is to introduce ways to escape that default behavior!

## Userland solutions

You can already escape those constraints as of today!

For instance you can create a bookkeeper object that stores references (symbols, but could be strings or numbers) in a separate object that you can pass around alongside your records and tuples:

```js
const server = (() => {
    const handlerRef = Symbol();
    function handler(req) { /* ... */ }
    const structure = #{
        port: 8080,
        handler: handlerRef,
    };
    const references = {
        [handlerRef]: handler,
    };
    return {
        structure,
        references,
    };
})();
server.references[server.structure.handler]({ /* ... */ });
```

This approach requires us to make sure the bookkeeper is moved around alongside the structure. Additionally this system is not very ergonomic.

However, we can always wrap things up in a class so the bookkeeper could generate and associate the symbol and help you "dereference" it:

```js
class RefBookkeeper {
    #references = {};
    ref(obj) { 
        const sym = Symbol();
        this.#references[sym] = obj;
        return sym;
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

Finally, we can always make that bookkeeping global and stop moving the refs with the structure:

```js
globalThis.refs = new RefBookkeeper();
const server = #{
    port: 8080,
    handler: refs.ref(function handler(req) { /* ... */ }),
};
refs.deref(server.handler)({ /* ... */ });
```

At this point one major flaw of the userland approach starts to appear: all referenced objects will stay in memory as long as the bookkeeper exists. If the bookkeeper is global, the referenced objects will stay in memory as long as the realm exists...

While those userland solutions make object referencing from record and tuple possible, they have two major flaws:

- Manual bookkeeping of references is necessary to avoid memory leaks
- Ergonomics: explicit referencing and dereferencing is something that is necessary because we still want to be deeply immutable _by default_  but in userland solutions they can be cumbersome to use, especially the dereferencing

## Boxing objects and automatic bookkeeping

This proposal intends to find a solution to the before-mentioned issues. In this section we explore possible solutions that we found that could help us solve those issues.

### The `box` primitive type

This potential solution introduces a new primitive type called `box`. It has a itself a boxing object `Box` that carries some very useful methods.

#### Examples

```js
const obj = { hello: "world" };
const box = Box(obj);
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

##### Boxes use a global automatic bookkeeper

Boxes will not keep objects indefinitely in memory. When a box is not needed anymore, the attached object can be let go.

##### Boxes are attached to their realm

The bookkeeper that the boxes use is attached to their realm. Dereferencing a box from another realm in a given realm will not return the object. (will either throw or return `undefined`)

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

### RefCollection

Our first attempt to solve that issue was [RefCollection](https://github.com/rricard/proposal-refcollection) which is in between the two befroe-mentioned solutions. It unfortunately does not solve the issues this proposal is putting forward well enough.

[rtp]: https://github.com/tc39/proposal-record-tuple
[rcp]: https://github.com/rricard/proposal-refcollection
