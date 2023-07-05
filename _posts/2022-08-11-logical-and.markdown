---
layout: post
title:  Logical AND
author: Keith Harper
date:   2022-08-11 18:16:27 -0400
parent: Home
---
Writing a query that uses a collection of values and returns results where all values in the input collection match values for a cardinality-many attribute is a challenge.

For example: Find release names where all artists in ?valid-artist-names are present under the:release/artists attribute.