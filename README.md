# String.prototype.codePoints

ECMAScript proposal for String.prototype.codePoints

## Status

The proposal is in stage 1 of [the TC39 process](https://tc39.github.io/process-document/).

## Motivation

Lexers for languages that involve code points above 0xFFFF (such as ECMAScript syntax itself), need
to be able to tokenise a string into separate code points before handling them with own state machine.

Currently language APIs provide two ways to access entire code points:

1.  `codePointAt` allows to retrieve a code point at a known position. The issue is that position is usually unknown in advance if you're just iterating over the string, and you need to manually
    calculate it on each iteration with a manual `for(;;)` loop and a magically looking expression like
    `pos += currentCodePoint <= 0xFFFF ? 1 : 2`.
1.  `String.prototype[Symbol.iterator]` which allows a hassle-free iteration over string codepoints,
    but yields their string values, which are inefficient to work with in performance-critical lexers, and still lack position information.

## Proposed solution

We propose the addition of a `codePoints()` method functionally similar to the `[@@iterator]`, but yielding positions and numerical values of code points instead of just string values, this way combining the benefits of both approaches presented above while avoiding the related pitfalls in consumer code.

## Naming

The name and casing of `codePoints` was chosen to be consistent with existing `codePointAt` API.

## Illustrative examples

### Test if something is an identifier

```javascript
function isIdent(input) {
    let codePoints = input.codePoints();
    let first = codePoints.next();

    if (first.done || !isIdentifierStart(first.value.codePoint)) {
        return false;
    }

    for (let { codePoint } of codePoints) {
        if (!isIdentifierContinue(codePoint)) {
            return false;
        }
    }

    return true;
}
```

### Full-blown tokeniser

```javascript
function toDigit(cp) {
    return cp - /* '0' */ 48;
}

// Generic helper
class LookaheadIterator {
    constructor(inner) {
        this[Symbol.iterator] = this;
        this.inner = inner;
        this.next();
    }

    next() {
        let next = this.lookahead;
        this.lookahead = this.inner.next();
        return next;
    }

    skipWhile(cond) {
        while (!this.lookahead.done && cond(this.lookahead.value.codePoint)) {
            this.next();
        }
        // even when `done == true`, the returned `.value.position` is still valid
        // and represents position at the end of the string
        return this.lookahead.value.position;
    }
}

// Main tokeniser
function* tokenise(input) {
    let iter = new LookaheadIterator(input.codePoints());

    for (let { position: start, codePoint } of iter) {
        if (isIdentifierStart(codePoint)) {
            yield {
                type: 'Identifier',
                start,
                end: iter.skipWhile(isIdentifierContinue)
            };
        } else if (isDigit(codePoint)) {
            yield {
                type: 'Number',
                start,
                end: iter.skipWhile(isDigit)
            };
        } else {
            throw new SyntaxError(`Expected an identifier or digit at ${start}`);
        }
    }
}
```

## FAQ

1. Why does iterator emit an object instead of an array like other key-value iterators?

    `[key, value]` format is usually used for entries of collections which can be directly indexed by `key`.

    Unlike those collections, strings in ECMAScript are indexed [as 16-bit units of UTF-16 text](https://tc39.github.io/ecma262/#sec-terms-and-definitions-string-value) and not code points, so emitted objects won't have consequent indices but rather positions which might be 1 or 2 16-bit units away from each other.

    To make the fact that they represent different measurement units and string representations explicit, we decided on `{ position, codePoint }` object format.

    See [#1](https://github.com/tc39/proposal-string-prototype-codepoints/issues/1) for more details.

1. What about iteration over different string representations - code units, grapheme clusters etc.?

    These are not covered by this particular proposal, but should be easy to add as separate methods or APIs. In particular, language-specific representations are being worked on as [`Intl.Segmenter` proposal](https://github.com/tc39/proposal-intl-segmenter).

## Specification

You can view the rendered spec [here](https://tc39.github.io/proposal-string-prototype-codepoints/).

## Implementations

- [Polyfill](https://github.com/zloirock/core-js#stringcodepoints)
