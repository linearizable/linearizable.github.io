---
layout: post
title: "Solr SpellCheck Result Ordering Caveat"
author:
- Vikas Kumar
comments: true
---

Solr spellcheck component finds terms similar to a given query from the index within a configured edit distance. The spellcheck results are sorted first by edit distance (lower the better) and within the same edit distance, by frequency.

Suppose we have indexed the following documents:

|  ID | title |
| -------- | -------- | -------- | -------- |
| 1     | camri     |
| 2     | camri     |
| 3     | camri     |
| 4     | camri     |
| 5     | capri     |
| 6     | capri     |
| 7     | carol     |
| 8     | carol     |
| 9     | carol     |
| 10     | carol     |
| 11     | carol     |
| 12     | carol     |

Here's the spellcheck config:

```xml
<lst name="spellchecker">
  <str name="name">default</str>
  <str name="field">title</str>
  <str name="classname">solr.DirectSolrSpellChecker</str>
  <str name="distanceMeasure">internal</str>
  <float name="accuracy">0.5</float>
  <int name="maxEdits">2</int>
  <int name="minPrefix">1</int>
  <int name="maxInspections">5</int>
  <int name="minQueryLength">4</int>
  <float name="maxQueryFrequency">0.01</float>
</lst>
```

We are using the default internal distance measure and allowing edit distance up to 2.

If we ask for spellCheck suggestions using query `cari`:

```
/spell?spellcheck.q=cari
```

We get the following results:

```javascript
"spellcheck": {
  "suggestions": [
    "cari",
    {
      "numFound": 3,
      "startOffset": 0,
      "endOffset": 4,
      "origFreq": 0,
      "suggestion": [
        {
          "word": "camri",
          "freq": 4
        },
        {
          "word": "capri",
          "freq": 2
        },
        {
          "word": "carol",
          "freq": 6
        }
      ]
    }
  ]
}
```

Both `camri` and `capri` are at edit distance 1, hence appear before `carol`, which is at edit distance 2. Between `camri` and `capri`, `camri` comes first because it appears in more documents than `capri`.

Now let's add 20 more documents with title `car`. `car` is also at one edit distance from `cari`.

```javascript
"spellcheck": {
  "suggestions": [
    "cari",
    {
      "numFound": 4,
      "startOffset": 0,
      "endOffset": 4,
      "origFreq": 0,
      "suggestion": [
        {
          "word": "camri",
          "freq": 4
        },
        {
          "word": "capri",
          "freq": 2
        },
        {
          "word": "car",
          "freq": 20
        },
        {
          "word": "carol",
          "freq": 6
        }
      ]
    }
  ]
}
```

A bit strange, hah! `car`, even though has the same edit distance as `camri` and `capri` and more frequency, still appears below these two. Why? `car` should be the top result.

After doing some investigation, I came to know that for `internal` distance measure, results are sorted not by edit distance, but by a custom score, which is calculated as:

```php
score = 1 - (editDistance / Min(termLength, queryTermLength))

editDistance = Calculated edit distance (1 or 2)
termLength = Length (no. of characters) in the term being matched
queryTermLength = Length (no. of characters) of the query
```

Lets calculate scores for each of the matched terms:

```
queryTermLength = 4

camri & capri
--------------

editDistance = 1
termLength = 4
score = 1 - 1/min(4, 4) = 0.75

car
--------------

editDistance = 1
termLength = 3
score = 1 - 1/min(3, 4) = 0.66

carol
--------------

editDistance = 2
termLength = 4
score = 1 - 2/min(4, 4) = 0.5

```

The score of `camri` and `capri` is higher so they appear on top. `car`, even though more popular, will come below these two.

It can be a bit unexpected when a hugely popular item does not come on top. It can not be handled on the application side as well, since you do not get the edit distance back. If we just re-sort all the results by frequency, we can get results with a higher edit distance on top, which may not be desirable.

> The same score value is also used for accuracy threshold. If you check the spellcheck config we defined earlier, you can can see `accuracy = 0.5`. It means that the results with score < 0.5 are discarded.

# What Can We Do?

### Use only edit distance = 1

Using edit distances greater than 1 (Solr allows only 1 & 2) results in a lot of noisy results. If you use 1 as maximum edit distance, you can re-sort the results in application by frequency.
