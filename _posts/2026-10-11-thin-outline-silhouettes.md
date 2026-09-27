---
layout: post
title: "ThinOutline: Silhouettes"
date: 2026-10-11 11:00:00 +0900
categories: [rendering, npr]
---

This post is not written yet. It will cover how the ThinOutline plugin
detects and reconstructs silhouettes, the edges where the depth jumps between
a foreground and a background: the depth-edge measure, which pixel records a
silhouette, why the records are kept on the foreground, and the depth and
velocity overwrite before the upscaler. It is part of the series on
[the ThinOutline plugin]({% post_url 2026-10-11-thin-outline-plugin %}).
