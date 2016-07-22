---
title: Correcting Malformed Data With Fuse+BPMS
banner: post-bg.png
permalink: correcting_data_with_fuse_and_bpms
date: 2015-10-30 00:00:00
tags:
- fuse
- karaf
- camel
- cxf
- jbpm
- bpms
- jboss
---

## The Problem

In many environments, it is common to ingest data as a batch process. This is usually accomplished by polling a directory (or FTP server) for files. The issue is that polling for files is a one-way communication. So if you get a file that you can't parse (or one that is invalid for any other reason), you have no way of communicating that back to the original sender. How do you handle the error? You likely can't just drop the data on the floor. That really only leaves one option and that is to correct the data and re-ingest it.
<!-- more -->

## The Solution

So we need to manually correct the data. We could create our own state machine to walk through the steps for correction. But we'd need to create a task queue, handle locking/timeouts, and create some UIs to allow the users to pull up the data and fix it. Also, we'll probably want to restrict access to some group (like "analyst"), and since the task will likely be long-running we'll want to create a set of UIs that allow operators to view the current state. All of this requires a significant amount of effort to complete (and is quite difficult to get right).

Or... we could just use BPMS which already does all of that for us.

Take a look at the [Malformed Data Fixer-Upper](https://github.com/joshdreagan/mdfu) project on GitHub. Follow the instructions in the [README.md](https://github.com/joshdreagan/mdfu/blob/master/README.md) file to get it up and running.

This example is a bit contrived, but it illustrates the general pattern that you might follow. Basically, Fuse is used to poll for files and validate them. If they are found to be invalid, Fuse will invoke a jBPM process with a _Human Task_ node. Once a human corrects the data, it will be passed back to Fuse to be validated/processed again. The interaction between Fuse and BPMS is done via SOAP/HTTP. So BPMS hosts a web service and Fuse hosts a callback web service. The following diagram shows the interaction in more detail (including what data is passed).

{% asset_img "async.png" [figure-1.1] %}
