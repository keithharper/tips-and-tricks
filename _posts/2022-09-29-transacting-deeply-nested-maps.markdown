---
layout: post
title:  "Transacting: Deeply Nested Entity Maps"
author: Keith Harper
date:   2022-09-29 14:16:27 -0400
categories: Transaction
parent: Home
---
When it comes to transacting data in Datomic, there is a certain level of comfort that users find in knowing that Datomic will only transact novel values when handed tx-data that includes values which already exist in the database. This can be very powerful and freeing since it provides the ability to do things like transact the entire schema for a database whenever a service starts up, but it’s very important to remember that nothing is free and there are costs that should be taken into consideration that might not feel impactful until your database becomes large enough to feel the pain of those costs (hopefully not in production, right? :nervous-smile:).

With the introduction of io-stats, we can now observe some costs that weren’t readily apparent in the past, but tend to be the most expensive parts of any system: retrieving data from storage & writing data to storage (i/o). 