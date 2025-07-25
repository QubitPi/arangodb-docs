---
title: '`SEARCH` operation in AQL'
menuTitle: SEARCH
weight: 20
description: >-
  The `SEARCH` operation lets you filter Views, accelerated by the underlying
  indexes
---
The `SEARCH` operation guarantees to use View indexes for an efficient
execution plan. If you use the `FILTER` keyword for Views, no indexes are
utilized and the filtering is performed as a post-processing step.

Conceptually, a View is just another document data source,
similar to an array or a document/edge collection, over which you can iterate
using a [`FOR` operation](for.md) in AQL:

```aql
FOR doc IN viewName
  RETURN doc
```

The optional `SEARCH` operation provides the capabilities to:

- filter documents based on AQL Boolean expressions and functions
- match documents located in different collections backed by a fast index
- sort the result set based on how closely each document matched the
  search conditions

See [`arangosearch` Views](../../index-and-search/arangosearch/arangosearch-views-reference.md) and
[`search-alias` Views](../../index-and-search/arangosearch/search-alias-views-reference.md) on how to set up Views.

## Syntax

The `SEARCH` keyword is followed by an ArangoSearch filter expressions, which
is mostly comprised of calls to ArangoSearch AQL functions.

<pre><code>FOR <em>doc</em> IN <em>viewName</em>
  SEARCH <em>expression</em>
  OPTIONS { … }
  ...</code></pre>

## Usage

The `SEARCH` statement, in contrast to `FILTER`, is treated as a part of the
`FOR` operation, not as an individual statement. It cannot be placed freely
in a query nor multiple times in the body of a `FOR` loop. `FOR ... IN` must be
followed by the name of a View, not a collection. The `SEARCH` operation has to
follow next, other operations before `SEARCH` such as `FILTER`, `COLLECT` etc.
are not allowed in this position. Subsequent operations are possible after
`SEARCH` and the expression however, including `SORT` to order the search
results based on a ranking value computed by the View.

*expression* must be an ArangoSearch expression. The full power of ArangoSearch
is harnessed and exposed via special [ArangoSearch functions](../functions/arangosearch.md),
during both the search and sort stages. On top of that, common AQL operators
are supported.

Note that inline expressions and a few other things are not supported by
`SEARCH`. The server will raise a query error in case of an invalid expression.

