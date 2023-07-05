---
layout: post
title:  Logical OR
author: Keith Harper
date:   2022-08-11 14:16:27 -0400
parent: Home
---
Binding a single variable to multiple values passed as a collection is what a collection binding does.

For example, the collection binding can be used to ask or questions, such as which publications are associated with either Paul McCartney or George Harrison.

# Database
A database of artists such as Anitta, Rick Astley, and Paul McCartney is used to demonstrate the concept collection bindings:

```clojure
(require '[datomic.api :as d])
(def db [[1 :release/name "Strange Collaboration"]
         [2 :release/name "Meu Lugar"]
         [3 :release/name "Only the Very Very Best"]

         [10 :artist/name "Anitta"]
         [1 :release/artists 10]
         [2 :release/artists 10]

         [11 :artist/name "Rick Astley"]
         [1 :release/artists 11]

         [12 :artist/name "Paul McCartney"]
         [3 :release/artists 12]])
```

# Logical OR
Since collection bindings provide logic or semantics, it is possible for them to find all releases in which either Anitta or Rick Astley were collaborators.

For example, find all releases where at least one of the following matches are found `[?artist :artist/name "Anitta"]` or `[?artist :artist/name "Rick Astley"]`:

```clojure
(d/q '[:find ?release-name
       :in $ [?artist-name ...]
       :where
       [?artist :artist/name ?artist-name]
       [?release :release/artists ?artist]
       [?release :release/name ?release-name]]
     db
     #{"Anitta"
       "Rick Astley"})

=>
{["Strange Collaboration"] ["Meu Lugar"]}
```

The semantics of Logical OR
Sometimes it can be confusing to understand the semantics of logical OR.  For example, the query below returns releases in which at least 1 of the collaborators is not Anitta or not Rick Astley. However, it is also possible to search for releases in which neither Anitta nor Rick Astley were collaborators.

For example, find all releases where at least one of the following matches are found: `(not [?artist :artist/name "Anitta"])` or `(not [?artist :artist/name "Rick Astley"])`:

```clojure
(d/q '[:find ?release-name
       :in $ [?artist-name ...]
       :where
       (not [?artist :artist/name ?artist-name])
       [?release :release/artists ?artist]
       [?release :release/name ?release-name]]
     db
     #{"Anitta"
       "Rick Astley"})

=>
{["Strange Collaboration"] ["Meu Lugar"] ["Only the Very Very Best"]}
```

# lvar (logic variable)
The query below finds releases in which neither Anitta nor Rick Astley were collaborators. A collection binding was not used here; instead, a lvar (logic variable) that contained the input collection of artist names was used.

There is a shortcoming with the approach shown here. The clause `[?artist :artist/name ?artist-name]` unifies to all entities that have a value for artist/name, even if the name isn't in the provided input collection of artist names.

For example, find all releases where ?artist-name is not present in the provided ?valid-artist-names:


(not [(contains? #{"Anitta" "Rick Astley"} "Paul McCartney")])
And


(not [(contains? #{"Anitta" "Rick Astley"} "Rick Astley")])
And


(not [(contains? #{"Anitta" "Rick Astley"} "Anitta")])

(d/q '[:find ?release-name
:in $ ?valid-artist-names
:where
[?artist :artist/name ?artist-name]                  
Warning: The clause above returns all ?artist with a value for :artist/name.


       (not [(contains? ?valid-artist-names ?artist-name)])
       [?release :release/artists ?artist]
       [?release :release/name ?release-name]]
     db
     #{"Anitta"
       "Rick Astley"})

=>
{["Only the Very Very Best"]}

