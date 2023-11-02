+++
title = "Data Immortality"
date = "2021-08-01"
description = ""
tags = [
  "misc",
]
+++

# Some background 
As a child of the "digital age" so to speak, something I find myself thinking about more and more is: how can I make my files last my lifetime? Perhaps longer?I think this is a problem that is presenting itself to more and more people as the internet and our technology reaches a certain age. The general public seems to be growing more aware of the problem of content on the internet becoming delisted (e.g. think low view count YouTube videos) or becoming corrupted over time (e.g. each time you migrate your photo library to the latest and greatest format it gets a bit more compressed or perhaps more corrupted).

# 3 2 1 Backup

A commonly echo'd sentiment in the datahoarding community is the practice of 3 2 1 backups ([backblaze has a great article on this](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/). The short version is that you should keep three copies of your data 

 * keep two copies locally but on different media (e.g. physically separate devices with separate failure modes)
 * keep one copy offsite (e.g. think in a physically separate region that is subject to different natural disasters etc)

The idea of this approach is that the two local copies facilitate easy and fast replication and restore and the offsite backup gives you high levels of resiliance against the worst types of disasters (at the cost of a slow restore). Beyond this, however, a backup is only as good as your trust in the data being stored. If your data becomes corrupted either at rest or in processing, you'll be synchronizing and backing up junk. 

# par2 Archives

How can I ensure my most important data stays immutable overtime and is resiliant to corruption (e.g. when copying from an old computer to a new one or on a faulty hard drive). Enter: par2.

par2 is, in my opinion, a criminally underpublicised format for parity encoded archive files. The archive format splits your data into a configurable number of chunks and and augments this with [Reed-Solomon erasure codes](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction). This scheme enables the integrity of the archive to be verified, but beyond this (and most importantly!) it doesn't leave you stuck once you've discovered your favorite photos are corrupted: the erasure encoding can be used to recompute lost or corrupted chunks of your dataset. 
