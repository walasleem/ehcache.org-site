---
---
# Cache Eviction Algorithms

 

## Introduction

A cache eviction algorithm is a way of deciding which element to evict when the cache is full.
In Ehcache, the `MemoryStore` may be limited in size (see [How to Size Caches](/documentation/2.7/configuration/cache-size) for more information). When the store gets full, elements are evicted. The eviction algorithms in Ehcache determine which
elements are evicted. The default is LRU.

What happens on eviction depends on the cache configuration. If a `DiskStore` is configured,
the evicted element will overflow to disk (is *flushed* to disk); otherwise it will be removed. The `DiskStore` size by default is unbounded. But a maximum size can be set (see [Sizing Caches](/documentation/2.7/configuration/cache-size) for more information). If the `DiskStore` is full, then adding an element
will cause one to be evicted unless it is unbounded. The `DiskStore` eviction algorithm is not configurable. It uses LFU.

**Notes for distributed caches**: There is no user selection of eviction algorithms with clustered caches. The attribute MemoryStoreEvictionPolicy is ignored (a clock eviction policy is used instead), and if allowed to remain in a clustered cache configuration, the MemoryStoreEvictionPolicy may cause an exception. In addition, the local `DiskStore` is not used in distributed cache, which relies on the Terracotta Server Array for storage.

## Provided MemoryStore Eviction Algorithms

The idea here is, given a limit on the number of items to cache, how to choose the thing to evict that
gives the *best* result.

In 1966 Laszlo Belady showed that the most efficient caching algorithm would be to always discard the
information that will not be needed for the longest time in the future. This it a theoretical result
that is unimplementable without domain knowledge. The Least Recently Used ("LRU") algorithm is often used as
a proxy. It works pretty well because of the locality of reference phenomenon and is the default in most caches. 

A variation of LRU is the default eviction algorithm in Ehcache.

Altogether Ehcache provides three eviction algorithms to choose from for the `MemoryStore`.

### Least Recently Used (LRU)

 This is the default and is a variation on Least Frequently Used.

 The oldest element is the Less Recently Used (LRU) element. The last used
timestamp is updated when an element is put into the cache or an
element is retrieved from the cache with a get call.

 This algorithm takes a random sample of the Elements and
evicts the smallest. Using the sample size of 15 elements, empirical testing shows
that an Element in the lowest quartile of use is evicted 99% of the time.

 If probabilistic eviction does not suit your application, a true Least Recently Used
deterministic algorithm is available by setting `java -Dnet.sf.ehcache.use.classic.lru=true`. 

### Least Frequently Used (LFU)

 For each get call on the element the number of hits is updated. When a
put call is made for a new element (and assuming that the max limit is
reached) the element with least number of hits,
the Least Frequently Used element, is evicted.

 If cache element use follows a pareto distribution, this algorithm may give better
results than LRU.

 LFU is an algorithm unique to Ehcache. It takes a random sample of the Elements and
evicts the smallest. Using the sample size of 15 elements, empirical testing shows
that an Element in the lowest quartile of use is evicted 99% of the time.

### First In First Out (FIFO)

 Elements are evicted in the same order as they come in. When a put call
is made for a new element (and assuming that the max limit is reached
for the memory store) the element that was placed first (First-In) in
the store is the candidate for eviction (First-Out).

 This algorithm is used if the use of an element makes it less likely to be used
in the future. An example here would be an authentication cache.

 It takes a random sample of the Elements and
evicts the smallest. Using the sample size of 15 elements, empirical testing shows
that an Element in the lowest quartile of use is evicted 99% of the time.

## Plugging in your own Eviction Algorithm

Ehcache 1.6 and higher allows you to plugin in your own eviction algorithm. You can utilise
any Element metadata which makes possible some very interesting approaches. For example, evict
an Element if it has been hit more than 10 times.

<pre>
/**
* Sets the eviction policy strategy. The Cache will use a policy at startup.
* There are three policies which can be configured: LRU, LFU and FIFO. However
* many other policies are possible. That the policy has access to the whole element
* enables policies based on the key, value, metadata, statistics, or a combination
* of any of the above. 
* 
* It is safe to change the policy of a store at any time. The new policy takes
* effect immediately.
*
* @param policy the new policy
*/
public void setMemoryStoreEvictionPolicy(Policy policy) {
  memoryStore.setEvictionPolicy(policy);
"/>
</pre>

A Policy must implement the following interface:

<pre>
public interface Policy {
  /**
   * @return the name of the Policy. Inbuilt examples are LRU, LFU and FIFO.
   */
  String getName();
  /**
   * Finds the best eviction candidate based on the sampled elements. What 
   * distinguishes this approach from the classic data structures approach is
   * that an Element contains metadata (e.g. usage statistics) which can be used
   * for making policy decisions, while generic data structures do not. It is
   * expected that implementations will take advantage of that metadata.
   *
   * @param sampledElements this should be a random subset of the population
   * @param justAdded we probably never want to select the element just added.
   * It is provided so that it can be ignored if selected. May be null.
   * @return the selected Element
   */
  Element selectedBasedOnPolicy(Element[] sampledElements, Element justAdded);
  /**
   * Compares the desirableness for eviction of two elements
   *
   * @param element1 the element to compare against
   * @param element2 the element to compare
   * @return true if the second element is preferable for eviction to the first
   * element under ths policy
   */
  boolean compare(Element element1, Element element2);
"/>
</pre>

## DiskStore Eviction Algorithms

The `DiskStore` uses the Least Frequently Used algorithm to evict an element when it is full.
