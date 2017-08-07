---
layout: post
title: "Coding Structure Control in libx264"
date: 2017-08-05
---

In this post, we will first see how the sliding window memory control is implemented in x264. Then, we will modify this default implementation to generate hierarchical-P streams with arbitrary number of temporal layers.

### Frame buffers

As previously mentioned [here]({{ site.baseurl }}{% post_url 2016-07-05-x264 %}), frame pointers and other variables are stored in the `frames` data member of `x264_t`. Essentially, the frame pointers `x264_frame_t*` are stored in the following arrays that act like deques.

1. **`h->frames.current[]`**: array of frame pointers (whose types have been decided) to be encoded.
2. **`h->frames.unused[]`**: table of frame pointers for unused frames. The second row holds the unused reconstructed frames.
3. **`h->frames.reference[]`**: array of frame pointers of length `X264_REF_MAX+2` (2 sentinels). May contain both past and future frames. `X264_REF_MAX` is `#defined` to be equal to 16. In the deafult implementation, the most recent reconstructed reference frame sits at the tail, while the oldest sits at index 0.

These lists are manipulated through functions defined in `common/frame.c`. Among them, the following are quite useful.

1. **`x264_frame_push()`**: inserts a frame pointer at the **tail** of a given array.
2. **`x264_frame_push_unused()`**: given a frame pointer, decrements its reference count, and inserts it at the **tail** of the `h->frames.unused[1]` array if the reference count drops to zero.
3. **`x264_frame_pop()`**: pops and returns the frame pointer at the **tail** of a given array.
4. **`x264_frame_pop_unused()`**: if the array is non-empty, pops and returns the frame pointer at the **tail** of the `h->frames.unused[1]`. Otherwise, it returns a pointer to a brand new frame.
5. **`x264_frame_shift()`**: pops and returns the frame pointer at the **head** of a given array. The rest of the array is shifted left.

### Default coding structure (IPPP)

In libx264, the default IPPP coding structure is implemented inside the `x264_reference_update()` function located inside `encoder/encode.c`. In the code snippet below, notice how the last reconstructed frame is placed at the tail of the `h->frames.reference` array. If this array contains more than `h->sps->i_num_ref_frames` frame pointers, the frame pointer at its head is popped, and then inserted at the tail of the `h->frames.unused[1]` array. Finally, the frame pointer at the tail of the unused array is popped to be used as the next reconstructed frame.

{% highlight c %}
/* move frame in the buffer */
x264_frame_push( h->frames.reference, h->fdec );
if( h->frames.reference[h->sps->i_num_ref_frames] )
	x264_frame_push_unused( h, x264_frame_shift( h->frames.reference ) );
h->fdec = x264_frame_pop_unused( h, 1 );
{% endhighlight %}