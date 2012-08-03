---
layout: post
title: "Continuous Bloom filter"
date: 2012-05-12 11:23
comments: true
categories: [algorithm, ruby, probability, streaming, opensource]
published: true
---
Working with large, continuous data streams has many challenges. The throughput or the quantity of data in a given time period is sometimes too high for some types of computations using traditional algorithms and data structures. In these cases [probabilistic data structures and algorithms](http://highlyscalable.wordpress.com/2012/05/01/probabilistic-structures-web-analytics-data-mining/) can be used to compute estimations and trade some precision for memory consumption. These probabilistic data structures are typically a much more compact alternative for stream based computing.

Probably one of the most widely used probabilistic data structure is the Bloom filter. Basically a Bloom filter is used to test whether an element is member of a set in a very memory efficient way. Because of the probabilistic nature of a Bloom filter, false positives are possible (but false negatives are not). This mean testing for the inclusion of an element will report it as being part of the set when it is not, statistically within a known error rate. There are several very good articles on the subject of Bloom filters which explain in great length everything there is to know about Bloom filters.  See the references section for more info.

When dealing with unbounded data streams, standard Bloom filters are not of much help because they are built using a static structure based on the required filter capacity and false positives error rate. In fact, a Bloom filter is initialized using two parameters, **m** and **k** where m in the total number of positions (or bits for standard filters) and k the numbers of hashing functions used for each element. A [simple formula](http://www.siaris.net/index.cgi/Programming/LanguageBits/Ruby/BloomFilter.rdoc) can derive optimal m and k values for a required filter capacity and error rate. For example, for a filter capacity of **1000** elements and an error rate of **0.1%**, **14378** positions (m) and **10** hash functions (k) are required.

With data streams the question usually is for how long do we need to accumulate data. If you can estimate the average throughput of the stream you can estimate how much data you will need to work with for the required period. We will call it the data TTL.  

### Rotating Filters
<div class="inline_image_right"><img src="/images/filterring.png"></div>My first idea for Bloom filters with unbounded data streams was to use a filter ring of N+1 filters where each filter would hold data for a period of TTL/N. The extra +1 filter would be for the overlap to insure elements would not expire before the required TTL. At each TTL/N period the oldest filter would be reset and used as the new current, most recent filter. Elements would always be inserted in the current filter and lookups would require testing for inclusion in the N+1 filters. This method is simple by design but requires N+1 lookups. In a dedupping scenario where each element must be tested, this may be a problem. An advantage of this design is that it would be possible to trigger the filter rotation upon reaching the current filter capacity thus preserving the required error rate at the expense of having a *dynamic* TTL. 

### Continuous Filter
Ilya Grigorik's post [Flow Analysis & Time-based Bloom Filters](http://www.igvita.com/2010/01/06/flow-analysis-time-based-bloom-filters/) and the [Stable Bloom Filters](http://webdocs.cs.ualberta.ca/~drafiei/papers/DupDet06Sigmod.pdf) paper inspired this idea of a time-based *continuous* filter. Attaching a TTL to each element of the filter is a good proposal but the challenge is how to encode the TTL in a space efficient way and how to actually track time and expire elements. My target was to see if I could come up with a solution by using 4 bits or less per filter position. Ideally the number of bits must be a power of 2 to avoid padding in our bit vector. 2 bits seemed too small, leaving 4 bits as the next smallest size.

<div class="inline_image_right"><img src="/images/timekeeping.png"></div>The rotating filters idea actually lead to the idea of encoding time in a round robin clock of 15 positions, that's the 4 bits 16 position less the 0 value reserved for the *unset* value. The internal clock resolution is set to half of the required TTL (resolution divisor of 2); the clock ticks every TTL / 2. The *current time* is the current clock tick modulo 15. The total time of our internal clock is 15 * (TTL / 2). 


We keep track of TTLs by writing the current time in the key k positions when inserting in the filter. When doing a key lookup if the interval between the current time and the inserted time value (elapsed time) at any of the k position is **greater than** 2 (resolution divisor) we know this key is expired. Any *expired* position is reset to zero.

<div class="center_image"><img src="/images/sequence.png"></div>

For example, in a filter using 3 hash functions, X is inserted at 4, Y inserted at 9 (overwriting one of X's position). A lookup for X is made at 11. In two of X position, elapsed time is 11 - 4 = 7 which is longer then the resolution divisor of 2. X is reported as not found and the two expired positions are reset to zero.

A fast continuous Bloom filter Ruby implementation is available in the [Bloombroom gem](https://rubygems.org/gems/bloombroom) and [source code on Github](https://github.com/colinsurprenant/bloombroom).

### References
- [Bloom filter on wikipedia](http://en.wikipedia.org/wiki/Bloom_filter)
- [Scalable Datasets: Bloom Filters in Ruby](http://www.igvita.com/2008/12/27/scalable-datasets-bloom-filters-in-ruby/)
- [Flow Analysis & Time-based Bloom Filters](http://www.igvita.com/2010/01/06/flow-analysis-time-based-bloom-filters/)
- [Stable Bloom filters paper](http://webdocs.cs.ualberta.ca/~drafiei/papers/DupDet06Sigmod.pdf)
- [Probabilistic Data Structures for Web Analytics and Data Mining](http://highlyscalable.wordpress.com/2012/05/01/probabilistic-structures-web-analytics-data-mining/)
- [The maths to compute optimal m and k ](http://www.siaris.net/index.cgi/Programming/LanguageBits/Ruby/BloomFilter.rdoc)

<div class="github"><a href="https://github.com/colinsurprenant/bloombroom">Blombroom on Github</a></div>
