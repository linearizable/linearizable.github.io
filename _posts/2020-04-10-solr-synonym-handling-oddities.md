---
layout: post
title: "An Investigation of Solr's Synonym Handling Oddities"
author:
- Vikas Kumar
---

>Do you use Solr?
>
>Do you use stopwords & synonyms with Solr?
>
>Do your synonyms have stopwords in them?
>
>If yes, then this article is probably for you. If you are a Solr/Lucene nerd like me who loves to deep dive into the intricacies of internals & under-the-hood implementations, then this article is definitely for you :-).

Solr has a long and turbulent history with multi-word synonyms (Just search `solr multi word synonyms` in Google or read [here](https://opensourceconnections.com/blog/2013/10/27/why-is-multi-term-synonyms-so-hard-in-solr/) & [here](https://lucidworks.com/post/multi-word-synonyms-solr-adds-query-time-support/)). `SynonymGraphFilter` finally solved most of these issues. But recently we (Search team at OLX Autos India) discovered a strange issue when dealing with synonyms which contain stop words, specifically if stop words appear at the beginning or end.

For example, if the synonyms file contains the following entry:
```
i phone, iphone
```
and `i` is also defined as a stopword, on account of it being a pronoun. If I search for `i phone` using the following Solr request:
```
/select?q=i phone&qf=title&defType=edismax&q.op=AND
```
Then this results into the equivalent of the following dismax parsed query:
```
+(title:iphone)
```
Whoa! Where did the `i phone` go? I expected the `i` of `i phone` being removed, on account of it being a stopword, and search with just `phone` i.e. I expected the parsedquery to be:
```
+(title:iphone title:phone)
```

>Note that this, in itself, isn't great from a good search experience perspective as it will return a lot of irrelevant results, but in terms of Lucene behaviour, this is what I expected (Ideally I would like i phone be searched as a phrase here. We'll talk about this later in this article).

If we have stopword somewhere in the middle, then it works as expected. So if I have the following in synonyms:
```
apple i phone, iphone
```
‌and I search `apple i phone`, then it generates the expected parsedquery:
```
+((title:iphone (+title:apple +title:phone)))
```
‌That's not it. It veers into even weirder territory if we have multiple synonyms (one or more of which contain stopwords at the end).

‌Let's take an example - synonyms for iPhone 6s:
```
iphone 6s, iphone 6 s, iphone6 s
```
‌with `s` being a stopword. When you search for `iphone 6s` using the following request:
```
/select?q=iphone 6s&qf=title&defType=edismax&q.op=AND
```
‌What is the final query that you expect? To my (and I'm sure, your) great surprise, the parsedquery is:
```
+(+(title:iphone) +(title:6) +(title:6s))
```
which is a weird combination. It'll try to find documents which contain `iphone` AND `6` AND `6s`. I was expecting it to match documents which contain `(iphone AND 6s) OR (iphone AND 6) OR (iphone6)`

We could not figure out why this was happening. This behaviour is not documented anywhere (or at least we could not find any). So I decided to debug the Solr codebase on my laptop to see what's going on at the implementation level. I cloned the [lucene-solr](https://github.com/apache/lucene-solr) repo, built it using ant and started debugging using IntelliJ debugger.

‌I'll share what I found but before that, I want to give some background on how multi-word synonyms are handled by Solr using `SynonymGraphFilter`.

>I use Lucene and Solr interchangeably in this article. Some of the functionalities described here are part of Solr and others are part of Lucene.


## Some Background on SynonymGraphFilter

This [article](http://blog.mikemccandless.com/2012/04/lucenes-tokenstreams-are-actually.html) provides a very good explanation of how a token stream with synonyms (potentially multi-word) is handled using a graph. Here I'll help you visualize how this graph is created using the examples we mentioned above.

Let's take the first example (`iphone` and `i phone`). If we analyse `i phone` using the analysis tool provided by Solr admin panel, we get the following analysis:

![alt text](https://cdn-images-1.medium.com/max/1600/1*DXIrh0p-lAoroTLRQpTWsQ.png)


We can think of each token as a node in the graph. To see how the graph is generated from this analysis, we need to consider the following:

* **position:** at which position does this node start. Here both i and iphone are at position 1 and phone is at the next position (position 2)
* **positionLength:** No. of hops that this node takes.

‌The above analysis generates the following graph:

![alt text](https://cdn-images-1.medium.com/max/1600/1*OxFeocbynHaLQoh00AtO9g.png)

Internally Lucene uses an attribute called `positionIncrement` to determine positions. `positionIncrement` for a node is defined as the gap (in terms of the number of nodes or positions) between start positions of this node and the previous node. `positionIncrement` always starts with the value of 1.

Let's see how this graph is represented by Lucene's internal data structure:

![alt text](https://cdn-images-1.medium.com/max/2400/1*kbza43Lyt6y5bK3lw0ZOEA.png)


‌In this case, the first token is `iphone` (`iphon` here because of stemming, but you can ignore that) so its `positionIncrement` is 1. Next token, `i`, starts at the same position as the previous token (iphone), so its increment will be 0. Next token, `phone`, starts at the next position, so its increment is 1.

‌Here's a more readable format:

|  | iphone | i | phone
| -------- | -------- | -------- | -------- |
| Position Increment     | 1     | 0     | 1
| Position Length     | 2    | 1     | 0

>positionLength and positionIncrement are important concepts in Lucene's synonym handling as well as for understanding the issues we are describing in this post.

Similarly, let's take a look at the analysis for the second example (`iphone 6s`), again without considering stopwords:

![alt text](https://cdn-images-1.medium.com/max/2400/1*0xknN3YhTSXjrGaQkDSvUQ.png)

And the corresponding graph:

![alt text](https://cdn-images-1.medium.com/max/1600/1*_TvxgJQwYMVXetTlEow5PQ.png)

Here's the internal representation (with a readable tabular format):

|  | iphone | iphone6 | iphone | 6 | s | s | 6s
| -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- |
| Position Increment     | 1     | 0     | 0 | 1 | 1 | 1 | 1
| Position Length     | 1    | 3     | 4 | 1 | 2 | 1 | 1

I hope it's pretty clear now how positionLength and positionIncrement attributes work and how we can visualize the graph with these.

## Adding Stopwords

Now that we have seen how SynonymGraphFilter works, let's add the stopwords back and investigate what causes the weird behaviour.

### Case #1

First, let's examine case #1 (`iphone` and `i phone`), where entire synonym is dropped if a stopword appears at the beginning or end. We add `i` as a stopword and analyse again. Here's the analysis:

![alt text](https://cdn-images-1.medium.com/max/1600/1*6O4YLPKitHalmffKMwkKQw.png)

As you can see, `i` is dropped. Rest everything remains the same. We can now visualize this as the following graph:

![alt text](https://cdn-images-1.medium.com/max/1600/1*tUcSakuzoTemeiSsdxZMxw.png)

The corresponding internal representation is:

|  | iphone | phone
| -------- | -------- | -------- |
| Position Increment     | 1     | 1
| Position Length     | 2    | 1

Lucene [builds](https://github.com/apache/lucene-solr/blob/releases/lucene-solr/8.4.0/lucene/core/src/java/org/apache/lucene/util/graph/GraphTokenStreamFiniteStrings.java#L200) an automaton (which is sort of a graph similar to above) to generate the final query strings. This method takes the above graph as input, does some processing and returns another graph (said automaton) which is finally traversed to generate the query. This conversion is where all the issues we encountered are found. Specifically, two things that it does:‌

* [Remove dead states](https://github.com/apache/lucene-solr/blob/releases/lucene-solr/8.4.0/lucene/core/src/java/org/apache/lucene/util/automaton/Operations.java#L983) i.e. states which are not reachable from the start or the end state is not reachable from it
* How it handles gaps introduced by removing stopword nodes

The current issue is caused by removing dead states. As we can see in the graph above, `phone` node is not reachable from the start state. So it is marked as a dead state and removed. Hence it does not appear in the final query.

This will happen for all the synonyms where either stopword appears at the beginning (not reachable from start) or at the end (the end state will not be reachable).

### Case #2

Now let's examine the other case where searching for `iphone 6s` results in a weird token combination being searched for.

If we analyse `iphone 6s` again with `s` added to stopword list, we can see the following analysis:

![alt text](https://cdn-images-1.medium.com/max/2400/1*upx5E23_8QG3cyNNm6ZYUQ.png)


It is equivalent to the following graph:

![alt text](https://cdn-images-1.medium.com/max/1600/1*_elE61LbtrnKJN1gSYiKnw.png)

If we go by the intuition gained from debugging case #1, we can say that this will ignore the lower two branches as they can not reach the end state and will only search `iphone 6s`, but that doesn't happen. Instead, Solr ends up searching for `iphone 6 6s`. How does that happen?

‌Let's look at the internal data structure:

![alt text](https://cdn-images-1.medium.com/max/2400/1*ONJa5AMdTO0c6kYtm6bMiw.png)


In tabular format, this translates to:

|  | iphone | iphone6 | iphone | 6 | 6s
| -------- | -------- | -------- | -------- | -------- | -------- |
| Position Increment     | 1     | 0     | 0 | 1 | 3
| Position Length     | 1    | 3     | 4 | 1 | 1

This will generate the same graph as above, which is then used to build the automaton.

The process of building the automaton is as follows:‌

* Traverse the tokes in the stream. For each token:
* Identify `start` and `end` index in the automaton (which correspond to nodes) and create an edge (which corresponds to the token)

The code is a bit involved but pretty straightforward if we step through it using the debugger. Let's walk through how the automaton is built. Indexes start with 0 (identified by `pos` in code).

The first token is `iphone`. It's `start` is 0 and `end` is 1 (based on `positionLength`)

![alt text](https://cdn-images-1.medium.com/max/1600/1*JJ8BauM3gqfiE8Y1-eWTTg.png)

The second token is `iphone6`. Since its `positionIncrement` is 0, we do not move the `start` forward (it'll still be 0). `end` will be 3 (`start + positionLength`)

![alt text](https://cdn-images-1.medium.com/max/1600/1*WC1ojA1eEm24muxfnR2VcQ.png)

The third token is `iphone`. `start` is still 0 as `positionIncrement` is 0. end is 4 (`start + positionLength`)

![alt text](https://cdn-images-1.medium.com/max/1600/1*ezyY9S1rAlEcB2-rI9H-Eg.png)

Next token is `6`. Since `positionIncrement` is 1, we move `start` by 1 so it becomes 1. `end` will be 2 (`start + positionLength`)

![alt text](https://cdn-images-1.medium.com/max/1600/1*Y4dzHVfhMBOLc2suX_MrTg.png)

The last token is `6s`. This is where it gets tricky. The `positionIncrement` is 3. So we expect the `start` to move to 4 and create an edge from `4 -> 5`, which will give us `iphone 6s (0 -> 4 -> 5`).

![alt text](https://cdn-images-1.medium.com/max/1600/1*P6-XDXIvRF4Z1WXqwywyhQ.png)

But as you might have guessed, this is not what happens. Instead `start` is incremented by only 1. It's always incremented by 1 (`pos++`) if `positionIncrement` is greater than 0. So what we get is an edge from `start` (which is now 2) to `end` (which is 5: `start + positionLength + gap`, where `gap` is `positionIncrement -1`)

![alt text](https://cdn-images-1.medium.com/max/1600/1*_rYGpdQY0kKDmGvIkGf-pA.png)

And there you have it. `0 -> 3 (iphone6)` and `0 -> 4 (iphone)` are eliminated on account of being dead states and we are left with iphone 6 6s, which is what's finally searched

I fixed this issue by adding the gap to pos before creating the edge.



![alt text](https://cdn-images-1.medium.com/max/1600/1*yPuUXvvMHY7QRjL4BEna9A.png)

But I'm not sure that it'll not break other cases (such as cases where stopwords appear in the middle e.g. `apple i phone`).

I think properly handling the gap introduced by removing stopwords requires identifying synonym boundaries and attaching tokens to their synonyms (I don't have the final solution yet, but I'm working on it :-))


## What Did We Do?
First, we removed single character words from our stopwords list, since most of the problematic cases were because of those.

Secondly, for better precision, we wanted the synonyms to be searched as phrases. In the case of `i phone`, instead of matching documents where `i` and `phone` appear anywhere (which could very well be in separate contexts e.g. `I want to sell my Samsung phone`), we wanted to match documents where `i` and `phone` appear together. Solr provides a way to do that. We just need to add `autoGeneratePhraseQueries="true"`` in the field type. For example:

```
<fieldType name="text_en" class="solr.TextField"autoGeneratePhraseQueries="true" positionIncrementGap="100" docValues="false" multiValued="false">
```
Note that this can sacrifice some recall in favour of better precision.

## Closing Thoughts
I had quite a fun time debugging this issue and getting to know the internals of Lucene better. I'll try to raise an issue on Lucene github or Jira to get help from the experts. We can either get this behaviour documented somewhere or probably fix it in the code.

Also, I would recommend everyone working on search with Solr to setup live debugging environment on their local machines. Lucene and Solr are complex beasts with a lot of complex algorithms and it helps immensely if we can step through the code and see what's happening exactly. Apart from this issue, it has also helped me debug/understand a number of other issues involving spellcheck, proximity boosting and geospatial search. I used the following to help me set up the environment-

[https://cwiki.apache.org/confluence/display/LUCENE/HowtoConfigureIntelliJ](https://cwiki.apache.org/confluence/display/LUCENE/HowtoConfigureIntelliJ)

[https://opensourceconnections.com/blog/2015/04/30/debugging-solr-5-in-intellij](https://opensourceconnections.com/blog/2015/04/30/debugging-solr-5-in-intellij)

Thanks for reading through.
