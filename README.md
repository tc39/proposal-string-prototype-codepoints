# String.prototype.codePoints

ECMAScript proposal for String.prototype.codePoints

## Status

The proposal is in stage 0 of [the TC39 process](https://tc39.github.io/process-document/).

## Motivation

Lexers for languages that involve code points above 0xFFFF (such as ECMAScript syntax itself), need
to be able to tokenise a string into separate code points before handling them with own state machine.

Currently language APIs provide two ways to access entire code points:

1. `codePointAt` allows to retrieve a code point at a known position. The issue is that position is usually unknown in advance if you're just iterating over the string, and you need to manually
calculate it on each iteration with a manual `for(;;)` loop and a magically looking expression like
`pos += currentCodePoint <= 0xFFFF ? 1 : 2`.
1. `String.prototype[Symbol.iterator]` which allows a hassle-free iteration over string codepoints,
but yields their string values, which are inefficient to work with in performance-critical lexers.

## Proposed solution

We propose the addition of a `codePoints()` method functionally similar to the `[@@iterator]`, but yielding numerical values of code points instead of string ones, this way combining the benefits of both approaches presented above while avoiding the related pitfalls in consumer code.

## Naming

The name and casing of `codePoints` was chosen to be consistent with existing `codePointAt` API.

## Illustrative examples

### Test if something is an identifier

```javascript
function isIdent(input) {
	let codePoints = input.codePoints();
	let first = codePoints.next();

	if (first.done || !isIdentifierStart(first.value)) {
		return false;
	}

	for (let cp of codePoints) {
		if (!isIdentifierContinue(cp)) {
			return false;
		}
	}

	return true;
}
```

### Tokenise a string with a state machine

```javascript
function toDigit(cp) {
	return cp - /* '0' */ 48;
}

function *tokenise(input) {
	let token = {};

	for (let cp of input) {
		let pos = /* see open question #1, we still need to know a pos somehow */;

		if (token.type === 'Identifier') {
			if (isIdentifierContinue(cp)) {
				continue;
			}
			token.end = pos;
			token.name = input.slice(token.start, token.end);
			yield token;
		} else if (token.type === 'Number') {
			if (isDigit(cp)) {
				token.value = token.value * 10 + toDigit(cp);
				continue;
			}
			token.end = pos;
			yield token;
		}

		if (isIdentifierStart(cp)) {
			token = { type: 'Identifier', start: pos };
		} else if (isDigit(cp)) {
			token = { type: 'Number', start: pos, value: toDigit(cp) };
		} else {
			throw new SyntaxError(`Expected an identifier or digit at ${tokenStart}`);
		}
	}
}
```

## Specification

You can view the rendered spec [here](https://rreverser.github.io/string-prototype-codepoints/).

## Open questions

1. [Should the API yield `[position, codePoint]` pairs like `entries` API of standard collections?](https://github.com/RReverser/string-prototype-codepoints/issues/1)
