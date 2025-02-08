# CS184 HW1 Writeup


# Write up for HW1

## Overview

## Question 1

To rasterize a triangle, what we fisrt did is to perform a point-in-triangle test. The test is accomplished by performing a cross product $(p_1-p_0)\times (p_2-p_1) \cdot k$. If the result is negative, then the winding order is clockwise, otherwise counterclockwise. The final result will never be zero for a well-defined triangle.
We fisrt assume a counter-clockwise winding order.
Then the following is the condition for point inside triangle:

$$
- (y1 - y0) (x - x0) &#43; (x1 - x0) (y - y0) \geq 0 \\
- (y2 - y1) (x - x1) &#43; (x2 - x1) (y - y1) \geq 0 \\
- (y0 - y2) (x - x2) &#43; (x0 - x2) (y - y2) \geq 0
$$

If the winding order is clockwise, simply change the direction of the signs.

The next thing we do is to enumerate across every row in the bounding box of the triangle, and color the pixel if the point inside the triangle. What we did to optimize this operation is that we keep track whether we have left the triangle. If we have left the triangle, then the rest of the picture on this row will never skip to the next row, so it is safe to escape to the next row.

We timed the rendering time of the dragon picture. The naive method which we enumerated every pixel in the bounding box took around 0.0078 second on average, and the one which we kept track of whether we have left the triangle took 0.0067 second on average.

# Question 2

In summary, the supersampling algorithm improves image quality by working in a higher-resolution space and then averaging down to the final image. The modifications sums up to three steps—using a larger sample buffer, scaling coordinates, and adding a resolve step.

We changed the size of sample buffer each time. One subtle difference is that if the total image is supersampled by $n$, then its coordinate is only scaled by $\sqrt{n}$, which is crucial in correctly sample everything. During the sampling process, instead of marking an entire pixel as either inside or outside, we check multiple subpixel sample locations especially on edge places.. For example, when we supersample by a factor of 4, the coordinates are scaled by a factor of 2. We also modified the function that&#39;s used to erase and rewrite sample buffer by a factor of $n$.

For each triangle, we write this into a bigger version of the sample buffer. When finally writing the sampled image onto the frame buffer, we take average of the corresponding pixel values to resolve to the final value. By capturing more detail about how a primitive covers a pixel, supersampling reduces the jagged edges (aliasing) that would otherwise appear when a pixel’s single sample is forced into a binary decision.






---

> Author: DY  
> URL: https://example.org/posts/eeccd09/  

