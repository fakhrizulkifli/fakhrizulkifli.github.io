---
layout: post
title: "BSD Hypervisor - Theoretical Stack-based Buffer Overflow"
comments: true
description: "BSD Hypervisor - Theoretical Stack-based Buffer Overflow"
keywords: "bhyve, stack-based, buffer overflow, hypervisor"
---

On Jun 24, a new PCI HDAudio device model from the Google Summer of Code 2016 project was added into the BSD Hypervisor (bhyve) as part of its sound card emulation. I took a chance to look at the source code and found a trivial stack-based buffer overflow but unfortunately it is not exploitable in practice due to the limitation of bhyve's arguments length. However, the code is patched just in case it is still going to be an issue in the future.

---
References:
1. https://wiki.freebsd.org/SummerOfCode2016/HDAudioEmulationForBhyve
2. https://svnweb.freebsd.org/base?view=revision&revision=349335
3. https://reviews.freebsd.org/rS349385