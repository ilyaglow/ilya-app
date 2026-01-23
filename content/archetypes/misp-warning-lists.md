---
title: Standalone MISP Warning Lists lookup web service
date: 2026-01-23
tags:
  - threatintel
  - blueteam
url: /blog/misp-warninglists-lookup
ShowReadingTime: true
---

## Background

Many years ago, I learned about [MISP Warning
Lists](https://www.circl.lu/doc/misp/warninglists/), which is an awesome and
actively maintained project. It aggregates many public datasets of well-known
indicators, including:

- Network subnets of popular Cloud providers
- Top websites lists
- Networks of many VPN providers
- Reserved IP subnets

And many more. Shout out to the CIRCL.lu folks, who are doing a really great job!

## The Challenge

Originally, these lists are intended as a supplement to the MISP framework, but
I always wanted to use them as a standalone tool. The main problem is that [the
official Python library,
PyMISPWarningLists](https://github.com/MISP/PyMISPWarningLists), loads all
lists into memory before querying. This makes it difficult to run queries on
resource-constrained environments, such as a 2GB VM or a Raspberry Pi.

## The Solution

For a long time, I wanted to implement a tool that imports these JSON files
into a simple SQLite database.

Today, I was experimenting with the Google Antigravity tool. Surprisingly, it
easily helped me implement a lightweight standalone web version of the MISP
Warning Lists database.

You can check it out here: <https://mwl.ilya.app>.
