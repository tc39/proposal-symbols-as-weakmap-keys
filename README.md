# Symbols as WeakMap keys

Stage 3

**Coauthors/champions**:

- Robin Ricard (@rricard)
- Rick Button (@rickbutton)
- Daniel Ehrenberg (@littledan)
- Leo Balter (@leobalter)
- Caridy Patiño (@caridy)
- Rick Waldron (@rwaldron)
- Ashley Claymore (@acutmore)

[Spec text](https://tc39.es/proposal-symbols-as-weakmap-keys)

---

## Introduction

This proposal extends the WeakMap API to allow usage of unique Symbols as keys.

Currently, WeakMaps are limited to only allow objects as keys, and this is a limitation for WeakMaps as the goal is to have unique values that can be eventually GC'ed.

Symbol is the only primitive type in ECMAScript that allows unique values. A symbol value - like the one produced by calling the `Symbol( [ description] )` expression - can only be identified with access to its original production. Any reproduction of the same expression - using the same value for description - will not restore the original value of any previous production. This is why we call the symbol values distinct.

Objects are used as keys for WeakMaps because they share the same identity aspect. The identity of an object can only be verified with access to the original production, no new object will match a pre-existing one in - e.g. - a strict comparison.

### Earlier discussions

See [earlier discussion](https://github.com/tc39/ecma262/issues/1194) on Symbols as WeakMap keys.

### Draft PR

See the current [draft PR to ECMA-262](https://github.com/tc39/ecma262/pull/2777) with the proposed spec.

## Use Cases

### Easy to create and share keys

Instead of requiring creating a new object to be only used as a key, a symbol would provide more clarity for the ergonomics of a WeakMap and the proper roles of its keys and mapped items.

```javascript
const weak = new WeakMap();

// Pun not intended: being a symbol makes it become a more symbolic key
const key = Symbol('my ref');
const someObject = { /* data data data */ };

weak.set(key, someObject);
```

### ShadowRealms, Membranes, and Virtualization

The [ShadowRealms proposal](https://github.com/tc39/proposal-shadowrealm) disallows access to object values. For most virtualization cases, a membrane system is built on top of Realms-related API to connect references using WeakMaps. A Symbol value, being a primitive value, is still accessible, allowing membranes being structured with proper weakmaps using connected identities.

```javascript
const objectLookup = new WeakMap();
const otherRealm = new ShadowRealm();
const coinFlip = otherRealm.evaluate(`(a, b) => Math.random() > 0.5 ? a : b;`);

// later...
let a = { name: 'alice' };
let b = { name: 'bob' };
let symbolA = Symbol();
let symbolB = Symbol();
objectLookup.set(symbolA, a);
objectLookup.set(symbolB, b);
a = b = null; // ok to drop direct object references

// connected identities preserved as the symbols round-tripped through the other realm
let chosen = objectLookup.get(coinFlip(symbolA, symbolB));
assert(['alice', 'bob'].includes(chosen.name));
```

### Record and Tuples

This proposal aims to solve a problem space introduced by the [Record & Tuple Proposal][rtp]; how can we reference and access non-primitive values in a primitive?

tl;dr We see Symbols, dereferenced through WeakMaps, as the most reasonable way forward to reference Objects from Records and Tuples, given all the constraints raised in the discussion so far.

There are some open questions as to how this should they work exactly, and also valid ergonomics/ecosystem coordination issues, which we hope to resolve/validate in the course of the TC39 stage process. We'll start with an understanding of the problem space, including why Records and Tuples are a good first step without this feature. Then, we'll examine various possible solutions, with their pros and cons,

[Records & Tuples][rtp] can't contain objects, functions, or methods and will throw a `TypeError` when someone attempts to do it:

```js
const server = #{
    port: 8080,
    handler: function (req) { /* ... */ }, // TypeError!
};
```

This limitation exists because the one of the **key goals** of the [Record & Tuple Proposal][rtp]  is to have deep immutability guarantees and structural equality _by default_.

The userland solutions mentioned below provide multiple methods of side-stepping this limitation, and `Record and Tuple` is viable and useful without additional language support for boxing objects. This proposal attempts to describe solutions that complement the usage of these userland solutions with `Record and Tuple`, but is not a prerequisite to landing `Record and Tuple` in the language.

Accepting Symbol values as WeakMap keys would allow JavaScript libraries to implement their own RefCollection-like things which could be reusable (avoiding the need to pass around the mapping all over the place, using a single global one, and just passing around [Records and Tuples](https://github.com/tc39/proposal-record-tuple)) while not leaking memory over time.

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

## Well-known and Registered symbols as WeakMap keys

Some TC39 delegates have argued strongly in either direction. We see both "allowing" and "disallowing" as acceptable options.

### [Registered](https://tc39.es/ecma262/multipage#sec-symbol.for) symbols

Disallowing registered symbols is discussed in [issue 21](https://github.com/tc39/proposal-symbols-as-weakmap-keys/issues/21).

### [Well-Known](https://tc39.es/ecma262/multipage#sec-well-known-symbols) symbols

Allowing well-known symbols doesn't seem so bad, since they are analogous to Objects that are held alive for the lifetime of the Realm. In the context of a Realm that stays alive as long as there is JS running (e.g., on the Web, the Realm of a Worker), things like `Symbol.iterator` are analogous to primordials like `Object.prototype` and `Array.prototype`. Just because these will stay alive doesn't mean we disallow them as WeakMap keys.

While 'registered' symbols can be detected using `Symbol.keyFor`, there is currently no built in predicate to test if a symbol is 'well-known' or not. If 'well-known' symbols were not allowed as keys in a WeakMap code would need to ensure it handles this potential abrupt completion.

## Support for Symbols in WeakRefs and FinalizationRegistry

We should also support Symbols in WeakRefs and FinalizationRegistry. Not only is this consistent with Objects as WeakMap keys but it also enables user-land to build/demonstrate more advanced functionality. e.g. WeakMaps that support Records/Tuples as keys.

## Summing up

We think that adding Symbols as WeakMap keys is a useful, minimal primitive enabling Records and Tuples to reference Objects while respecting the constraints imposed by the goal to support membrane-based isolation within a single Realm. At the same time, the userspace solutions seem sufficient for many/most use cases; we believe that Records and Tuples are very useful without any additional mechanism for referencing objects from primitives, and therefore makes sense to proceed with Records and Tuples independently of this proposal.

[rtp]: https://github.com/tc39/proposal-record-tuple
