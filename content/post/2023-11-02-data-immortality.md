+++
title = "Data Immortality"
date = "2023-11-02"
description = ""
tags = [
  "misc",
]
+++

# Some background

Having grown up with digital devices, I've produced a huge amount of data in the course of my life -- and much of it with at least some sentimental value in the form of photos but also various projects, notes, and such that I feel document my growth as a person. What I find myself contemplating is: how can I ensure that this data lasts my lifetime? Perhaps longer? 

I think that thinking critically about how we archive the data we care about is becoming more important as it becomes increasingly apparent that the web does infact loose things (e.g. the loss of [GeoCities](https://en.wikipedia.org/wiki/GeoCities) or YouTube beginning to delete older content). I think it's common to only think critically about where one keeps copies of data when migrating computers or perhaps when disaster strikes (e.g. a hard drive failure or computer crash). Developers or folks in industries where their data has high value (e.g. photographers, legal requirements, etc) may think about it more often -- but even so, data resiliancy is a hard problem.

In the past decade or so cloud based solutions have evolved which promise to help, most popular today are probably "backup and sync solutions" e.g.:

 * Google Drive
 * Dropbox
 * Amazon Cloud Drive
 * etc

And these approaches are vastly better than nothing -- they give us a great place to dump our files and they ensure those files are always up to date and easily accessible, but they don't constitute a backup on their own. Often these apps are vulnerable to propagating sync errors. If one deletes a file (or accidentally moves it outside a sync'd directory) on one computer, one is vulnerable to loosing that file on all devices. Perhaps more worryingly: if a file becomes corrupted or is accidentally overwritten, that corruption will immediately sync to all devices overwriting any copies of the good data.

A good backup strategy includes minimally:
 
 * Multiple copies
 * *And versioning*

The versioning property is something we can get in some sync providers (e.g. [Dropbox provides a 30 day version history](https://help.dropbox.com/delete-restore/version-history-overview) if you discover your corruption in that time), but more generally it's something we can address with a proper backup strategy.

# What works for me

My approach to versioned backups has been to select a single device as a "source of truth". This isn't essential but greatly simplifies the cognitive load of managing data sources for me and makes it easier to monitor backups from a single machine. I sync all of my devices and any important Cloud Storage I use to this device in a directory structure that I appreciate for backups e.g.:

 * /cloud-drives/
   * ./google-drive
   * ./dropbox
   * ./photos
   * etc
 * /computers/
   * ./laptop-mac-2023
   * ./desktop-windows-2022
   * etc

I then run backups on a daily schedule (this proves frequent enough for me but you may want something more frequent if you're worried about recovering revisions of your work) across all of those drives. Important here is that I want versioned backups that will allow me to roll back to any version of a file if I discover a recent copy is corrupt or has been accidentally deleted. To this end, I've experimented with a few pieces of backup software:

  * [restic](https://restic.net/) - OSS project, the backup option I'm deploying today. restic lacks good GUI options (it's typically used on CLI) 
  * [borg](https://borgbackup.readthedocs.io/en/stable/) - OSS project, very reliable and easy to trust but provides extremely limited storage options (only SSH targets).
  * [duplicati](https://www.duplicati.com/) - OSS project, offers a nice web UI but its archive format and implementation are reportedly not fantastically reliable, users report restores hanging and occasional failures. This is anecdotal but my end impression was that restic offers better reliability due to its simpler scope (cli only with a highly reliable format).
  * [Arq backup](https://www.arqbackup.com/) - commercial product, I've used this for backups on MacOS but eventually dropped the tool as I migrated my backups to a Linux system. It gets an honorable mention for having been fantastically reliable in my use and executing filesystem scans fantastically quickly thanks to its use of clever APFS snapshotting tricks to diff the filesystem.

At the end of the day, the tool I use is [restic](https://restic.net/) as I find myself quite comfortable with the CLI and able to trust it's reliablility based on user reviews and some reading of the tool's [storage specification which is well designed and documented](https://restic.readthedocs.io/en/v0.2.0/Design/).

# 3 2 1 Backup

Where do I send my data? A commonly echo'd sentiment in the datahoarding community is the practice of 3 2 1 backups ([backblaze has a great article on this](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/)). The short version is that you should keep three copies of your data 

 * keep two copies locally but on different media (e.g. physically separate devices with separate failure modes)
 * keep one copy offsite (e.g. think in a physically separate region that is subject to different natural disasters etc)

The idea of this approach is that the two local copies facilitate easy and fast replication and restore and the offsite backup gives you high levels of resiliance against the worst types of disasters (at the cost of a slow restore).

The destinations I choose to use are:

 * An offsite hard drive (not exactly local but these are only guidelines and I can easily drive and fetch this).
 * backblaze b2 (at $5/TB for today's pricing this is an economic target for my ~1TB of personal data).

# Erasure codes for great good (and the benefits of par2 archives)

Something I want to highlight in this article is that not all data needs be treated equally. For my most important files, which are typically the ones that are old and that I know to be immutable (e.g. old school projects, notes from past courses, photos from a given time period, etc), I like to go a step further to ensure they stay immutable and verifiably intact.

I think the practices discussed so far go beyond the level of "super good enough" for most of my data. And while I can be fairly confident that my data will stay safe at rest, I want to additionally ensure that data I choose to explicitly archive will last a lifetime. This means I need to be able to copy it between the various systems I'll own (or Cloud Providers I'll use) over the years and can be sure that, in doing so, nothing is lost or corrupted. Typically this isn't something we need to worry too much, but when working with data on a system without ECC ram or when considering the laws of large numbers it's likely that I'll eventually see some bit errors in my life as my data grows it's important to at least consider.

To this end, I've adopted par2 archives as a great archive format. par2 is, in my opinion, a criminally underpublicised format for parity encoded archive files. The archive format splits your data into a configurable number of chunks and and augments this with [Reed-Solomon erasure codes](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction). This scheme enables the integrity of the archive to be verified, but beyond this (and most importantly!) it doesn't leave you stuck once you've discovered your favorite photos are corrupted: the erasure encoding can be used to recompute lost or corrupted chunks of your dataset. 

When I finish a significant milestone I tend to gather up the data I care most about and merge it into a `.tar.gz` archive (again OSS formats that I expect to be decodable in perpituity) and I then augment this archive with `par2` erasure codes. I add recovery blocks that add ~10% to the size of the archive. This gives me a few nice properties:

 * I can verify the integrity of the archive (and recover from corruption) with `par2verify`
 * I can recover from data loss (e.g. a hard drive failure) with `par2repair`
 * I can recover from bit errors (e.g. when copying the archive between systems) with `par2repair`

# Wrapping up

Ultimately my goal with this article is mostly to summarize what I think are some good practices, at the end of the day however there's a lot of value in simplicity. I think the most important thing is to have a backup strategy that you're comfortable with, understand well, and that you can trust. I think the practices I've outlined here are a good starting point and hopefully you've found ideas you can borrow.

# Related Reading 

Some related articles I found of interest

 * [A linux user's experience copying 39 TB with `cp -r`](https://lists.gnu.org/archive/html/coreutils/2014-08/msg00012.html)