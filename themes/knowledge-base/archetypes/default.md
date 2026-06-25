+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
draft = true
# Stable id -> becomes a #anchor and a badge prefixed on the page and in listings.
# When set, also mirror it into `slug` below so the URL is /<focus>/<id>/.
id = ''
# slug = ''   # set equal to `id` for an id-based URL; omit to use the filename.
# A post has EXACTLY ONE resource (declared as a single-element list).
resources = ['best-practice']
# Taxonomy tags: what the post relates to / contains. Drive badges + list pages.
# One or more toolchains and one or more concepts.
toolchains = []
concepts = []
toc = true
# FOCUS: a post's focus is its folder. Place this file under
# content/posts/<focus-term>/ (e.g. content/posts/cpp/ or content/posts/testing/)
# to set which merged pages include it (e.g. "C++ Best Practices") and to give it
# a /<focus-term>/<slug>/ URL. The focus term should also appear in the matching
# taxonomy field above so the post stays on its term list pages.
+++
