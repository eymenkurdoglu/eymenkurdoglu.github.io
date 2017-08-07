---
layout: post
title: "Coding Structure Control in libx264"
date: 2017-08-05
---

In this post, we will first see how the sliding window memory control is implmented in x264. Then, we will modify this default implementation to generate hierarchical-P streams with arbitrary number of temporal layers.

### Frame Arrays

As previously mentioned [here]({{ site.baseurl }}{% post_url 2016-07-05-x264 %}), frame pointers and other variables are stored in the `frames` data member of `x264_t`. Essentially, the frame pointers `x264_frame_t*` are stored in the following arrays.

1. **`h->frames.current[]`**: array of frame pointers (whose types have been decided) to be encoded.
2. **`h->frames.unused[]`**: table of frame pointers for unused frames. The second row holds the decoded frames.
3. **`h->frames.reference[]`**: array of frame pointers of length `X264_REF_MAX+2` (2 sentinels). May contain both past and future frames. `X264_REF_MAX` is `#defined` to be equal to 16.

