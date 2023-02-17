# Progress allowing RegExps

## Status

Champion(s): *champion name(s)*
Author(s): Bradley Meck Farias (Socket)
Stage: -1

## Motivation

JS RegExp requires a full input to perform a match. Common JS idioms include streaming and async chunked I/O. This means that for streaming inputs that come in chunks, any input that does not create a match may potentially need to be buffered and concatenated. Even when a match is found, it is unsafe to slice the string due to a variety of reasons.

### Motivating Examples

The goal of this proposal is to solve the variety of scenarios listed here.

The standard workflow of a stream of data from JS I/O can be imagined as follows. Showing a problem once chunking is involved using current RegExp methods:

```mjs
const pattern = /ab/g
async function* stream() {
  yield 'never needed but kept all the same.'
  yield 'a'
  yield 'b'
}
for await (const chunk of stream()) {
  chunk.matchAll(pattern) // never matches
}
// but if we buffer
let buffer = ''
for await (const chunk of stream()) {
  buffer += chunk
  pattern.exec(buffer) // matches
}
```

This leads to a few direct effects:

* Bloated memory usage due to inability to GC chunks
* Underutilized CPU during I/O idle
* Needing to create extra ticks just to append strings to no effect
* Loss of RegExp matching state on each search

This also has some indirect effects:

#### Complexity of false negatives with lone surrogates / in-progress quantifiers

Surrogates are a simple form, but even things like removing `/ab+/g` has problems with chunking.

```mjs
const fire_surrogates = '🔥'.split('')
const emoji_pattern = /.../g
const non_emoji_str = ''
for (const chunk of fire_surrogates) {
  non_emoji_str += chunk.replaceAll(emoji_pattern, '')
}
```

#### Problems parallelizing/caching results

Often for long lived parses it is desirable to save data in a way to reduce re-processing. Existing means to do this are fragile at best and do not work with complex RegExp.

```mjs
const progress_pattern = /(?=$|bc?)/g
const chunks = 'abc'.split(progress_pattern) // ['a', 'bc']
// can skip processing for 'a' of 'abc' for future chunks
// this approach currently isn't possible for complex regexp
```

#### Problem knowing starting range of search

Due to lack of knowledge of lookbehind constant buffering must be used to safely account for it.

```mjs
const pattern = /(?<a=)ba/g
let chunks = ['ab', 'a', 'ba']
let buffer = ''
for (const chunk of chunks) {
  // lookbehind means buffering it all even if match found, cannot drop 'aba' since last 'a' is needed
  buffer += chunk
  // state loss due to new string ref and possibly
  // repetitive searching from start of string
  const matches = buffer.matchAll(pattenr)
}
```

#### Problem knowing incomplete quantifier vs EOF

```mjs
const pattern = /ab+(?=b)/g
let chunks = ['ab', 'bb']
let buffer = ''
for (const chunk of chunks) {
  buffer += chunk
  const match
  if (pattern.lastIndex === buffer.length) {
    // may be unfinished, due to lookahead aggreagation
  } else {
    // cannot slice buffer due to lookbehind again
  }
}
```

## Use cases

**Streaming resource utilization**: when doing I/O in JS it often comes that data is streamed in chunks, from websockets, HTTP requests, file I/O, etc. Doing so helps to allow forward progress while waiting on I/O allowing CPU pressure and memory pressure to be spread out during I/O gaps. RegExp should be able to be account for streaming in a way that allows forward progress as well as releaving memory pressure.

**Streaming data boundaries**: when processing chunked data RegExp should allow for handling patterns around data boundaries in a way that prevents false positives or negatives from having incomplete data.

**Text manipulation pattern**: when performing matching, it can be valuable to know if a chunk of data will potentially be used in a future match and if so what parts. This can aid in various text manipulation or viewing that requires scanning a full text of a document by allowing it to be split up into discrete chunks and processed async/in parallel.

## Comparison

- Java `Matcher.hitEnd` https://docs.oracle.com/javase/9/docs/api/java/util/regex/Matcher.html#hitEnd--

Uses a matcher result separated from the pattern. Lacks info on where in chunk to resume next attempt for things like lookbehind. Requires other class like Scanner to mitigate bookkeeping.

- Python `regex` `partial=True` https://github.com/mrabarnett/mrab-regex#added-partial-matches-hg-issue-102

Uses a matcher result separated from the pattern. Lacks info on where in chunk to resume next attempt for things like lookbehind. Requires manual bookkeeping for buffering string properly.

- C++ `boost/regex` https://www.boost.org/doc/libs/1_81_0/libs/regex/doc/html/boost_regex/partial_matches.html

Uses a matcher result separated from the pattern. Lacks info on where in chunk to resume next attempt for things like lookbehind. Requires manual bookkeeping for buffering string properly.

- PCRE2 Multi Segment Support https://pcre.org/current/doc/html/pcre2partial.html#SEC6

Manages all state on the pattern itself. Lacks info on where in chunk to resume next attempt for things like lookbehind. Requires manual bookkeeping for buffering results properly.

## Implementations

### Polyfill/transpiler implementations

N/A

### Native implementations

N/A

## Q&A

*Frequently asked questions, or questions you think might be asked. Issues on the issue tracker or questions from past reviews can be a good source for these.*

**Q**: Why is the proposal this way?

**A**: Because reasons!

**Q**: Why does this need to be built-in, instead of being implemented in JavaScript?

**A**: We could encourage people to continue doing this in user-space. However, that would significantly increase load time of web pages. Additionally, web browsers already have a built-in frobnicator which is higher quality.

**Q**: Is it really necessary to create such a high-level built-in construct, rather than using lower-level primitives?

**A**: Instead of providing a direct `frobnicate` method, we could expose more basic primitives to compose an md5 hash with rot13. However, rot13 was demonstrated to be insecure in 2012 (citation), so exposing it as a primitive could serve as a footgun.