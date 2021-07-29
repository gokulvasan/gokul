---
layout: post
title: Understanding Memory Management Watermarks in Unix
categories:
  - Blogs
last_modified_at: 2021-03-10T12:25:10-05:00
---

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
  
## Determining Watermark in Linux

Watermarks are statically determined waypoints over free pages in the main memory. 
Watermark determines the scan rate policy of the memory management.

Linux kernel as described in my [previous writing](https://gokulvasan.github.io/gokul/blog%20posts/2021/03/10/Empathising-mammoth-s-mentality.html) has 3 major watermarks, High, Min and Low. 
This blog is a brief view of the watermark calculation within the Linux kernel.


### 1. Use variable from admin window /proc/sys/vm/min_free_kbytes.

Converts kbytes unit to page unit.

pages_min = min_free_kbytes >> (PAGE_SHIFT – 10);

### 2. Calculate total managed pages in each zone except [highmem](https://linux-mm.org/HighMemory/)  zone( if it exists )

> 
``` js
for_each_zone(zone) {
       if(!is_highmem(zone))
            lowmem_pages += zone->managed_pages;
}
```


<u>For Each zone calculate the following</u>


### 3. Calculate fraction for each zone’s pages_min with respect to lowmem_pages.
**tmp**= ( pages_min / lowmem_pages ) * zone->managed_pages.
- Managed_pages are total available page count in a zone that could be used for allocation.

### 4. Calculate WMARK_MIN for HIGHMEM using following calculation:
min_pages = zone->managed_pages  /  1024

min_pages = **MIN( MAX**(min_pages, SWAP_CLUSTER_MAX), 128)

### 5. Else, simply use the tmp variable as min_pages watermark count.

### 6. Calculate the distance for the rest of the watermarks.
Its the fraction of managed_pages by 10000, scaled by

**watermark_scale_factor = 10**

fraction = ( zone->managed_pages / 10000 )  * watermark_scale_factor.

distance = **MAX**( tmp >> 2,  fraction);

### 7. Assign watermarks
zone->watermark[WMARK_MIN] = min_pages;

zone->watermark[WMARK_LOW] = min_pages + distance;

zone->watermark[WMARK_HIGH] = min_pages + distance * 2;