The `OPTIONS` keyword and an object can optionally follow the search expression
to set [Search Options](#search-options).

### Logical operators

Logical or Boolean operators allow you to combine multiple search conditions.

- `AND`, `&&` (conjunction)
- `OR`, `||` (disjunction)
- `NOT`, `!` (negation / inversion)

[Operator precedence](../operators.md#operator-precedence) needs to be taken
into account and can be controlled with parentheses.

Consider the following contrived expression:

`doc.value < 0 OR doc.value > 5 AND doc.value IN [-10, 10]`

`AND` has a higher precedence than `OR`. The expression is equivalent to:

`doc.value < 0 OR (doc.value > 5 AND doc.value IN [-10, 10])`

The conditions are thus:
- values less than 0
- values greater than 5, but only if it is 10
  (or -10, but this can never be fulfilled)

Parentheses can be used as follows to apply the `AND` condition to both of the
`OR` conditions:

`(doc.value < 0 OR doc.value > 5) AND doc.value IN [-10, 10]`

The conditions are now:
- values less than 0, but only if it is -10
- values greater than 5, but only if it is 10

### Comparison operators

- `==` (equal)
- `<=` (less than or equal)
- `>=` (greater than or equal)
- `<` (less than)
- `>` (greater than)
- `!=` (unequal)
- `IN` (contained in array or range), also `NOT IN`
- `LIKE` (equal with wildcards), also `NOT LIKE`

Also see the [`IN_RANGE()` function](../functions/arangosearch.md#in_range) for
an alternative to a combination of `<`, `<=`, `>`, `>=` operators for range
searches.

```aql
FOR doc IN viewName
  SEARCH ANALYZER(doc.text == "quick" OR doc.text == "brown", "text_en")
  // -- or --
  SEARCH ANALYZER(doc.text IN ["quick", "brown"], "text_en")
  RETURN doc
```

{{< warning >}}
The alphabetical order of characters is not taken into account by ArangoSearch,
i.e. range queries in SEARCH operations against Views will not follow the
language rules as per the defined Analyzer locale (except for the
[`collation` Analyzer](../../index-and-search/analyzers.md#collation)) nor the server language
(startup option `--default-language`)!
Also see [Known Issues](../../release-notes/version-3.12/known-issues-in-3-12.md#arangosearch).
{{< /warning >}}

### Array comparison operators

[Array comparison operators](../operators.md#array-comparison-operators) are
supported:

```aql
LET tokens = TOKENS("some input", "text_en")                 // ["some", "input"]
FOR doc IN myView SEARCH tokens  ALL IN doc.text RETURN doc // dynamic conjunction
FOR doc IN myView SEARCH tokens  ANY IN doc.text RETURN doc // dynamic disjunction
FOR doc IN myView SEARCH tokens NONE IN doc.text RETURN doc // dynamic negation
FOR doc IN myView SEARCH tokens  ALL >  doc.text RETURN doc // dynamic conjunction with comparison
FOR doc IN myView SEARCH tokens  ANY <= doc.text RETURN doc // dynamic disjunction with comparison
FOR doc IN myView SEARCH tokens NONE <  doc.text RETURN doc // dynamic negation with comparison
FOR doc IN myView SEARCH tokens AT LEAST (1+1) IN doc.text RETURN doc // dynamically test for a subset of elements
```

The following operators are equivalent in `SEARCH` expressions:
- `ALL IN`, `ALL ==`, `NONE !=`, `NONE NOT IN`
- `ANY IN`, `ANY ==`
- `NONE IN`, `NONE ==`, `ALL !=`, `ALL NOT IN`
- `ALL >`, `NONE <=`
- `ALL >=`, `NONE <`
- `ALL <`, `NONE >=`
- `ALL <=`, `NONE >`
- `AT LEAST (...) IN`, `AT LEAST (...) ==`
- `AT LEAST (1) IN`, `ANY IN`

The stored attribute referenced on the right side of the operator is like a
single, primitive value. In case of multiple tokens, it is like having multiple
such values as opposed to an array of values, even if the actual document
attribute is an array. `IN` and `==` as part of array comparison operators are
treated the same in `SEARCH` expressions for ease of use. The behavior is
different outside of `SEARCH`, where `IN` needs to be followed by an array.

### Question mark operator

You can use the [Question mark operator](../operators.md#question-mark-operator)
to perform [Nested searches with ArangoSearch](../../index-and-search/arangosearch/nested-search.md)
:

```aql
FOR doc IN myView
  SEARCH doc.dimensions[? FILTER CURRENT.type == "height" AND CURRENT.value > 40]
  RETURN doc
```

It allows you to match nested objects in arrays that satisfy multiple conditions
each, and optionally define how often these conditions should be fulfilled for
the entire array. You need to configure the View specifically for this type of
search using the `nested` property in [`arangosearch` Views](../../index-and-search/arangosearch/arangosearch-views-reference.md#link-properties)
or in the definition of [Inverted Indexes](../../index-and-search/indexing/working-with-indexes/inverted-indexes.md#nested-search)
that you can add to [`search-alias` Views](../../index-and-search/arangosearch/search-alias-views-reference.md).

## Handling of non-indexed fields

Document attributes which are not configured to be indexed by a View are
treated by `SEARCH` as non-existent. This affects tests against the documents
emitted from the View only.

For example, given a collection `myCol` with the following documents:

```js
{ "someAttr": "One", "anotherAttr": "One" }
{ "someAttr": "Two", "anotherAttr": "Two" }
```

… with an `arangosearch` View where `someAttr` is indexed by the following View `myView`:

```js
{
  "type": "arangosearch",
  "links": {
    "myCol": {
      "fields": {
        "someAttr": {}
      }
    }
  }
}
```

… a search on `someAttr` yields the following result:

```aql
FOR doc IN myView
  SEARCH doc.someAttr == "One"
  RETURN doc
```

```json
[ { "someAttr": "One", "anotherAttr": "One" } ]
```

A search on `anotherAttr` yields an empty result because only `someAttr`
is indexed by the View:

```aql
FOR doc IN myView
  SEARCH doc.anotherAttr == "One"
  RETURN doc
```

```json
[]
```

You can use the special `includeAllFields`
[`arangosearch` View property](../../index-and-search/arangosearch/arangosearch-views-reference.md#link-properties)
to index all (sub-)attributes of the source documents if desired.

## `SEARCH` with `SORT`

The documents emitted from a View can be sorted by attribute values with the
standard [`SORT()` operation](sort.md), using one or multiple
attributes, in ascending or descending order (or a mix thereof).

```aql
FOR doc IN viewName
  SORT doc.text, doc.value DESC
  RETURN doc
```

If the (left-most) fields and their sorting directions match up with the
[primary sort order](../../index-and-search/arangosearch/performance.md#primary-sort-order) definition
of the View then the `SORT` operation is optimized away.

Apart from simple sorting, it is possible to sort the matched View documents by
relevance score (or a combination of score and attribute values if desired).
The document search via the `SEARCH` keyword and the sorting via the
[ArangoSearch Scoring Functions](../functions/arangosearch.md#scoring-functions),
namely `BM25()` and `TFIDF()`, are closely intertwined.
The query given in the `SEARCH` expression is not only used to filter documents,
but also is used with the scoring functions to decide which document matches
the query best. Other documents in the View also affect this decision.

Therefore the ArangoSearch scoring functions can work _only_ on documents
emitted from a View, as both the corresponding `SEARCH` expression and the View
itself are consulted in order to sort the results.

```aql
FOR doc IN viewName
  SEARCH ...
  SORT BM25(doc) DESC
  RETURN doc
```

The [`BOOST()` function](../functions/arangosearch.md#boost) can be used to
fine-tune the resulting ranking by weighing sub-expressions in `SEARCH`
differently.

If there is no `SEARCH` operation prior to calls to scoring functions or if
the search expression does not filter out documents (e.g. `SEARCH true`) then
a score of `0` will be returned for all documents.

## Search Options

The `SEARCH` operation supports an optional `OPTIONS` clause to modify the
behavior. The general syntax is as follows:

<pre><code>SEARCH <em>expression</em> OPTIONS { <em>option</em>: <em>value</em>, <em>...</em> }</code></pre>

### `collections`

You can specify an array of strings with collection names to restrict the search
to certain source collections.

Given a View with three linked collections `coll1`, `coll2`, and `coll3`, you
can return documents from the first two collections only and ignore the third
collection by setting the `collections` option to `["coll1", "coll2"]`:

```aql
FOR doc IN viewName
  SEARCH true OPTIONS { collections: ["coll1", "coll2"] }
  RETURN doc
```

The search expression `true` in the above example matches all View documents.
You can use any valid expression here while limiting the scope to the chosen
source collections.

### `conditionOptimization`

You can specify one of the following values for this option to control how
search criteria get optimized:

- `"auto"` (default): convert conditions to disjunctive normal form (DNF) and
  apply optimizations. Removes redundant or overlapping conditions, but can
  take quite some time even for a low number of nested conditions.
- `"none"`: search the index without optimizing the conditions.
<!-- Internal only: nodnf, noneg -->

See [Optimizing View and inverted index query performance](../../index-and-search/arangosearch/performance.md#condition-optimization-options)
for an example.

### `countApproximate`

This option controls how the total count of rows is calculated if the `fullCount`
option is enabled for a query or when a `COLLECT WITH COUNT` clause is executed.
You can set it to one of the following values:

- `"exact"` (default): rows are actually enumerated for a precise count.
- `"cost"`: a cost-based approximation is used. Does not enumerate rows and
  returns an approximate result with O(1) complexity. Gives a precise result
  if the `SEARCH` condition is empty or if it contains a single term query
  only (e.g. `SEARCH doc.field == "value"`), the usual eventual consistency
  of Views aside.

See [Optimizing View and inverted index query performance](../../index-and-search/arangosearch/performance.md#count-approximation)
for an example.

### `parallelism`

A `SEARCH` operation can optionally process index segments in parallel using
multiple threads. This can speed up search queries but increases CPU and memory
utilization.

If you omit the `parallelism` option, then the default parallelism as defined by
the [`--arangosearch.default-parallelism` startup option](../../components/arangodb-server/options.md#--arangosearchdefault-parallelism)
is used. If you set it to a value of `1`, the search execution is not
parallelized. If the value is greater than `1`, then up to that many worker
threads can be used for concurrently processing index segments. The maximum
number of total parallel execution threads is defined by the
[`--arangosearch.execution-threads-limit` startup option](../../components/arangodb-server/options.md#--arangosearchexecution-threads-limit)
that defaults to twice the number of CPU cores.

{{< info >}}
Using too high parallelization can overload your hardware. It is recommended to
leave the default parallelism at `1` and set the `parallelism` option for queries
that highly benefit from the parallelization only. Use a moderate value in
accordance with the number of available CPU cores.
{{< /info >}}

The `parallelism` option should be considered a hint. Not all search queries are
eligible. Queries also don't wait for the specified number of threads to be
available. They start immediately even if only single-threaded and may acquire
more threads later.

```aql
FOR doc IN restaurantsView
  SEARCH ANALYZER(GEO_INTERSECTS(rect, doc.geometry), "geojson")
  OPTIONS { parallelism: 16 }
  RETURN doc.geometry
```
