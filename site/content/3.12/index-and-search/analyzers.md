---
title: Transforming data with Analyzers
menuTitle: Analyzers
weight: 160
description: >-
  Analyzers allow you to transform data, for sophisticated text processing and
  searching, either standalone or in combination with Views and inverted indexes
---
While AQL string functions allow for basic text manipulation, true text
processing including tokenization, language-specific word stemming, case
conversion and removal of diacritical marks (accents) from characters only
become possible with Analyzers.

Analyzers parse input values and transform them into sets of sub-values,
for example by breaking up text into words. If they are used in Views then
the documents' attribute values of the linked collections are used as input
and additional metadata is produced internally. The data can then be used for
searching and sorting to provide the most appropriate match for the specified
conditions, similar to queries to web search engines.

Analyzers can be used on their own to tokenize and normalize strings in AQL
queries with the [`TOKENS()` function](../aql/functions/string.md#tokens).
The following example shows the creation of a custom Analyzer and how it
transforms an example input:

```js
---
name: analyzerCustomTokens
description: ''
---
var analyzers = require("@arangodb/analyzers")
var a = analyzers.save("custom", "text", {
  locale: "en",
  stopwords: ["a", "example"]
}, []);
db._query(`RETURN TOKENS("UPPER & lower, a Stemming Example.", "custom")`).toArray();
~analyzers.remove(a.name);
```

How Analyzers process values depends on their type and configuration.
The configuration is comprised of type-specific properties and list of features.
The features control the additional metadata to be generated to augment View
indexes, to be able to rank results for instance.

Analyzers can be managed via an [HTTP API](../develop/http-api/analyzers.md) and through
a [JavaScript module](../develop/javascript-api/analyzers.md).

{{< youtube id="tbOTYL26reg" >}}

## Value Handling

While most of the Analyzer functionality is geared towards text processing,
there is no restriction to strings as input data type when using them through
Views or inverted indexes – your documents may have attributes of any data type
after all.

Strings are processed according to the Analyzer, whereas other primitive data
types (`null`, `true`, `false`, numbers) are generally left unchanged.
Exceptions are Analyzers that specifically work with other data types, like
geo-spatial or query-based Analyzers.

The elements of arrays are processed individually, regardless of the level of
nesting, if you use Analyzers stand-alone. That is, strings are processed by the
configured Analyzer(s) and other primitive values are returned as-is.
This also applies if you use Analyzers in `arangosearch` Views, or in
`search-alias` Views with inverted indexes that have the `searchField` option
enabled. The array elements are unpacked, processed, and indexed individually.

If you use inverted indexes with the `searchField` option disabled, optionally
through `search-alias` Views, array elements are not unpacked by default. Most
Analyzers do not accept arrays as input in this context. You can unpack one
array level and let the configured Analyzer process the individual elements by
using `[*]` as a suffix for a field in the index definition. Primitive values
other than strings are indexed as-is.

Analyzers cannot process objects as a whole. However, you can work with
individual object attributes. You can use inverted indexes and Views to index
specific object attributes or sub-attributes, or index all sub-attributes with
the `includeAllFields` option enabled. Each non-object value is handled as
described above. Sub-objects in arrays can be indexed, too (with limitations).
However, only primitive values are added to the index. Arrays and objects
cannot be searched for as a whole.

Also see:
- [`SEARCH` operation](../aql/high-level-operations/search.md) on how to query indexed
  values such as numbers and nested values
- [`arangosearch` Views](arangosearch/arangosearch-views-reference.md) and
  [Inverted indexes](indexing/working-with-indexes/inverted-indexes.md) for details about how
  compound data types (arrays, objects) get indexed

## Analyzer Names

Each Analyzer has a name for identification with the following
naming constraints:

- The name must only consist of the letters `a` to `z` (both in lower and
  upper case), the numbers `0` to `9`, underscore (`_`) and dash (`-`) symbols.
  This also means that any non-ASCII names are not allowed.
- It must always start with a letter.
- The maximum allowed length of a name is 254 bytes.
- Analyzer names are case-sensitive.

Custom Analyzers are stored per database, in a system collection `_analyzers`.
The names get prefixed with the database name and two colons, e.g.
`myDB::customAnalyzer`.This does not apply to the globally available
[built-in Analyzers](#built-in-analyzers), which are not stored in an
`_analyzers` collection.

Custom Analyzers stored in the `_system` database can be referenced in queries
against other databases by specifying the prefixed name, e.g.
`_system::customGlobalAnalyzer`. Analyzers stored in databases other than
`_system` cannot be accessed from within another database however.

## Analyzer Types

The following Analyzer types are available:

- [`identity`](#identity): treats value as atom (no transformation)
- [`delimiter`](#delimiter): splits into tokens at a user-defined character sequence
- [`multi_delimiter`](#multi_delimiter): splits into tokens at user-defined character sequences
- [`stem`](#stem): applies stemming to the value as a whole
- [`norm`](#norm): applies normalization to the value as a whole
- [`ngram`](#ngram): creates _n_-grams from the value with user-defined lengths
- [`text`](#text): tokenizes text strings into words, optionally with stemming,
  normalization, stop-word filtering and edge _n_-gram generation
- [`segmentation`](#segmentation): tokenizes text in a language-agnostic manner,
  optionally with normalization
- [`wildcard`](#wildcard): can apply another Analyzer and creates _n_-grams to
  enable fast partial matching for large strings
- [`aql`](#aql): runs an AQL query to prepare tokens for index
- [`pipeline`](#pipeline): chains multiple Analyzers
- [`stopwords`](#stopwords): removes the specified tokens from the input
- [`collation`](#collation): respects the alphabetic order of a language in range queries
- [`minhash`](#minhash): applies another Analyzer and then a locality-sensitive
  hash function, to find candidates for set comparisons based on the
  Jaccard index
- [`classification`](#classification): classifies the input text using a
  word embedding model
- [`nearest_neighbors`](#nearest_neighbors): finds tokens similar to the ones
  in the input text using a word embedding model
- [`geojson`](#geojson): breaks up a GeoJSON object into a set of indexable tokens
- [`geo_s2`](#geo_s2): like `geojson` but offers more efficient formats for
  indexing geo-spatial data
- [`geopoint`](#geopoint): breaks up JSON data describing a coordinate pair into
  a set of indexable tokens

The following table compares the Analyzers for **text processing**:

Analyzer  /  Capability                   | Tokenization | Stemming | Normalization | _N_-grams
:----------------------------------------:|:------------:|:--------:|:-------------:|:--------:
[`stem`](#stem)                           |      No      |   Yes    |      No       |   No
[`norm`](#norm)                           |      No      |    No    |     Yes       |   No
[`ngram`](#ngram)                         |      No      |    No    |      No       |  Yes
[`text`](#text)                           |     Yes      |   Yes    |     Yes       | (Yes)
[`segmentation`](#segmentation)           |     Yes      |    No    |     Yes       |   No

The available normalizations are case conversion and accents/diacritics removal.
The `segmentation` Analyzer only supports case conversion.

The `text` Analyzer supports edge _n_-grams but not full _n_-grams.

## Tokenization 

The `text` and `segmentation` Analyzer types tokenize text into words (or a
comparable concept of a word). See
[Word Boundaries](https://www.unicode.org/reports/tr29/#Word_Boundaries)
in the Unicode Standard Annex #29 about Unicode text segmentation for details.

These tokenizing Analyzers extract tokens, which removes characters like
punctuation and whitespace. An exception is the [`segmentation` Analyzer](#segmentation)
if you select `"graphic"` or `"all"` for the `break` option. They preserve `@`
and `.` characters of email addresses, for instance. There are also exceptions
with both Analyzer types for sequences like numbers, for which decimal and
thousands separators (`.` and `,`) are preserved.

## Normalization

The `norm`, `text`, and `segmentation` Analyzer types allow you to convert the
input text to all lowercase or all uppercase for normalization purposes, namely
case insensitive search. Case folding is not supported. Also see
[Case Mapping](https://unicode-org.github.io/icu/userguide/transforms/casemappings.html)
in the ICU documentation.

The `norm` and `text` Analyzer types also allow you to convert characters with
diacritical marks to the base characters. This normalization enables
accent-insensitive search.

## Analyzer Features

The *features* of an Analyzer determine what searching capabilities are
available and are only applicable in the context of Views and inverted indexes.

The valid values for the features are dependant on both the capabilities of
the underlying Analyzer *type* and the query filtering and sorting functions that the
result can be used with. For example, the `text` type produces
`frequency` + `norm` + `position`, and the `PHRASE()` AQL function requires
`frequency` + `position` to be available.

{{< tip >}}
You should only enable the features you require, as there is a cost associated
with them. The metadata they produce needs to be computed and stored, requiring
time and disk space.

The examples in the documentation only set the required features for the shown
examples, which is often none (empty array `[]` in the call of `analyzers.save()`).
{{< /tip >}}

The following *features* are supported:

- **frequency**: track how often a term occurs.
  Required for [`PHRASE()`](../aql/functions/arangosearch.md#phrase),
  [`NGRAM_MATCH()`](../aql/functions/arangosearch.md#ngram_match),
  [`BM25()`](../aql/functions/arangosearch.md#bm25),
  [`TFIDF()`](../aql/functions/arangosearch.md#tfidf), and
  [`OFFSET_INFO()`](../aql/functions/arangosearch.md#offset_info).
- **norm**: calculate and store the field normalization factor that is used to
  score fairer if the same term is repeated, reducing its importance.
  Required for [`BM25()`](../aql/functions/arangosearch.md#bm25)
  (except BM15) and [`TFIDF()`](../aql/functions/arangosearch.md#tfidf)
  (if called with normalization enabled). It is recommended to enable this
  feature for custom Analyzers.
- **position**: enumerate the tokens for position-dependent queries. Required
  for [`PHRASE()`](../aql/functions/arangosearch.md#phrase),
  [`NGRAM_MATCH()`](../aql/functions/arangosearch.md#ngram_match), and
  [`OFFSET_INFO()`](../aql/functions/arangosearch.md#offset_info).
  If present, then the `frequency` feature is also required.
- **offset**: enable search highlighting capabilities.
  Required for [`OFFSET_INFO()`](../aql/functions/arangosearch.md#offset_info).
  If present, then the `position` and `frequency` features are also required.

## Analyzer Properties

The valid attributes/values for the *properties* are dependant on what *type*
is used. For example, the `delimiter` type needs to know the desired delimiting
character(s), whereas the `text` type takes a locale, stop-words and more.

### `identity`

An Analyzer applying the `identity` transformation, i.e. returning the input
unmodified.

It does not support any *properties* and will ignore them.

**Examples**

Applying the identity Analyzers does not perform any transformations, hence
the input is returned unaltered:

```js
---
name: analyzerIdentity
description: ''
---
db._query(`RETURN TOKENS("UPPER lower dïäcríticš", "identity")`).toArray();
```

### `delimiter`

An Analyzer capable of breaking up delimited text into tokens.

It follows [RFC 4180](https://tools.ietf.org/html/rfc4180) but without starting
new records on newlines and letting you freely choose the delimiter. You can
wrap tokens in the input string in double quote marks to quote the delimiter.
For example, a `delimiter` Analyzer that uses `,` as delimiter and an input
string of `foo,"bar,baz"` results in the tokens `foo` and `bar,baz` instead of
`foo`, `bar`, and `baz`.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `delimiter` (string): the delimiting character or character sequence.
  The whole string is considered as one delimiter.

**Examples**

```js
---
name: analyzerDelimiter1
description: Split comma-separated text into tokens but do not split quoted fields
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("delimiter_csv", "delimiter", {
  delimiter: ","
}, []);
db._query(`RETURN TOKENS('foo,bar,baz,"bar,baz"', "delimiter_csv")`).toArray();
~analyzers.remove(a.name);
```

```js
---
name: analyzerDelimiter2
description: >
  Split input strings into tokens at every character sequence of hyphen-minus,
  right angled bracket, and a space
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("delimiter_arrow", "delimiter", {
  delimiter: "-> "
}, []);
db._query(`RETURN TOKENS("some-> hand-picked-> words", "delimiter_arrow")`).toArray();
~analyzers.remove(a.name);
```

### `multi_delimiter`

An Analyzer capable of breaking up text into tokens using multiple delimiters.

Unlike with the `delimiter` Analyzer, the `multi_delimiter` Analyzer does not
support quoting fields.


The *properties* allowed for this Analyzer are an object with the following
attributes:

- `delimiters` (array): a list of strings of which each is considered as one
  delimiter that can be one or multiple characters long. The delimiters must not
  overlap, which means that a delimiter cannot be a prefix of another delimiter.

**Examples**

```js
---
name: analyzerMultiDelimiter1
description: >
  Split at delimiting characters `,` and `;`, as well as the character sequence
  `||` but not a single `|` character
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("delimiter_multiple", "multi_delimiter", {
  delimiters: [",", ";", "||"]
}, []);
db._query(`RETURN TOKENS("differently,delimited;words||one|token", "delimiter_multiple")`).toArray();
~analyzers.remove(a.name);
```

```js
---
name: analyzerMultiDelimiter2
description: Double quote marks have no effect on the splitting
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("delimiter_noquote", "multi_delimiter", {
  delimiters: [","]
}, []);
db._query(`RETURN TOKENS('foo,bar,baz,"bar,baz"', "delimiter_noquote")`).toArray();
~analyzers.remove(a.name);
```

### `stem`

An Analyzer capable of stemming the text, treated as a single token,
for supported languages.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `locale` (string): a locale in the format `language`, e.g. `"de"` or `"en"`.
  The locale is forwarded to the Snowball stemmer without checks.
  An invalid locale does not prevent the creation of the Analyzer.
  Also see [Supported Languages](#supported-languages).

**Examples**

Apply stemming to the input string as a whole:

```js
---
name: analyzerStem
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("stem_en", "stem", {
  locale: "en"
}, []);
db._query(`RETURN TOKENS("databases", "stem_en")`).toArray();
~analyzers.remove(a.name);
```

### `norm`

An Analyzer capable of normalizing the text, treated as a single
token, i.e. case conversion and accent removal.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `locale` (string): a locale in the format `language[_COUNTRY][_VARIANT]`
  (square brackets denote optional parts), e.g. `"de"`, `"en_US"`, or `"es__TRADITIONAL"`.
  See the [ICU Documentation](https://unicode-org.github.io/icu/userguide/locale/)
  for details. The locale is forwarded to ICU without checks.
  An invalid locale does not prevent the creation of the Analyzer.
  Also see [Supported Languages](#supported-languages).
- `accent` (boolean, _optional_):
  - `true` to preserve accented characters (default)
  - `false` to convert accented characters to their base characters
- `case` (string, _optional_):
  - `"lower"` to convert to all lower-case characters
  - `"upper"` to convert to all upper-case characters
  - `"none"` to not change character case (default)

**Examples**

Convert input string to all upper-case characters:

```js
---
name: analyzerNorm1
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("norm_upper", "norm", {
  locale: "en",
  case: "upper"
}, []);
db._query(`RETURN TOKENS("UPPER lower dïäcríticš", "norm_upper")`).toArray();
~analyzers.remove(a.name);
```

Convert accented characters to their base characters:

```js
---
name: analyzerNorm2
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("norm_accent", "norm", {
  locale: "en",
  accent: false
}, []);
db._query(`RETURN TOKENS("UPPER lower dïäcríticš", "norm_accent")`).toArray();
~analyzers.remove(a.name);
```

Convert input string to all lower-case characters and remove diacritics:

```js
---
name: analyzerNorm3
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("norm_accent_lower", "norm", {
  locale: "en",
  accent: false,
  case: "lower"
}, []);
db._query(`RETURN TOKENS("UPPER lower dïäcríticš", "norm_accent_lower")`).toArray();
~analyzers.remove(a.name);
```

### `ngram`

An Analyzer capable of producing _n_-grams from a specified input in a range of
min..max (inclusive). Can optionally preserve the original input.

This Analyzer type can be used to implement substring matching.
Note that it slices the input based on bytes and not characters by default
(*streamType*). The *"binary"* mode supports single-byte characters only;
multi-byte UTF-8 characters raise an *Invalid UTF-8 sequence* query error.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `min` (number): unsigned integer for the minimum _n_-gram length
- `max` (number): unsigned integer for the maximum _n_-gram length
- `preserveOriginal` (boolean):
  - `true` to include the original value as well
  - `false` to produce the _n_-grams based on *min* and *max* only
- `startMarker` (string, _optional_): this value will be prepended to _n_-grams
  which include the beginning of the input. Can be used for matching prefixes.
  Choose a character or sequence as marker which does not occur in the input.
- `endMarker` (string, _optional_): this value will be appended to _n_-grams
  which include the end of the input. Can be used for matching suffixes.
  Choose a character or sequence as marker which does not occur in the input.
- `streamType` (string, _optional_): type of the input stream
  - `"binary"`: one byte is considered as one character (default)
  - `"utf8"`: one Unicode codepoint is treated as one character

**Examples**

With *min* = `4` and *max* = `5`, the Analyzer will produce the following
_n_-grams for the input string `"foobar"`:
- `"foob"`
- `"fooba"`
- `"foobar"` (if *preserveOriginal* is enabled)
- `"ooba"`
- `"oobar"`
- `"obar"`

An input string `"foo"` will not produce any _n_-gram unless *preserveOriginal*
is enabled, because it is shorter than the *min* length of 4.

Above example but with *startMarker* = `"^"` and *endMarker* = `"$"` would
produce the following:
- `"^foob"`
- `"^fooba"`
- `"^foobar"` (if *preserveOriginal* is enabled)
- `"foobar$"` (if *preserveOriginal* is enabled)
- `"ooba"`
- `"oobar$"`
- `"obar$"`

Create and use a trigram Analyzer with `preserveOriginal` disabled:

```js
---
name: analyzerNgram1
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("trigram", "ngram", {
  min: 3,
  max: 3,
  preserveOriginal: false,
  streamType: "utf8"
}, []);
db._query(`RETURN TOKENS("foobar", "trigram")`).toArray();
~analyzers.remove(a.name);
```

Create and use a bigram Analyzer with `preserveOriginal` enabled and with start
and stop markers:

```js
---
name: analyzerNgram2
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("bigram_markers", "ngram", {
  min: 2,
  max: 2,
  preserveOriginal: true,
  startMarker: "^",
  endMarker: "$",
  streamType: "utf8"
}, []);
db._query(`RETURN TOKENS("foobar", "bigram_markers")`).toArray();
~analyzers.remove(a.name);
```

### `text`

An Analyzer capable of breaking up strings into individual words while also
optionally filtering out stop-words, extracting word stems, applying
case conversion and accent removal.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `locale` (string): a locale in the format `language[_COUNTRY][_VARIANT]`
  (square brackets denote optional parts), e.g. `"de"`, `"en_US"`, or `"es__TRADITIONAL"`.
  See the [ICU Documentation](https://unicode-org.github.io/icu/userguide/locale/)
  for details. The locale is forwarded to ICU without checks.
  An invalid locale does not prevent the creation of the Analyzer.
  Also see [Supported Languages](#supported-languages).
- `accent` (boolean, _optional_):
  - `true` to preserve accented characters
  - `false` to convert accented characters to their base characters (default)
- `case` (string, _optional_):
  - `"lower"` to convert to all lower-case characters (default)
  - `"upper"` to convert to all upper-case characters
  - `"none"` to not change character case
- `stemming` (boolean, _optional_):
  - `true` to apply stemming on returned words (default)
  - `false` to leave the tokenized words as-is
- `edgeNgram` (object, _optional_): if present, then edge _n_-grams are generated
  for each token (word). That is, the start of the _n_-gram is anchored to the
  beginning of the token, whereas the `ngram` Analyzer would produce all
  possible substrings from a single input token (within the defined length
  restrictions). Edge _n_-grams can be used to cover word-based auto-completion
  queries with an index, for which you should set the following other options:
  `accent: false`, `case: "lower"` and most importantly `stemming: false`.
  - `min` (number, _optional_): minimal _n_-gram length
  - `max` (number, _optional_): maximal _n_-gram length
  - `preserveOriginal` (boolean, _optional_): whether to include the original
    token even if its length is less than *min* or greater than *max*
- `stopwords` (array, _optional_): an array of strings with words to omit
  from result. Default: load words from `stopwordsPath`. To disable stop-word
  filtering provide an empty array `[]`. If both `stopwords` and
  `stopwordsPath` are provided then both word sources are combined.
- `stopwordsPath` (string, _optional_): path with a *language* sub-directory
  (e.g. `en` for a locale `en_US`) containing files with words to omit.
  Each word has to be on a separate line. Everything after the first whitespace
  character on a line will be ignored and can be used for comments. The files
  can be named arbitrarily and have any file extension (or none).

  Default: if no path is provided then the value of the environment variable
  `IRESEARCH_TEXT_STOPWORD_PATH` is used to determine the path, or if it is
  undefined then the current working directory is assumed. If the `stopwords`
  attribute is provided then no stop-words are loaded from files, unless an
  explicit `stopwordsPat` is also provided.

  Note that if the `stopwordsPath` cannot be accessed, is missing language
  sub-directories or has no files for a language required by an Analyzer,
  then the creation of a new Analyzer is refused. If such an issue is 
  discovered for an existing Analyzer during startup then the server will
  abort with a fatal error.

The Analyzer uses a fixed order of operations:

1. Tokenization
2. Accent removal (if `accent` is set to `false`)
3. Case conversion (unless `case` is set to `none`)
4. Stop word removal (if any are defined)
5. Word stemming (if `stemming` is set to `true`)

If you require a different order, consider using a [`pipeline` Analyzer](#pipeline).

Stop words are removed after case/accent operations but before stemming.
The reason is that stemming could map multiple words to the same one, and you
would not be able to filter out specific words only.

The case/accent operations are not applied to the stop words for performance
reasons. You need to pre-process them accordingly, for example, using the
[`TOKENS()` function](../aql/functions/string.md#tokens) with a
[`text` Analyzer](#text) that has the same `locale`, `case`, and `accent`
settings as the planned `text` Analyzer, but with `stemming` set to `false` and
`stopwords` set to `[]`.

**Examples**

The built-in `text_en` Analyzer has stemming enabled (note the word endings):

```js
---
name: analyzerTextStem
description: ''
---
db._query(`RETURN TOKENS("Crazy fast NoSQL-database!", "text_en")`).toArray();
```

You may create a custom Analyzer with the same configuration but with stemming
disabled like this:

```js
---
name: analyzerTextNoStem
description: ''
---
var analyzers = require("@arangodb/analyzers")
var a = analyzers.save("text_en_nostem", "text", {
  locale: "en",
  case: "lower",
  accent: false,
  stemming: false,
  stopwords: []
}, [])
db._query(`RETURN TOKENS("Crazy fast NoSQL-database!", "text_en_nostem")`).toArray();
~analyzers.remove(a.name);
```

Custom text Analyzer with the edge _n_-grams capability and normalization enabled,
stemming disabled and `"the"` defined as stop-word to exclude it:

```js
---
name: analyzerTextEdgeNgram
description: ''
---
~var analyzers = require("@arangodb/analyzers")
var a = analyzers.save("text_edge_ngrams", "text", {
  edgeNgram: { min: 3, max: 8, preserveOriginal: true },
  locale: "en",
  case: "lower",
  accent: false,
  stemming: false,
  stopwords: [ "the" ]
}, []);
db._query(`RETURN TOKENS(
  "The quick brown fox jumps over the dogWithAVeryLongName",
  "text_edge_ngrams"
)`).toArray();
~analyzers.remove(a.name);
```

### `collation`

<small>Introduced in: v3.9.0</small>

An Analyzer capable of converting the input into a set of language-specific
tokens. This makes comparisons follow the rules of the respective language,
most notable in range queries against Views.

For example, the Swedish alphabet has 29 letters: `a` to `z` plus `å`, `ä`, and
`ö`, in that order. Using a Swedish locale (like `sv`), the sorting order is
`å` after `z`, whereas using an English locale (like `en`), it is `å` after `a`.
This impacts queries with `SEARCH` expressions like `doc.text < "c"`, excluding
`å` when using a Swedish locale but including it when using an English locale.

{{< info >}}
Sorting by the output of the `collation` Analyzer like
`SORT TOKENS(<text>, <collationAnalyzer>)` is not a supported feature and
doesn't produce meaningful results.
{{< /info >}}

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `locale` (string): a locale in the format
  `language[_COUNTRY][_VARIANT][@keywords]` (square brackets denote optional parts),
  e.g. `"de"`, `"en_US"`, `"es__TRADITIONAL"`, or `fr@collation=phonebook`. See the
  [ICU Documentation](https://unicode-org.github.io/icu/userguide/locale/)
  for details. The locale is forwarded to ICU without checks.
  An invalid locale does not prevent the creation of the Analyzer.
  Also see [Supported Languages](#supported-languages).

**Examples**

In Swedish, the letter `å` (note the small circle above the `a`) comes after
`z`. Other languages treat it like a regular `a`, putting it before `b`.
Below example creates two `collation` Analyzers, one with an English locale
(`en`) and one with a Swedish locale (`sv`). It then demonstrates the
difference in alphabetical order using a simple range query that returns
letters before `c`:

```js
---
name: analyzerCollation
description: ''
---
var analyzers = require("@arangodb/analyzers");
var en = analyzers.save("collation_en", "collation", { locale: "en" }, []);
var sv = analyzers.save("collation_sv", "collation", { locale: "sv" }, []);
var test = db._create("test");
var docs = db.test.save([
  { text: "a" },
  { text: "å" },
  { text: "b" },
  { text: "z" },
]);
var view = db._createView("view", "arangosearch",
  { links: { test: { analyzers: [ "collation_en", "collation_sv" ], includeAllFields: true }}});
~assert(db._query(`FOR d IN view COLLECT WITH COUNT INTO c RETURN c`).toArray()[0] === 4);
db._query("FOR doc IN view SEARCH ANALYZER(doc.text < TOKENS('c', 'collation_en')[0], 'collation_en') RETURN doc.text").toArray();
db._query("FOR doc IN view SEARCH ANALYZER(doc.text < TOKENS('c', 'collation_sv')[0], 'collation_sv') RETURN doc.text").toArray();
~db._dropView(view.name());
~db._drop(test.name());
~analyzers.remove(en.name);
~analyzers.remove(sv.name);
```

### `aql`

<small>Introduced in: v3.8.0</small>

An Analyzer capable of running a restricted AQL query to perform
data manipulation / filtering.

The query must not access the storage engine. This means no `FOR` loops over
collections or Views, no use of the `DOCUMENT()` function, no graph traversals.
AQL functions are allowed as long as they do not involve Analyzers (`TOKENS()`,
`PHRASE()`, `NGRAM_MATCH()`, `ANALYZER()` etc.) or data access, and if they can
be run on DB-Servers in case of a cluster deployment. User-defined functions
are not permitted.

The input data is provided to the query via a bind parameter `@param`.
It is always a string. The AQL query is invoked for each token in case of
multiple input tokens, such as an array of strings.

The output can be one or multiple tokens (top-level result elements). They get
converted to the configured `returnType`, either booleans, numbers or strings
(default).

{{< tip >}}
If `returnType` is `"number"` or `"bool"` then it is unnecessary to set this
AQL Analyzer as context Analyzer with `ANALYZER()` in View queries. You can
compare indexed fields to numeric values, `true` or `false` directly, because
they bypass Analyzer processing.
{{< /tip >}}

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `queryString` (string): AQL query to be executed
- `collapsePositions` (boolean):
  - `true`: set the position to 0 for all members of the query result array
  - `false` (default): set the position corresponding to the index of the
    result array member
- `keepNull` (boolean):
  - `true` (default): treat `null` like an empty string
  - `false`: discard `null`s from View index. Can be used for index filtering
    (i.e. make your query return null for unwanted data). Note that empty
    results are always discarded.
- `batchSize` (integer): number between 1 and 1000 (default = 1) that
  determines the batch size for reading data from the query. In general, a
  single token is expected to be returned. However, if the query is expected
  to return many results, then increasing `batchSize` trades memory for
  performance.
- `memoryLimit` (integer): memory limit for query execution in bytes.
  (default is 1048576 = 1Mb) Maximum is 33554432U (32Mb)
- `returnType` (string): data type of the returned tokens. If the indicated
  type does not match the actual type then an implicit type conversion is
  applied (see [`TO_STRING()`](../aql/functions/type-check-and-cast.md#to_string),
  [`TO_NUMBER()`](../aql/functions/type-check-and-cast.md#to_number),
  [`TO_BOOL()`](../aql/functions/type-check-and-cast.md#to_bool))
  - `"string"` (default): convert emitted tokens to strings
  - `"number"`: convert emitted tokens to numbers
  - `"bool"`: convert emitted tokens to booleans

**Examples**

Soundex Analyzer for a phonetically similar term search:

```js
---
name: analyzerAqlSoundex
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("soundex", "aql", { queryString: "RETURN SOUNDEX(@param)" }, []);
db._query("RETURN TOKENS('ArangoDB', 'soundex')").toArray();
~analyzers.remove(a.name);
```

Concatenating Analyzer for conditionally adding a custom prefix or suffix:

```js
---
name: analyzerAqlConcat
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("concat", "aql", { queryString:
  "RETURN LOWER(LEFT(@param, 5)) == 'inter' ? CONCAT(@param, 'ism') : CONCAT('inter', @param)"
}, []);
db._query("RETURN TOKENS('state', 'concat')").toArray();
db._query("RETURN TOKENS('international', 'concat')").toArray();
~analyzers.remove(a.name);
```

Filtering Analyzer that ignores unwanted data based on the prefix `"ir"`,
with `keepNull: false` and explicitly returning `null`:

```js
---
name: analyzerAqlFilterNull
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("filter", "aql", { keepNull: false, queryString:
  "RETURN LOWER(LEFT(@param, 2)) == 'ir' ? null : @param"
}, []);
db._query("RETURN TOKENS('regular', 'filter')").toArray();
db._query("RETURN TOKENS('irregular', 'filter')").toArray();
~analyzers.remove(a.name);
```

Filtering Analyzer that discards unwanted data based on the prefix `"ir"`,
using a filter for an empty result, which is discarded from the View index even
without `keepNull: false`:

```js
---
name: analyzerAqlFilter
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("filter", "aql", { queryString:
  "FILTER LOWER(LEFT(@param, 2)) != 'ir' RETURN @param"
}, []);
var coll = db._create("coll");
var doc1 = db.coll.save({ value: "regular" });
var doc2 = db.coll.save({ value: "irregular" });
var view = db._createView("view", "arangosearch",
  { links: { coll: { fields: { value: { analyzers: ["filter"] }}}}})
~assert(db._query(`FOR d IN view COLLECT WITH COUNT INTO c RETURN c`).toArray()[0] > 0);
db._query("FOR doc IN view SEARCH ANALYZER(doc.value IN ['regular', 'irregular'], 'filter') RETURN doc").toArray();
~db._dropView(view.name())
~analyzers.remove(a.name);
~db._drop(coll.name());
```

Custom tokenization with `collapsePositions` on and off:
The input string `"A-B-C-D"` is split into an array of strings
`["A", "B", "C", "D"]`. The position metadata (as used by the `PHRASE()`
function) is set to 0 for all four strings if `collapsePositions` is enabled.
Otherwise the position is set to the respective array index, 0 for `"A"`,
1 for `"B"` and so on.

| `collapsePositions` | A | B | C | D |
|--------------------:|:-:|:-:|:-:|:-:|
|              `true` | 0 | 0 | 0 | 0 |
|             `false` | 0 | 1 | 2 | 3 |

```js
---
name: analyzerAqlCollapse
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a1 = analyzers.save("collapsed", "aql", { collapsePositions: true, queryString:
  "FOR d IN SPLIT(@param, '-') RETURN d"
}, ["frequency", "position"]);
var a2 = analyzers.save("uncollapsed", "aql", { collapsePositions: false, queryString:
  "FOR d IN SPLIT(@param, '-') RETURN d"
}, ["frequency", "position"]);
var coll = db._create("coll");
var doc = db.coll.save({ text: "A-B-C-D" });
var view = db._createView("view", "arangosearch",
  { links: { coll: { analyzers: [ "collapsed", "uncollapsed" ], includeAllFields: true }}});
~assert(db._query(`FOR d IN view COLLECT WITH COUNT INTO c RETURN c`).toArray()[0] === 1);
db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, 1, {TERM: 'D'}, 'uncollapsed') RETURN d");
db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, -1, {TERM: 'D'}, 'uncollapsed') RETURN d");
db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, 1, {TERM: 'D'}, 'collapsed') RETURN d");
db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, -1, {TERM: 'D'}, 'collapsed') RETURN d");
~db._dropView(view.name());
~analyzers.remove(a1.name);
~analyzers.remove(a2.name);
~db._drop(coll.name());
```

The position data is not directly exposed, but we can see its effects through
the `PHRASE()` function. There is one token between `"B"` and `"D"` to skip in
case of uncollapsed positions. With positions collapsed, both are in the same
position, thus there is negative one to skip to match the tokens.

### `pipeline`

<small>Introduced in: v3.8.0</small>

An Analyzer capable of chaining effects of multiple Analyzers into one.
The pipeline is a list of Analyzers, where the output of an Analyzer is passed
to the next for further processing. The final token value is determined by last
Analyzer in the pipeline.

The Analyzer is designed for cases like the following:
- Normalize text for a case insensitive search and apply _n_-gram tokenization
- Split input with `delimiter` Analyzer, followed by stemming with the `stem`
  Analyzer

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `pipeline` (array): an array of Analyzer definition-like objects with
  `type` and `properties` attributes
  
{{< info >}}
- You cannot use Analyzers of the types `geopoint`, `geojson`, and `geo_s2` in pipelines.
  These Analyzers require additional postprocessing and can only be applied to
  document fields directly.
- The output data type of an Analyzer needs to be compatible with the input
  data type of the next Analyzer in the chain. The `aql` Analyzer, in particular,
  has a `returnType` property, and if you set it to `number` or `bool`, the
  subsequent Analyzer in the pipeline needs to support this data type as input.
  Most Analyzers expect string inputs and are thus incompatible with such a setup.
{{< /info >}}

**Examples**

Normalize to all uppercase and compute bigrams:

```js
---
name: analyzerPipelineUpperNgram
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("ngram_upper", "pipeline", { pipeline: [
  { type: "norm", properties: { locale: "en", case: "upper" } },
  { type: "ngram", properties: { min: 2, max: 2, preserveOriginal: false, streamType: "utf8" } }
] }, []);
db._query(`RETURN TOKENS("Quick brown foX", "ngram_upper")`).toArray();
~analyzers.remove(a.name);
```

Split at delimiting character `,`, then stem the tokens:

```js
---
name: analyzerPipelineDelimiterStem
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("delimiter_stem", "pipeline", { pipeline: [
  { type: "delimiter", properties: { delimiter: "," } },
  { type: "stem", properties: { locale: "en" } }
] }, []);
db._query(`RETURN TOKENS("delimited,stemmable,words", "delimiter_stem")`).toArray();
~analyzers.remove(a.name);
```

### `stopwords`

<small>Introduced in: v3.8.1</small>

An Analyzer capable of removing specified tokens from the input.

It uses binary comparison to determine if an input token should be discarded.
It checks for exact matches. If the input contains only a substring that
matches one of the defined stop words, then it is not discarded. Longer inputs
such as prefixes of stop words are also not discarded.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `stopwords` (array): array of strings that describe the tokens to
  be discarded. The interpretation of each string depends on the value of
  the `hex` parameter.
- `hex` (boolean): If false (default), then each string in `stopwords` is used
  verbatim. If true, then the strings need to be hex-encoded. This allows for
  removing tokens that contain non-printable characters. To encode UTF-8
  strings to hex strings you can use e.g.
  - AQL:
    ```aql
    FOR token IN ["and","the"] RETURN TO_HEX(token)
    ```
  - arangosh / Node.js:
    ```js
    ["and","the"].map(token => Buffer(token).toString("hex"))
    ```
  - Modern browser:
    ```js
    ["and","the"].map(token => Array.from(new TextEncoder().encode(token), byte => byte.toString(16).padStart(2, "0")).join(""))
    ```

**Examples**

Create and use a `stopword` Analyzer that removes the tokens `and` and `the`.
The stop word array with hex-encoded strings for this looks like
`["616e64","746865"]` (`a` = 0x61, `n` = 0x6e, `d` = 0x64 and so on).
Note that `a` and `theater` are not removed, because there is no exact match
with either of the stop words `and` and `the`:

```js
---
name: analyzerStopwords
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("stop", "stopwords", {
  stopwords: ["616e64","746865"], hex: true
}, []);
db._query("RETURN FLATTEN(TOKENS(SPLIT('the fox and the dog and a theater', ' '), 'stop'))").toArray();
~analyzers.remove(a.name);
```

Create and use an Analyzer pipeline that normalizes the input (convert to
lower-case and base characters) and then discards the stopwords `and` and `the`:

```js
---
name: analyzerPipelineStopwords
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("norm_stop", "pipeline", { "pipeline": [
  { type: "norm", properties: { locale: "en", accent: false, case: "lower" } },
  { type: "stopwords", properties: { stopwords: ["and","the"], hex: false } },
]}, []);
db._query("RETURN FLATTEN(TOKENS(SPLIT('The fox AND the dog äñḏ a ţhéäter', ' '), 'norm_stop'))").toArray();
~analyzers.remove(a.name);
```

### `segmentation`

<small>Introduced in: v3.9.0</small>

An Analyzer capable of breaking up the input text into tokens in a
language-agnostic manner as per
[Unicode Standard Annex #29](https://unicode.org/reports/tr29),
making it suitable for mixed language strings. It can optionally preserve all
non-whitespace or all characters instead of keeping alphanumeric characters only,
as well as apply case conversion.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `break` (string, _optional_):
  - `"all"`: return all tokens
  - `"alpha"`: return tokens composed of alphanumeric characters only (default).
    Alphanumeric characters are Unicode codepoints from the Letter and Number
    categories, see
    [Unicode Technical Note #36](https://www.unicode.org/notes/tn36/).
  - `"graphic"`: return tokens composed of non-whitespace characters only.
    Note that the list of whitespace characters does not include line breaks:
    - `U+0009` Character Tabulation
    - `U+0020` Space
    - `U+0085` Next Line
    - `U+00A0` No-break Space
    - `U+1680` Ogham Space Mark
    - `U+2000` En Quad
    - `U+2028` Line Separator
    - `U+202F` Narrow No-break Space
    - `U+205F` Medium Mathematical Space
    - `U+3000` Ideographic Space
- `case` (string, _optional_):
  - `"lower"` to convert to all lower-case characters (default)
  - `"upper"` to convert to all upper-case characters
  - `"none"` to not change character case

**Examples**

Create different `segmentation` Analyzers to show the behavior of the different
`break` options:

```js
---
name: analyzerSegmentationBreak
description: ''
---
var analyzers = require("@arangodb/analyzers");
var all = analyzers.save("segment_all", "segmentation", { break: "all" }, []);
var alpha = analyzers.save("segment_alpha", "segmentation", { break: "alpha" }, []);
var graphic = analyzers.save("segment_graphic", "segmentation", { break: "graphic" }, []);
db._query(`LET str = 'Test\twith An_EMAIL-address+123@example.org\n蝴蝶。\u2028бутерброд'
  RETURN {
    "all": TOKENS(str, 'segment_all'),
    "alpha": TOKENS(str, 'segment_alpha'),
    "graphic": TOKENS(str, 'segment_graphic'),
  }
`).toArray();
~analyzers.remove(all.name);
~analyzers.remove(alpha.name);
~analyzers.remove(graphic.name);
```

### `wildcard`

<small>Introduced in: v3.12.0</small>

An Analyzer that creates _n_-grams to enable fast partial matching for wildcard
queries if you have large string values, especially if you want to search for
suffixes or substrings in the middle of strings (infixes) as opposed to prefixes.

It can apply an Analyzer of your choice before creating the _n_-grams, for example,
to normalize text for case-insensitive and accent-insensitive search.

See [Wildcard Search with ArangoSearch](arangosearch/wildcard-search.md#wildcard-analyzer-examples)
for an example of how to use this Analyzer with Views, and
[Inverted indexes](indexing/working-with-indexes/inverted-indexes.md#wildcard-search)
for an example using a standalone inverted index.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `ngramSize` (number, _required_): unsigned integer for the _n_-gram length,
  needs to be at least `2`. It can be greater than the substrings between
  wildcards that you want to search for, e.g. `4` with an expected search pattern
  of `%up%if%ref%` (substrings of length 2 and 3 between `%`), but this leads to
  a slower search (for `ref%` with post-validation using the ICU regular expression
  engine). A value of `3` is a good default, `2` is better for short strings
- `analyzer` (object, _optional_): an Analyzer definition-like objects with
  `type` and `properties` attributes

You cannot set the `offset` [Analyzer features](#analyzer-features) for this
Analyzer. You can create the Analyzer without features, or use
`["frequency", "position"]` which uses more memory but doesn't need to read
stored column values for prefix, suffix, infix, and exact queries (`something%`,
`%something`, `%something%`, `something`, but not `some%thing` or `%some%thin_`)

The set of _n_-grams that are created for each input token includes one start
_n_-gram and `ngramSize - 1` end _n_-grams using `"\uFFFD"` as start marker
respectively end marker. For example, with an `ngramSize` of `3` and the input
`"graph"`, the generated _n_-grams are:
- `"�gr"`
- `"gra"`
- `"rap"`
- `"aph"`
- `"ph�"`
- `"h�"`

**Examples**

```js
---
name: analyzerWildcard1
description: |
  Create a `wildcard` Analyzer with an _n_-gram size of 4 and without enabling
  Analyzer features:
---
var analyzers = require("@arangodb/analyzers");
var analyzerWildcard = analyzers.save("wildcard_4", "wildcard", { ngramSize: 4 }, []);

db._query(`RETURN TOKENS("The quick brown Foxx", "wildcard_4")`).toArray();
~analyzers.remove(analyzerWildcard.name);
```

```js
---
name: analyzerWildcard2
description: |
  Use a `norm` Analyzer in a `wildcard` Analyzer to normalize the input to
  lowercase and convert accented characters to their base characters before
  creating trigrams from the string:
---
var analyzers = require("@arangodb/analyzers");
var analyzerWildcard = analyzers.save("wildcard_3", "wildcard", { ngramSize: 3, "analyzer": {
  type: "norm", properties: { locale: "en", accent: false, case: "lower" }
} }, ["frequency","position"]);

db._query(`RETURN TOKENS("The quick brown Foxx", "wildcard_3")`).toArray();
~analyzers.remove(analyzerWildcard.name);
```

### `minhash`

<small>Introduced in: v3.10.0</small>

An Analyzer that computes so called MinHash signatures using a
locality-sensitive hash function. It applies an Analyzer of your choice before
the hashing, for example, to break up text into words.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `analyzer` (object, _required_): an Analyzer definition-like objects with
  `type` and `properties` attributes
- `numHashes` (number, _required_): the size of the MinHash signature. Must be
  greater or equal to `1`. The signature size defines the probabilistic error
  (`err = rsqrt(numHashes)`). For an error amount that does not exceed 5%
  (`0.05`), use a size of `1 / (0.05 * 0.05) = 400`.

**Examples**

Create a `minhash` Analyzers:

```js
---
name: analyzerMinHash
description: ''
---
var analyzers = require("@arangodb/analyzers");
var analyzerMinHash = analyzers.save("minhash5", "minhash", { analyzer: { type: "segmentation", properties: { break: "alpha", case: "lower" } }, numHashes: 5 }, []);
var analyzerSegment = analyzers.save("segment", "segmentation", { break: "alpha", case: "lower" }, []);
db._query(`
  LET str1 = "The quick brown fox jumps over the lazy dog."
  LET str2 = "The fox jumps over the crazy dog."
  RETURN {
    approx: JACCARD(TOKENS(str1, "minhash5"), TOKENS(str2, "minhash5")),
    actual: JACCARD(TOKENS(str1, "segment"), TOKENS(str2, "segment"))
  }`).toArray();
~analyzers.remove(analyzerMinHash.name);
~analyzers.remove(analyzerSegment.name);
```

### `classification`

<small>Introduced in: v3.10.0</small>

{{< warning >}}
This feature is experimental and under active development.
The naming and interfaces may change at any time.
Execution times are not representative of the final product.
{{< /warning >}}

An Analyzer capable of classifying tokens in the input text.

It applies a user-provided [supervised fastText](https://fasttext.cc/docs/en/supervised-tutorial.html)
word embedding model to classify the input text. It is able to classify
individual tokens as well as entire inputs.

The *properties* allowed for this Analyzer are an object with the following attributes:

- `model_location` (string): the on-disk path to the trained fastText supervised model.
  Note: if you are running this in an ArangoDB cluster, this model must exist on
  every machine in the cluster.
- `top_k` (number, optional): the number of class labels that will be produced
  per input (default: 1).
- `threshold` (number, optional): the probability threshold for which a label
  will be assigned to an input. A fastText model produces a probability per
  class label, and this is what will be filtered (default: `0.99`).

**Examples**

Create and use a `classification` Analyzer with a stored "cooking" classifier
to classify items.

```js
var analyzers = require("@arangodb/analyzers");
var classifier_single = analyzers.save("classifier_single", "classification", { "model_location": "/path_to_local_fasttext_model_directory/model_cooking.bin" }, []);
var classifier_top_two = analyzers.save("classifier_double", "classification", { "model_location": "/path_to_local_fasttext_model_directory/model_cooking.bin", "top_k": 2 }, []);
db._query(`LET str = "Which baking dish is best to bake a banana bread ?"
    RETURN {
      "all": TOKENS(str, "classifier_single"),
      "double": TOKENS(str, "classifier_double")
    }
  `);
```

```json
[
  {
    "all" : [
      "__label__baking"
    ],
    "double" : [
      "__label__baking",
      "__label__bananas"
    ]
  }
]
```

### `nearest_neighbors`

<small>Introduced in: v3.10.0</small>

{{< warning >}}
This feature is experimental and under active development.
The naming and interfaces may change at any time.
Execution times are not representative of the final product.
{{< /warning >}}

An Analyzer capable of finding nearest neighbors of tokens in the input.

It applies a user-provided [supervised fastText](https://fasttext.cc/docs/en/supervised-tutorial.html)
word embedding model to retrieve nearest neighbor tokens in the text.
It is able to find neighbors of individual tokens as well as entire input strings.
For entire input strings, the Analyzer will return nearest neighbors for each
token within the input string.

The *properties* allowed for this Analyzer are an object with the following attributes:

- `model_location` (string): the on-disk path to the trained fastText supervised model.
  Note: if you are running this in an ArangoDB cluster, this model must exist on
  every machine in the cluster.
- `top_k` (number, optional): the number of class labels that will be produced
  per input (default: `1`).

**Examples**

Create and use a `nearest_neighbors` Analyzer with a stored "cooking" classifier
to find similar terms.

```js
var analyzers = require("@arangodb/analyzers");
var nn_single = analyzers.save("nn_single", "nearest_neighbors", { "model_location": "/path_to_local_fasttext_model_directory/model_cooking.bin" }, []);
var nn_top_two = analyzers.save("nn_double", "nearest_neighbors", { "model_location": "/path_to_local_fasttext_model_directory/model_cooking.bin", "top_k": 2 }, []);
db._query(`LET str = "salt, oil"
    RETURN {
      "all": TOKENS(str, "nn_single"),
      "double": TOKENS(str, "nn_double")
    }
  `);
```

```json
[
  {
    "all" : [
      "pepper",
      "olive"
    ],
    "double" : [
      "pepper",
      "table",
      "olive",
      "avocado"
    ]
  }
]
```

### `geojson`

<small>Introduced in: v3.8.0</small>

An Analyzer capable of breaking up a GeoJSON object or coordinate array in
`[longitude, latitude]` order into a set of indexable tokens for further usage
with [ArangoSearch Geo functions](../aql/functions/arangosearch.md#geo-functions).

The Analyzer can be used for two different coordinate representations:

- a GeoJSON feature like a Point or Polygon, using a JSON object like the following:

  ```js
  {
    "type": "Point",
    "coordinates": [ -73.97, 40.78 ] // [ longitude, latitude ]
  }
  ```

- a coordinate array with two numbers as elements in the following format:

  ```js
  [ -73.97, 40.78 ] // [ longitude, latitude ]
  ```

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `type` (string, _optional_):
  - `"shape"` (default): index all GeoJSON geometry types (Point, Polygon etc.)
  - `"centroid"`: compute and only index the centroid of the input geometry
  - `"point"`: only index GeoJSON objects of type Point, ignore all other
    geometry types
- `options` (object, _optional_): options for fine-tuning geo queries.
  These options should generally remain unchanged
  - `maxCells` (number, _optional_): maximum number of S2 cells (default: 20)
  - `minLevel` (number, _optional_): the least precise S2 level (default: 4)
  - `maxLevel` (number, _optional_): the most precise S2 level (default: 23)
- `legacy` (boolean, _optional_):
  This option controls how GeoJSON Polygons are interpreted (introduced in v3.10.5).
  Also see [Legacy Polygons](indexing/working-with-indexes/geo-spatial-indexes.md#legacy-polygons) and
  [GeoJSON interpretation](../aql/functions/geo.md#geojson-interpretation).

  - If `legacy` is `true`, the smaller of the two regions defined by a
    linear ring is interpreted as the interior of the ring and a ring can at most
    enclose half the Earth's surface.
  - If `legacy` is `false`, the area to the left of the boundary ring's
    path is considered to be the interior and a ring can enclose the entire
    surface of the Earth.

  The default is `false`.

  {{< warning >}}
  If you use `geojson` Analyzers and upgrade from a version below 3.10 to a
  version of 3.10 or higher, the interpretation of GeoJSON Polygons changes.

  If you have polygons in your data that mean to refer to a relatively small
  region but have the boundary running clockwise around the intended interior,
  they are interpreted as intended prior to 3.10, but from 3.10 onward, they are
  interpreted as "the other side" of the boundary.

  Whether a clockwise boundary specifies the complement of the small region
  intentionally or not cannot be determined automatically. Please test the new
  behavior manually.

  If you require the old behavior, upgrade to at least 3.10.5, drop your
  `geojson` Analyzers, and create new ones with `legacy` set to `true`.
  If these Analyzers are used in `arangosearch` Views, then they need to be
  dropped as well before dropping the Analyzers, and recreated after creating
  the new Analyzers.
  {{< /warning >}}

You should not set any of the [Analyzer features](#analyzer-features) as they
cannot be utilized for Geo Analyzers.

**Examples**

Create a collection with GeoJSON Points stored in an attribute `location`, a
`geojson` Analyzer with default properties, and a View using the Analyzer.
Then query for locations that are within a 3 kilometer radius of a given point
and return the matched documents, including the calculated distance in meters.
The stored coordinate pairs and the `GEO_POINT()` arguments are expected in
longitude, latitude order:

```js
---
name: analyzerGeoJSON
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("geo_json", "geojson", {}, []);
var coll = db._create("geo");
var docs = db.geo.save([
  { location: { type: "Point", coordinates: [6.937, 50.932] } },
  { location: { type: "Point", coordinates: [6.956, 50.941] } },
  { location: { type: "Point", coordinates: [6.962, 50.932] } },
]);
var view = db._createView("geo_view", "arangosearch", {
  links: {
    geo: {
      fields: {
        location: {
          analyzers: ["geo_json"]
        }
      }
    }
  }
});
~assert(db._query(`FOR d IN geo_view COLLECT WITH COUNT INTO c RETURN c`).toArray()[0] === 3);
db._query(`LET point = GEO_POINT(6.93, 50.94)
  FOR doc IN geo_view
    SEARCH ANALYZER(GEO_DISTANCE(doc.location, point) < 2000, "geo_json")
    RETURN MERGE(doc, { distance: GEO_DISTANCE(doc.location, point) })`).toArray();
~db._dropView("geo_view");
~analyzers.remove("geo_json", true);
~db._drop("geo");
```

### `geo_s2`

<small>Introduced in: v3.10.5</small>

An Analyzer capable of breaking up a GeoJSON object or coordinate array in
`[longitude, latitude]` order into a set of indexable tokens for further usage
with [ArangoSearch Geo functions](../aql/functions/arangosearch.md#geo-functions).

The Analyzer is similar to the `geojson` Analyzer, but it internally uses a
format for storing the geo-spatial data that is more efficient. You can choose
between different formats to make a tradeoff between the size on disk, the
precision, and query performance.

The Analyzer can be used for two different coordinate representations:

- a GeoJSON feature like a Point or Polygon, using a JSON object like the following:

  ```js
  {
    "type": "Point",
    "coordinates": [ -73.97, 40.78 ] // [ longitude, latitude ]
  }
  ```

- a coordinate array with two numbers as elements in the following format:

  ```js
  [ -73.97, 40.78 ] // [ longitude, latitude ]
  ```

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `format` (string, _optional_): the internal binary representation to use for
  storing the geo-spatial data in an index
  - `"latLngDouble"` (default): store each latitude and longitude value as an
    8-byte floating-point value (16 bytes per coordinate pair). This format preserves
    numeric values exactly and is more compact than the VelocyPack format used
    by the `geojson` Analyzer.
  - `"latLngInt"`: store each latitude and longitude value as an 4-byte integer
    value (8 bytes per coordinate pair). This is the most compact format but the
    precision is limited to approximately 1 to 10 centimeters.
  - `"s2Point"`: store each longitude-latitude pair in the native format of
    Google S2 which is used for geo-spatial calculations (24 bytes per coordinate pair).
    This is not a particular compact format but it reduces the number of
    computations necessary when you execute geo-spatial queries.
    This format preserves numeric values exactly.
- `type` (string, _optional_):
  - `"shape"` (default): index all GeoJSON geometry types (Point, Polygon etc.)
  - `"centroid"`: compute and only index the centroid of the input geometry
  - `"point"`: only index GeoJSON objects of type Point, ignore all other
    geometry types
- `options` (object, _optional_): options for fine-tuning geo queries.
  These options should generally remain unchanged
  - `maxCells` (number, _optional_): maximum number of S2 cells (default: 20)
  - `minLevel` (number, _optional_): the least precise S2 level (default: 4)
  - `maxLevel` (number, _optional_): the most precise S2 level (default: 23)

You should not set any of the [Analyzer features](#analyzer-features) as they
cannot be utilized for Geo Analyzers.

**Examples**

Create a collection with GeoJSON Points stored in an attribute `location`, a
`geo_s2` Analyzer with the `latLngInt` format, and a View using the Analyzer.
Then query for locations that are within a 3 kilometer radius of a given point
and return the matched documents, including the calculated distance in meters.
The stored coordinate pairs and the `GEO_POINT()` arguments are expected in
longitude, latitude order:

```js
---
name: analyzerGeoS2
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("geo_efficient", "geo_s2", { format: "latLngInt" }, []);
var coll = db._create("geo");
var docs = db.geo.save([
  { location: { type: "Point", coordinates: [6.937, 50.932] } },
  { location: { type: "Point", coordinates: [6.956, 50.941] } },
  { location: { type: "Point", coordinates: [6.962, 50.932] } },
]);
var view = db._createView("geo_view", "arangosearch", {
  links: {
    geo: {
      fields: {
        location: {
          analyzers: ["geo_efficient"]
        }
      }
    }
  }
});
~assert(db._query(`FOR d IN geo_view COLLECT WITH COUNT INTO c RETURN c`).toArray()[0] === 3);
db._query(`LET point = GEO_POINT(6.93, 50.94)
  FOR doc IN geo_view
    SEARCH ANALYZER(GEO_DISTANCE(doc.location, point) < 2000, "geo_efficient")
    RETURN MERGE(doc, { distance: GEO_DISTANCE(doc.location, point) })`).toArray();
~db._dropView("geo_view");
~analyzers.remove("geo_efficient", true);
~db._drop("geo");
```

The calculated distance between the reference point and the point stored in
the second document is `1825.1307…`. If you change the search condition to
`< 1825.1303`, then the document is still returned despite the distance being
higher than this value. This is due to the precision limitations of the
`latLngInt` format. The returned distance is unaffected because it is calculated
independent of the Analyzer. If you use either of the other two formats which
preserve the exact coordinate values, then the document is filtered out as
expected.

### `geopoint`

<small>Introduced in: v3.8.0</small>

An Analyzer capable of breaking up a coordinate array in `[latitude, longitude]`
order or a JSON object describing a coordinate pair using two separate attributes
into a set of indexable tokens for further usage with
[ArangoSearch Geo functions](../aql/functions/arangosearch.md#geo-functions).

The Analyzer can be used for two different coordinate representations:

- an array with two numbers as elements in the following format:

  ```js
  [ 40.78, -73.97 ] // [ latitude, longitude ]
  ```

- two separate numeric attributes, one for latitude and one for longitude, as
  shown below:

  ```js
  { "location": { "lat": 40.78, "lon": -73.97 } }
  ```

  The attributes cannot be at the top level of the document, but must be nested
  like in the example, so that the Analyzer can be defined for the field
  `location` with the Analyzer properties
  `{ "latitude": ["lat"], "longitude": ["lon"] }`.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `latitude` (array, _optional_): array of strings that describes the
  attribute path of the latitude value relative to the field for which the
  Analyzer is defined in the View
- `longitude` (array, _optional_): array of strings that describes the
  attribute path of the longitude value relative to the field for which the
  Analyzer is defined in the View
- `options` (object, _optional_): options for fine-tuning geo queries.
  These options should generally remain unchanged
  - `maxCells` (number, _optional_): maximum number of S2 cells (default: 20)
  - `minLevel` (number, _optional_): the least precise S2 level (default: 4)
  - `maxLevel` (number, _optional_): the most precise S2 level (default: 23)

You should not set any of the [Analyzer features](#analyzer-features) as they
cannot be utilized for Geo Analyzers.

**Examples**

Create a collection with coordinate pairs stored in an attribute `location`,
a `geopoint` Analyzer with default properties, and a View using the Analyzer.
Then query for locations that are within a 3 kilometer radius of a given point.
The stored coordinate pairs are in latitude, longitude order, but `GEO_POINT()` and
`GEO_DISTANCE()` expect longitude, latitude order:

```js
---
name: analyzerGeoPointPair
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("geo_pair", "geopoint", {}, []);
var coll = db._create("geo");
var docs = db.geo.save([
  { location: [50.932, 6.937] },
  { location: [50.941, 6.956] },
  { location: [50.932, 6.962] },
]);
var view = db._createView("geo_view", "arangosearch", {
  links: {
    geo: {
      fields: {
        location: {
          analyzers: ["geo_pair"]
        }
      }
    }
  }
});
~assert(db._query(`FOR d IN geo_view COLLECT WITH COUNT INTO c RETURN c`).toArray()[0] === 3);
db._query(`LET point = GEO_POINT(6.93, 50.94)
  FOR doc IN geo_view
    SEARCH ANALYZER(GEO_DISTANCE(doc.location, point) < 2000, "geo_pair")
    RETURN MERGE(doc, { distance: GEO_DISTANCE([doc.location[1], doc.location[0]], point) })`).toArray();
~db._dropView("geo_view");
~analyzers.remove("geo_pair", true);
~db._drop("geo");
```

Create a collection with coordinate pairs stored in an attribute `location` as
separate nested attributes `lat` and `lng`, a `geopoint` Analyzer that
specifies the attribute paths to the latitude and longitude attributes
(relative to `location` attribute), and a View using the Analyzer.
Then query for locations that are within a 3 kilometer radius of a given point:

```js
---
name: analyzerGeoPointLatLng
description: ''
---
var analyzers = require("@arangodb/analyzers");
var a = analyzers.save("geo_latlng", "geopoint", {
  latitude: ["lat"],
  longitude: ["lng"]
}, []);
var coll = db._create("geo");
var docs = db.geo.save([
  { location: { lat: 50.932, lng: 6.937 } },
  { location: { lat: 50.941, lng: 6.956 } },
  { location: { lat: 50.932, lng: 6.962 } },
]);
var view = db._createView("geo_view", "arangosearch", {
  links: {
    geo: {
      fields: {
        location: {
          analyzers: ["geo_latlng"]
        }
      }
    }
  }
});
~assert(db._query(`FOR d IN geo_view COLLECT WITH COUNT INTO c RETURN c`).toArray()[0] === 3);
db._query(`LET point = GEO_POINT(6.93, 50.94)
  FOR doc IN geo_view
    SEARCH ANALYZER(GEO_DISTANCE(doc.location, point) < 2000, "geo_latlng")
    RETURN MERGE(doc, { distance: GEO_DISTANCE([doc.location.lng, doc.location.lat], point) })`).toArray();
~db._dropView("geo_view");
~analyzers.remove("geo_latlng", true);
~db._drop("geo");
```

## Built-in Analyzers

There is a set of built-in Analyzers which are available by default for
convenience and backward compatibility. They cannot be removed.

The `identity` Analyzer has no properties and the `frequency` and `norm`
features. The Analyzers of type `text` all tokenize strings with stemming
enabled, no stop words configured, accent removal and case conversion to
lowercase turned on and the `frequency`, `norm` and `position` features

Name       | Type       | Locale (Language) | Case    | Accent  | Stemming | Stop words | Features |
-----------|------------|-------------------|---------|---------|----------|------------|----------|
`identity` | `identity` |                   |         |         |          |            | `["frequency", "norm"]`
`text_de`  | `text`     | `de` (German)     | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_en`  | `text`     | `en` (English)    | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_es`  | `text`     | `es` (Spanish)    | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_fi`  | `text`     | `fi` (Finnish)    | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_fr`  | `text`     | `fr` (French)     | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_it`  | `text`     | `it` (Italian)    | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_nl`  | `text`     | `nl` (Dutch)      | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_no`  | `text`     | `no` (Norwegian)  | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_pt`  | `text`     | `pt` (Portuguese) | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_ru`  | `text`     | `ru` (Russian)    | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_sv`  | `text`     | `sv` (Swedish)    | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`
`text_zh`  | `text`     | `zh` (Chinese)    | `lower` | `false` | `true`   | `[ ]`      | `["frequency", "norm", "position"]`

Note that _locale_, _case_, _accent_, _stemming_ and _stopwords_ are Analyzer
properties. `text_zh` does not have actual stemming support for Chinese despite
what the property value suggests.

## Supported Languages

### Tokenization and Normalization

Analyzers rely on [ICU](http://site.icu-project.org/) for
tokenization and normalization, which is language-dependent.
The ICU data file `icudtl.dat` that ArangoDB ships with contains information for
a lot of languages, which are technically all supported.

Setting an unsupported or invalid locale does not raise a warning or error.
ICU will fall back to a locale without the requested variant, country, or
script, or use its default locale if neither of the former is valid.

{{< warning >}}
The alphabetical order of characters is not taken into account by ArangoSearch,
i.e. range queries in SEARCH operations against Views will not follow the
language rules as per the defined Analyzer locale (except for the
[`collation` Analyzer](#collation)) nor the server language
(startup option `--default-language`)!
Also see [Known Issues](../release-notes/version-3.12/known-issues-in-3-12.md#arangosearch).
{{< /warning >}}

### Stemming

Stemming support is provided by [Snowball](https://snowballstem.org/),
which supports the following languages:

Language    | Code
------------|-----
Arabic      | `ar`
Armenian  * | `hy`
Basque      | `eu`
Catalan     | `ca`
Danish      | `da`
Dutch       | `nl`
English     | `en`
Finnish     | `fi`
French      | `fr`
German      | `de`
Greek       | `el`
Hindi       | `hi`
Hungarian   | `hu`
Indonesian  | `id`
Irish       | `ga`
Italian     | `it`
Lithuanian  | `lt`
Nepali      | `ne`
Norwegian   | `no`
Portuguese  | `pt`
Romanian    | `ro`
Russian     | `ru`
Serbian     | `sr`
Spanish     | `es`
Swedish     | `sv`
Tamil       | `ta`
Turkish     | `tr`
Yiddish   * | `yi`

\* <small>Introduced in: v3.10.0</small>
