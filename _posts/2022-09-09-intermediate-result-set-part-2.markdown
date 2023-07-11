---
layout: post
title:  Intermediate Result Set part 2
date:   2022-09-09 14:16:27 -0400
parent: Home
---
Awareness and understanding of the intermediate result set that is incrementally built up as query clauses are processed can be very helpful because it expands your limit of pursuable optimization options. There are certain scenarios where knowledge of the intermediate result set can result in a very different query pattern.

```clojure
[[?parent :parent/id]            ;; #{[?parent] [?parent] ...}
 [?child :child/parent ?parent]] ;; #{[?parent ?child] [?parent ?child] ...}
```
For example, consider a query with the query clauses listed above which unify around 100 distinct entities, creating an intermediate result set with the following structure: #{[?parent] [?parent] ...}. The following query clause then unifies to ?child entities that reference each ?parent, thus changing the structure of the intermediate result set to: #{[?parent ?child] [?parent ?child] ...}. Let’s explore what that intermediate result set might look like once we fill in the bindings with actual values.