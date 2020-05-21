# ECMAScript Proposal for Boxing Objects into Primitives

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
const record = #{ ...object };
```

#### If these solutions are not sufficient, there are several userland alternatives that allow you to indirectly reference objects from records and tuples:

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

However, we can always wrap things up in a class so that the bookkeeper can generate and associate the symbol and help you "dereference" it:

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

At this point one flaw of the userland approach starts to appear: all referenced objects will stay in memory as long as the bookkeeper exists. If the bookkeeper is global, the referenced objects will stay in memory as long as the realm exists...

While these userland solutions make referencing objects from record and tuple possible, they have two flaws:

- Manual bookkeeping of references is necessary to avoid memory leaks
- Ergonomics: explicit referencing and dereferencing is something that is necessary because we still want to be deeply immutable _by default_  but in userland solutions they can be cumbersome to use, especially dereferencing.

# Boxing objects and automatic bookkeeping

This proposal intends to find a solution to the aforementioned issues. In this section we explore possible solutions that we found that could help us solve those issues.

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

Our initial attempt to provide a solution for this problem space was [RefCollection](https://github.com/rricard/proposal-refcollection) which sits in between the two aforementioned solutions. There is not a fundamental difference between these solutions, as they all attempt to leverage a primitive's identity in order to point to an object, just via different mechanisms.

[rtp]: https://github.com/tc39/proposal-record-tuple
[rcp]: https://github.com/rricard/proposal-refcollection
