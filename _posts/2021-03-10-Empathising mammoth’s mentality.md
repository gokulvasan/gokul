---
layout: post
title: Empathising mammoth’s mentality
categories:
  - Blog Posts
last_modified_at: 2021-03-10T12:25:10-05:00
---

  <style>
div.scrollFormula {
  overflow: auto;
  white-space: nowrap;
}
</style>
  <script type="text/x-mathjax-config"> MathJax.Hub.Config({ TeX: { equationNumbers: { autoNumber: "all" } } }); </script>
  <script type="text/x-mathjax-config">
         MathJax.Hub.Config({
           tex2jax: {
             inlineMath: [ ['$','$'], ["\\(","\\)"] ],
             processEscapes: true
           }
         });
  </script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML" type="text/javascript"></script>

#  A study on Linux Memory Management.
This series of articles tries a hand on hand analysis of memory management policies accommodated in acclaimed Unix-like operating systems like OpenSolaris, FreeBSD with Linux.

Memory management conventionally  has three major policies.

1. **Fetch Policy**: When to fetch a page from auxiliary memory.
2. **Placement Policy:** Where to place the fetched page in main memory.
3. **Replacement Policy:** Which page to replace in case of memory shortage.

The mentioned policies are well studied and reasoned( Though I will write on this as well), nevertheless another policy which directly impacts the performance of a system is **scan rate policy**.

Rest of this specific work will focus on this policy and its approach in different operating system.

**```What is a scan rate policy?```**

Operating systems keeps updating the position of the allocated pages in their corresponding list based on its access frequency and other parameters. This enables the replacement policy to fetch the pages based on the position for replacement. Scanning through the entire list of allocated pages and updating the position of every page each time is an expensive operation. To avoid such a colossal expense, operating systems will only scan a portion of the list at a given moment of time and updates the corresponding page’s position. The policy that decides on the fraction to scan at a given instance of time is termed as scan rate policy.

**Generalities in the model.** Habitually on nearing a particular distance to exhaustion, the system starts a kernel level daemon, that runs in concurrence with other processes, termed either as pageout daemon( OpenSolaris and OpenBSD) or Kswapd daemon( Linux); Albeit the name of the process changes based on platform implementation, its intention remains the same .i.e., scan a certain amount of pages, apply the desired replacement policy and evict a set of pages to leeway for the current memory need. The free list of pages might have many waypoints in between the list called watermark. In Common, watermark decides on when to start the scan, rate of scan, number of times to scan, amount of processor utilisation page scanner can take( called CPU CAP), amount of I/O utilisation page scanner can use( called I/O CAP), etc., but the mentioned set varies both in implementation and inclusion based on the platform of implementation. Extensively, many operating systems quantify the measure of watermarks in page numbers.

Implementation In OpenSolaris:

OpenSolaris uses watermark based policy, i.e., scan rate policy changes based on the watermark .

Model. OpenSolaris have three major watermarks, namely lotsfree, desmem, minfree. The system has two different scan policies called slow scan and fast scan. The system also have two different rate at which the scanning will happen per second, i.e., 4times/second and 100times/second. Further, Solaris restricts the CPU utilisation of the page scanner between 4( min_percent_cpu) to 80( max_percent_cpu) percent( Used as a scaling factor) and the I/O is  capped at either 40 or 60 IO/second( maxpgio ), but this restriction is based on the formulae $ \frac{diskRPM*2}{3}$,Intuitively it means $ \frac{2}{3}$  busy on disk arm is all that is tolerable for paging.

