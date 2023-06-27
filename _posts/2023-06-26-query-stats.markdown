---
layout: post
title:  Leveraging Query Stats
date:   2023-06-26 14:16:27 -0400
nav_order: 2
parent: Home
---
# {{ page.title }}
![before-after.png]({{ site.baseurl }}/assets/img/qs-before-after.png)
As the writer of a  [Datomic query](https://docs.datomic.com/on-prem/query/query.html), you have control over query clause ordering which can be very powerful if you have sufficient information to order query clauses by most-to-least selective. Some measures of clause selectivity are the amount of rows flowing in and out of each clause, as well as the boundedness of logic variables used by each clause. The information returned by query-stats can inform you of those measures and be used to make an informed decision about clause ordering.

We will be using the [MusicBrainz sample database code repository](https://github.com/Datomic/mbrainz-sample) for all examples in this article. For a quick walkthrough on how to get Datomic running locally with the [MusicBrainz sample database](https://github.com/Datomic/mbrainz-sample#getting-the-data), please check out the [README](https://github.com/Datomic/mbrainz-sample/blob/master/README.md#getting-started).

We’re going to use query-stats to achieve the “after” state depicted in the image above by keeping the intermediate result set as small as possible in between clauses.

## How to request query-stats
Starting in [v1.0.6610 of Datomic On-Prem一soon](https://forum.datomic.com/t/datomic-1-0-6610-now-available/2176) to be released to Datomic Cloud一users can now ask for query-stats when running queries.

To run a query while accumulating query-stats, we will need to leverage [datomic.api/query](https://docs.datomic.com/on-prem/clojure/index.html#datomic.api/query) and supply `:query-stats true` in the query-map argument.

Some pieces of query-stats will be explained in more detail in the following sections. For additional information, please refer to the [query-stats user documentation](https://docs.datomic.com/on-prem/api/query-stats.html).

## Scenario: use query-stats while optimizing clause ordering

In this scenario, we are going to wield query-stats to guide our thought process as we seek optimal clause ordering.

We will run a query that will find all albums released before 1970 in which at least one of the artists was John Lennon, then return the album name and year it was released.
```clojure
{:query       '{:find  [?album-name ?year]
                :in    [$ ?artist-name]
                :where [[?artist :artist/name ?artist-name]
                        [?release :release/artists ?artist]
                        [?release :release/name ?album-name]
                        [?release :release/year ?year]
                        [(< ?year 1970)]]}
 :args        [db "John Lennon"]
 :query-stats true}
```

The results for requesting query-stats for this query can be found at: https://gist.github.com/keithharper/7931e836d51b453eb3ab43141685a3d2

### `:phases`

Each item under `:phases` represents a collection of clauses that will be processed as a single unit of work. For most queries, there may only be a single item under `:phases`, but certain queries will have multiple items under `:phases` (e.g. ones that use query rules)

### `:sched`

`:sched` is short for “schedule” and represents the rough plan that the query engine will use when processing this query.
```clojure
(([(ground $__in__2) ?artist-name]
  [?artist :artist/name ?artist-name]
  [?release :release/artists ?artist]
  [?release :release/name ?album-name]
  [?release :release/year ?year]
  [(< ?year 1970)]))
```
You may notice that there is an extra query clause in the schedule which wasn’t in the collection of clauses we supplied to `:where`. `[(ground $__in__2) ?artist-name]` is how the value (John Lennon) and logic variable or lvar (`?artist-name`) passed to `:in` get bound by the query engine.

### `:clauses`

`:clauses` is fairly self-explanatory, but for clarity this contains the query clauses that were run for this phase.

| clause                               | rows-in | rows-out | binds-in                | binds-out               | expansion | preds              |
|--------------------------------------|---------|----------|-------------------------|-------------------------|-----------|--------------------|
| [(ground $__in__2) ?artist-name]     | 0       | 1        | ()                      | [?artist-name]          | 1         |                    |
| [?artist :artist/name ?artist-name]  | 1       | 1        | [?artist-name]          | [?artist]               |           |                    |
| [?release :release/artists ?artist]  | 1       | 21       | [?artist]               | [?release]              | 20        |                    |
| [?release :release/name ?album-name] | 21      | 21       | [?release]              | [?album-name  ?release] |           |                    |
| [?release :release/year ?year]       | 21      | 3        | [?album-name  ?release] | [?year  ?album-name]    |           | ([(< ?year 1970)]) |

The table above might make it somewhat easier to interpret the query-stats for clauses. Notice how the `:rows-in` & `:binds-in` for one clause directly correspond to the `:rows-out` & `:binds-out` for the previous clause.

#### Clause 1
```clojure
{:clause    [(ground $__in__2) ?artist-name],
 :rows-in   0,
 :rows-out  1,
 :binds-in  (),
 :binds-out [?artist-name],
 :expansion 1}
```
Again, this clause simply represents the binding of the value and lvar that were passed to `:in`.

`:rows-in 0` indicates that there were no rows in the result set before processing this clause.

`:rows-out 1` indicates that the result set now has 1 clause in it which contains the binding indicated by `:binds-out [?artist-name]`.

`:expansion 1` simply means that `:rows-out` expanded in size by 1 row after processing this clause.

#### Clause 2
```clojure
{:clause    [?artist :artist/name ?artist-name],
 :rows-in   1,
 :rows-out  1,
 :binds-in  [?artist-name],
 :binds-out [?artist]}
```

Notice how `:binds-out` no longer has `?artist-name`, but does have a new binding of `?artist`. This is because the query engine has visibility into the clauses that will be processed after this one, and none of those clauses use `?artist-name`, so it can be safely removed from the result set.

#### Clause 3
```clojure
{:clause    [?release :release/artists ?artist],
 :rows-in   1,
 :rows-out  21,
 :binds-in  [?artist],
 :binds-out [?release],
 :expansion 20}
```

Now we get to the first clause that introduces an actual result set expansion. The query-stats for this clause indicate that 21 releases were found in which John Lennon was one of the artists.

![clause3-clause4.png]({{ site.baseurl }}/assets/img/clause3-clause4.png)

#### Clause 4
```clojure
{:clause    [?release :release/name ?album-name],
 :rows-in   21,
 :rows-out  21,
 :binds-in  [?release],
 :binds-out [?album-name ?release]}
```

This clause introduces new values into the result set (as indicated by the difference between binds in and out), but the number of rows in the result set doesn’t grow. What do you think would happen if we were to make this the last clause in our `:where` clauses?

#### Clause 5
```clojure
{:clause    [?release :release/year ?year],
 :rows-in   21,
 :rows-out  3,
 :binds-in  [?album-name ?release],
 :binds-out [?album-name ?year],
 :preds     ([(< ?year 1970)])}
```

The final clause really narrows down the result set and showcases a piece of information that wasn’t readily apparent before: `:preds`. The presence of a :preds key here indicates that the corresponding clauses were used as filter predicates while finding answers to `[?release :release/year ?year]`.

Now let’s take a look at what would happen if we were to move clause #4 to the end of our `:where` clauses.

## Scenario: Perform All Necessary Joins, Then Project Values

We’re going to take the same query we ran before, except this time we’ll observe what happens when we move `[?release :release/name ?album-name]` to the very end of our `:where` clauses.

### Query
```clojure
{:query '{:find  [?album-name ?year]
          :in    [$ ?artist-name]
          :where [[?artist :artist/name ?artist-name]
                  [?release :release/artists ?artist]
                  [?release :release/year ?year]
                  [(< ?year 1970)]
                  [?release :release/name ?album-name]]}
 :args  [db "John Lennon"]}
```

When we run this query, something very interesting happens that might not be immediately obvious.

The results for requesting query-stats for this query can be found at: <https://gist.github.com/keithharper/2e8d1498d2176c3fcdb0311fc7180002>

| clause                               | rows-in | rows-out | binds-in          | binds-out            | expansion | preds              |
|--------------------------------------|---------|----------|-------------------|----------------------|-----------|--------------------|
| [(ground $__in__2) ?artist-name]     | 0       | 1        | ()                | [?artist-name]       | 1         |                    |
| [?artist :artist/name ?artist-name]  | 1       | 1        | [?artist-name]    | [?artist]            |           |                    |
| [?release :release/artists ?artist]  | 1       | 21       | [?artist]         | [?release]           | 20        |                    |
| [?release :release/year ?year]       | 21      | 3        | [?release]        | [?year  ?release]    |           | ([(< ?year 1970)]) |
| [?release :release/name ?album-name] | 3       | 3        | [?year  ?release] | [?album-name  ?year] |           |                    |

If we take another look at clause #4 from our first query:
```clojure
{:clause    [?release :release/name ?album-name],
 :rows-in   21,
 :rows-out  21,
 :binds-in  [?release],
 :binds-out [?album-name ?release]}
```
We can observe the fact that the result set now contains 21 items which each contain 2 values, resulting in a total of 42 values in the result set.

Now let’s compare that to the second query:

```clojure
;; was clause #5, now clause #4
{:clause    [?release :release/year ?year],
 :rows-in   21,
 :rows-out  3,
 :binds-in  [?release],
 :binds-out [?year ?release],
 :preds     ([(< ?year 1970)])}
```

```clojure
;; was clause #4, now clause #5
{:clause    [?release :release/name ?album-name],
 :rows-in   3,
 :rows-out  3,
 :binds-in  [?year ?release],
 :binds-out [?album-name ?year]}
```
Clause #4 in the second query reduces our result set from 21 down to 3, but introduces another value into the result set. The result set now contains 3 rows which each contain 2 values, resulting in a total of 6 values.

Now our clause that finds the album name only has to do so for 3 albums instead of finding them for 21 albums and throwing out all but 3 of those album names after narrowing down the release year, which you can imagine might save quite a bit of work at larger scales.

## Conclusion

This is just one of the ways you can use the information returned by query-stats to understand and improve your queries. You may also find query-stats to be useful when reasoning through why your query performs different given a different distribution of the underlying data, reasoning through recursive query rules, etc.