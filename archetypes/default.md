---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true

# Preview text shown on the home/list pages + used for SEO and social cards.
# Leave blank to auto-use the first ~30 words. `description` feeds meta/OG tags.
summary: ""
description: ""

# Taxonomies — keep consistent with existing posts.
# Convention: Title Case, no leading spaces, acronyms UPPERCASE (CTF, PE, DLL, DNS, API).
# Categories in use: Guide, Writeups
categories:
  - Guide
tags:
  - Malware

# Multi-part post? Use the same series name in each part and order them.
# series:
#   - Keyloggers
# series_order: 1
---

<!--
Tip: drop an image named `feature.png` (or feature.jpg/webp) in this post's
folder and it becomes the card thumbnail + hero automatically. See WRITING-POSTS.md.
-->
