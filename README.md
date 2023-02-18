# Progress allowing RegExps

## Status

Champion(s): *champion name(s)*
Author(s): Bradley Meck Farias (Socket)
Stage: -1

## Motivation

JS RegExp requires complete input to perform a match. Common JS idioms include streaming and async chunked I/O. This means that for streaming inputs that come in chunks, any input that does not create a match may potentially need to be buffered and concatenated. Even when a match is found, it is unsafe to slice the string due to a variety of reasons.

### Motivating Examples

The goal of this proposal is to address the scenarios listed here.

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
// but if we buffer...
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
* Loss of pattern matching state on each search

This also has some indirect effects:

#### False negatives

```mjs
const fire_surrogates = '🔥'.split('')
const emoji_pattern = /\p{Emoji_Presentation}/gu
const text_presentation_str = ''
for await (const chunk of fire_surrogates) {
  text_presentation_str += chunk.replaceAll(emoji_pattern, '�') // never matches
}
```

#### False positives

See [Scunthorpe problem](https://en.wikipedia.org/wiki/Scunthorpe_problem).

```mjs
const pattern = new RegExp(String.raw`\b(${DIRTY_WORDS.join('|')})\b`, 'gi')
async function* stream(str, size) {
  while (str.length) {
    yield str.slice(0, size)
    str = str.slice(size)
  }
}
let reject = false;
for await (const chunk of stream('shitakemushrooms.example.com', 4)) {
  if (pattern.test(chunk)) {
    reject = true // misclassifies based on the first chunk
    break
  }
}
```

#### Problems parallelizing/caching results

Often for long lived parses it is desirable to save data in a way that minimizes re-processing. Existing means to do this are fragile at best and do not work with complex patterns.

```mjs
const progress_pattern = /(?=$|bc?)/g
const chunks = 'abc'.split(progress_pattern) // ['a', 'bc']
// can skip processing for 'a' of 'abc' for future chunks
// this approach currently isn't possible for complex patterns
```

#### Problem knowing starting range of search

Due to lack of knowledge regarding lookbehind, constant buffering must be used to safely account for it.

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

**Streaming resource utilization**: Data is often streamed to JS in chunks from websockets, HTTP messages, file reading, etc. This generally supports forward progress while waiting on I/O, spreading out CPU and memory pressure during gaps. RegExp should be able to effectively participate in such processes.

**Streaming data boundaries**: When processing chunked data, RegExp should allow for handling patterns around data boundaries in a way that prevents false positives or false negatives on incomplete data.

**Text manipulation pattern**: When performing matching, it can be valuable to know if a chunk of data will potentially be used in a future match and if so what parts. This can aid in various text manipulation or viewing use cases by allowing complete text to be chunked and processed async/in parallel.

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

**A**: We could encourage people to continue doing this in user-space. However, that would significantly increase load time of web pages. Additionally, web browsers already have a built-in RegExp engine which is higher quality.

**Q**: Is it really necessary to create such a high-level built-in construct, rather than using lower-level primitives?

**A**: Low level constructs and book keeping are the general source of the issues and don't allow for the same fidelity of results or robust handling of edge cases.