By default lotsfree is $ \frac{1}{64}$  of total memory, desmem is $ \frac{1}{2}$ of lotsfree and minfree is $ \frac{1}{2}$ of desmem.  Fast scan is scaled to $ \frac{Total Memory}{2}$ and slow scan is hardcoded with 100. The whole of this  value initialisation is done  in function [setupclock](http://fxr.watson.org/fxr/source/common/os/vm_pageout.c?v=OPENSOLARIS;im=3#L214).

Another watermark worth mentioning is throttlefree. This watermark in general is equated with minfree.

![solarisscanrate2](https://user-images.githubusercontent.com/79076337/110906829-87825080-8332-11eb-8098-c2f1dc09b0f6.png)


**Policy.** Once memory reaches lotsfree watermark, pageout daemon kickstarts. The amount of page to scan is determined by the formulae:

$$
ScanRate= [[ \frac{lotsfree-freeMem}{lotsfree} ] *fastscan+ [ \frac{freeMem}{lotsfree}
*slowscan]]
$$

where freeMem mentions the current instance of free memory in quantity of pages. Intuitively intial values of scan rate is majorly contributed by slow scan, but as it nears the exhaustion fastscan’s value starts increasing and contributing towards scanrate.

CPU utilisation is interpolated with the scanrate by sporadic check  and comparison of the time taken to scan n pages ( called [PAGES_POLL_MASK](http://fxr.watson.org/fxr/source/common/os/vm_pageout.c?v=OPENSOLARIS;im=bigexcerpts#L134) hard coded with 1023)with the utilisation.

- CPU utilisation is calculated as follows:

1. min_pageout_ticks = $ \frac{hz*min-percent-cpu}{100}$
2. max_pageout_tick =  $ \frac{hz*max-percent-cpu}{100}$
3. ScalingFactor = max_pageout_tick  –  min_pageout_tick
4. [pageout_ticks](http://fxr.watson.org/fxr/source/common/os/vm_pageout.c?v=OPENSOLARIS;im=bigexcerpts#L579) = min_pageout_tick + $ \frac{lotsfree-FreeMem}{lotsfree} \{*} \{Scaling Factor}$

- Basically the fraction of memory available with respect to lotsfree is scaled to the ScalingFactor of CPU.
- Between lotsfree and desfree the memory is scanned 4 times/second. Once the memory reaches less than desfree its scanned 100 times/second.
- The throttleFree watermark suspends the allocation of pages that has the wait flag set till scanner brings the memory back above minsfree.


P.S. The same logic is implemented in 4.3BSD, but lotsfree is $ \frac{1}{4}$ of total memory, desfree is $ \frac{1}{2}$ of lotsfree and minfree is $ \frac{1}{2}$ of desfree.

**`Implementation in FreeBSD:`**

FreeBSD have a much simple scan policy compared to Solaris and 4.3BSD. The watermarks are replaced by a minimum percent of memory that needs to be maintained. If the memory reaches below this threshold then pageout daemon is started.

**Model.** FreeBSD divides the pages into 5 categories of pools, namely wired, active, inactive, cache and free. Pages that are unmovable from the physical memory are named wired, pages that is currently part of active execution is called active, pages that are not part of active execution is called inactive, pages that are not used anymore are cached and pages that are not allocated is in free list.

[Policy](http://fxr.watson.org/fxr/source/vm/vm_pageout.c#L1531). The system has 2 thresholds, namely minimum and target thresholds. The systems goal is to maintain a minimum threshold of pages in active, cache and freelist, Once the pages reach below the minimum threshold then the pageout daemon kickstarts and starts pushing the pages back to target threshold. By default the values are:

```js
Free + Cache( free_target) : 3.7% and 9%
inactive( Inactive_target): 0% and 4.5%
```

But these values are modifiable through user level interfaces.

**Procedure:**

1. Pageout daemon calculates the minimum required pages that are needed to get above the threshold.
2. Tries to acquire the required pages through 3 passes.
   1. page_shortage = free_target – (free_count + cache_count). Tries to acquire this target. If the achieved target does not satisfy the needed target, then we move to pass 2.
   2. page_shortage = ( free_target + inactive_target) – ( free_count + cache_count + inactive_count). Tries to acquire this target. If the achieved target does not satisfy the needed, then we move to pass 3.
   3. kickstart swap out daemon, still all memory is full, then kill the biggest memory consumed process.
3. After the 1st pass  check the time difference between previous pass and current pass. If the difference is more than lowmem_period(10), then trigger vm_lowmem event. This event tries to decrease many other registered cache sizes and reclaims the same.

**Implementation in Linux:**

Linux uses watermark amalgamated with demand based scanning, i.e., on allocating a page, the watermark is tested and if the current watermark before the allocation is less than the desired, then try reclaiming the zone directly and also start kswapd if required.  The scan amount is based on user defined parameter called [swappiness](https://elixir.bootlin.com/source/mm/vmscan.c?v=4.4#L1) . The reclamation policy is

**Model.** The Memory( A node in [NUMA](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms178144(v=sql.105)?redirectedfrom=MSDN) based machines) is basically divided into [zones](https://dl.acm.org/doi/10.1145/366199.366256). Each zone has High, Low and Min watermark. The ratio of the watermarks can be set by the  user. Scan rate is determined by variable called swappiness, default value is 60, but can also be set by user, maximum value recommended is 100.

![linuxscan](https://user-images.githubusercontent.com/79076337/110910507-71c35a00-8337-11eb-92db-01e29eb2a5de.png)

Fig2: Linux Scan Rate With Respect To Watermarks

**Policy.**

1. Before allocation,  [test for Min watermark](https://elixir.bootlin.com/linux/v4.9/source/mm/page_alloc.c#L2805) is made by adding the [allocation order count](https://en.wikipedia.org/wiki/Buddy_memory_allocation) of the request to free page count. If pages are below the low watermark and [GFP](https://elixir.bootlin.com/linux/latest/source/include/linux/gfp.h) is not either set to GFP_MEMALLOC, then direct reclamation of the zone is tried with the count of SWAP_CLUSTER_MAX, provided GFP_DIRECT_RECLAIM is set for the allocation request.
    - If the GFP is set with GFP_KSWAPD_RECLAIM, then kswapd is triggered to reclaim MAX( SWAP_CLUSTER_MAX, high watermark).
2. If the watermark falls below Min watermark then synchronous direct reclamation is attempted, i.e., any allocation request for pages with GFP_WAIT set is suspended on the [wait queue](https://elixir.bootlin.com/linux/v4.12/source/mm/vmscan.c#L2946). Concurrently kswapd tries to bring back the pages above high watermark.
3. In both the cases reclaim uses the following mechanism to scan the pages.
    - The variable named priority  scan_control decides on the portion of the zone to scan.
      - Scan_portion = [lru_type_within_zone](https://elixir.bootlin.com/source/include/linux/mmzone.h?v=4.4#L175)>> priority.
    - Anonyomous page priority(ap) = swappiness.
    - File page priority(fp) = 200 – ap.
    - Pressure is applied on each zone on file or anon lru formulae derived as:
          
         $$
         pressure \propto \frac{1}{\frac{Numberofpagesrotated}{Numberofpagesscanned}}
         $$
          
         $$
         pressure=respectivepriority(ap(or)fp)* \frac{Number of pages scanned}{Number of pages rotated}
         $$
          
    - respective pressure is either anonyomous or file page priority.
- Fraction per list Computation
    - Denominator = ap + fp .
    - Numerator = ap (or) fp.
    - Fraction to scan = $ \frac{Numerator}{Denominator}$
 
- Scaling: Fraction to scan is scaled to scan_portion
    - Scan_amount = Fraction to scan * Scan_portion.
    - Basically, scan portion is scaled to the fraction we derived.
- If the priority is zero and swappiness are non-zero, then Scan count is set to SCAN_EQUAL. This means the ignore the fraction calculation and directly derive the scan count based on the priority for each inactive list.


P.S. The statistical variables rotated and scanned is decomposed to half once it reaches a threshold.

intuitively, it is a heuristical thought that reasons with a belief that more the pages are rotated( [see Second Chance Algorithm](http://www.mathcs.emory.edu/~cheung/Courses/355/Syllabus/9-virtual-mem/SC-replace.html)) in a particular lru list then the possibility to reclaim pages from the same are less.  Finally, it’s worth to note that described policies are simplified to provide clarity on the core idea.


![scanning](https://user-images.githubusercontent.com/79076337/110758936-1465d500-8273-11eb-9d5c-5a31bf4d8ce6.png)

Detail Description Of Linux Scan count determination.

