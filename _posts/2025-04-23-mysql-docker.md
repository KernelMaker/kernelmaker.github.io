---
layout: post
title: Is MySQL Ready for Running in Containers?
---

**MySQL 9.3.0** was [released](https://dev.mysql.com/doc/relnotes/mysql/9.3/en/news-9-3-0.html) last week (April 15, 2025), and one particular update caught my attention:

> “InnoDB now supports container-aware resource allocation, allowing it to adhere to the restrictions imposed by the container.”

Curious about this, I went through the related patch and did a quick review of what MySQL has done to support container environments so far. I’d like to share a few thoughts on it.

In my opinion, for MySQL to be **fully container-friendly**, it needs to support the following three capabilities in order of importance:

### ✅ Step 1: Auto-detect container resource limits at startup

On startup, MySQL should be able to automatically detect CPU and memory limits configured for the container and use them to generate sensible default values for critical parameters—without requiring manual intervention. Otherwise, users must manually tune parameters to match each container’s resources, which is inefficient and error-prone.

Some of the key parameters that should be auto-tuned include:

- **CPU-related:**
  - `innodb_buffer_pool_instances`
  - `innodb_page_cleaners`
  - `innodb_purge_threads`
  - `innodb_read_io_threads`
  - `innodb_parallel_read_threads`
  - `innodb_log_writer_threads`
  - `innodb_redo_log_capacity`
- **Memory-related:**
  - `innodb_buffer_pool_size`
  - `innodb_buffer_pool_instances`
  - `temptable_max_ram`

### ✅ Step 2: Support online adjustment of these parameters

This is absolutely critical. If the container's CPU or memory resources are updated at runtime, but MySQL requires a restart to apply updated parameters, that causes unnecessary downtime and potentially expensive data reloading—an unacceptable tradeoff in many production environments.

### ✅ Step 3: Allow MySQL to dynamically respond to container resource changes

If Steps 1 and 2 are supported, then adapting to container changes becomes feasible, though still a bit manual:

1. Dynamically adjust container CPU and memory from the outside.
2. Recalculate the relevant MySQL parameters and apply them using the online configuration capabilities from Step 2.

This workflow is workable—but still tedious. It would be much more elegant if MySQL could:

1. Detect new CPU and memory limits in real time.
2. Automatically adjust internal configuration accordingly **without user input(or by simply passing in the latest CPU and memory info)**.


### So… where does MySQL stand today?

Unfortunately, not quite there yet.

<img src="/public/images/2025-04-23/1.png" alt="image-1"/>

- As of **MySQL 9.3.0**, Step 1 is now properly supported.
- Step 2 is **partially** implemented—some parameters can be modified at runtime, while others still require a restart.
- Step 3 is **not supported at all**.

Looking at the progress over time, it’s clear that development in this area has been relatively slow.

<img src="/public/images/2025-04-23/2.png" alt="image-2"/> 

To be honest, most of the work to support containers isn't very difficult—probably the most technically challenging piece is online resizing of `innodb_buffer_pool_size`, but MySQL has supported that since **version 5.7.5**!

The rest should be manageable in a minor release cycle or two.


### ✨ 

I hope the MySQL team can recapture the spirit of the 5.6/5.7 era and deliver some bold innovations in upcoming releases—especially in the Innovation Track. There's a lot of potential to make MySQL significantly better—with improved performance, smarter features...

--- *"Make MySQL Great Again..."* :)