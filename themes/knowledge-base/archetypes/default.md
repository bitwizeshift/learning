+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
draft = true
# Stable id -> becomes a #anchor and a badge prefixed on the page and in listings.
id = ''
# A post has EXACTLY ONE type (declared as a single-element list).
types = ['best-practice']
# Taxonomy tags: what the post relates to / contains. Drive badges + list pages.
# One or more toolchains and one or more kinds.
toolchains = []
kinds = []
toc = true

# Focus: what this post is a primary reference FOR. Drives which merged pages
# (e.g. "Testing Best Practices", "C++ Best Practices") include it. Keys are the
# plural taxonomy/data names. Every focus term should also appear in the matching
# taxonomy field above so the post stays on its list pages.
[focus]
  # kinds = ['testing']
  # toolchains = ['cpp']
+++
