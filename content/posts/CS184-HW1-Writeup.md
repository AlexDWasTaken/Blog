---
title: CS184 HW1 Writeup
subtitle:
date: 2025-02-08T00:05:27-08:00
slug: eeccd09
draft: false
author:
  name: Xize Duan
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - CS184
categories:
  - CS184
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: true
lightgallery: false
password:
message:
repost:
  enable: true
  url:

---

# Write up for HW1

> **Group members: Xize Duan, Phoenix Ye.**
> **Link to the page itself: [https://alexdwastaken.github.io/Blog/posts/eeccd09/index.html](https://alexdwastaken.github.io/Blog/posts/eeccd09/index.html)**

# Overview

Our project implemented core rasterization and texture mapping techniques. 
- For triangle rasterization, we used cross-product-based point-in-triangle tests with row-based optimization, improving render time from 0.0078s to 0.0067s. 
- Supersampling was implemented by scaling coordinates by the square root of the sample rate and averaging down. We enabled shape manipulation through matrix-based transforms, demonstrated with animated exercising figures. 
- Barycentric coordinates are calculated from distance ratios and can be intepreted as area ratios. It can be used to interpolate colors. 
- For texture mapping, we implemented both nearest neighbor (fast but jagged) and bilinear sampling (smoother but slower). 
- Level sampling with mipmaps addressed aliasing at different scales. Key challenges included optimizing rasterization, correct coordinate scaling for supersampling, and handling texture mapping edge cases.

<div style="page-break-after: always;"></div>

# Question 1

To rasterize a triangle, what we fisrt did is to perform a point-in-triangle test. The test is accomplished by performing a cross product $(p_1-p_0)\times (p_2-p_1) \cdot k$. If the result is negative, then the winding order is clockwise, otherwise counterclockwise. The final result will never be zero for a well-defined triangle.
We fisrt assume a counter-clockwise winding order.
Then the following is the condition for point inside triangle:

$$
(y1 - y0) (x - x0) + (x1 - x0) (y - y0) \geq 0 \newline
(y2 - y1) (x - x1) + (x2 - x1) (y - y1) \geq 0 \newline
(y0 - y2) (x - x2) + (x0 - x2) (y - y2) \geq 0
$$

If the winding order is clockwise, simply change the direction of the signs.

The next thing we do is to enumerate across every row in the bounding box of the triangle, and color the pixel if the point inside the triangle. What we did to optimize this operation is that we keep track whether we have left the triangle. If we have left the triangle, then the rest of the picture on this row will never skip to the next row, so it is safe to escape to the next row.

We timed the rendering time of the dragon picture. The naive method which we enumerated every pixel in the bounding box took around 0.0078 second on average, and the one which we kept track of whether we have left the triangle took 0.0067 second on average.

Here is a png screenshot of `basic/test4.svg` with the default viewing parameters and with the pixel inspector centered on an interesting part of the scene.

<img src="https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsScreenshot%202025-02-16%20at%2012.38.37%E2%80%AFAM.png" alt="Screenshot 2025-02-16 at 12.38.37 AM" style="zoom:50%;" />

<div style="page-break-after: always;"></div>

# Question 2

In summary, the supersampling algorithm improves image quality by working in a higher-resolution space and then averaging down to the final image. The modifications sums up to three steps—using a larger sample buffer, scaling coordinates, and adding a resolve step.

We changed the size of sample buffer each time. One subtle difference is that if the total image is supersampled by $n$, then its coordinate is only scaled by $\sqrt{n}$, which is crucial in correctly sample everything. During the sampling process, instead of marking an entire pixel as either inside or outside, we check multiple subpixel sample locations especially on edge places.. For example, when we supersample by a factor of 4, the coordinates are scaled by a factor of 2. We also modified the function that's used to erase and rewrite sample buffer by a factor of $n$.

For each triangle, we write this into a bigger version of the sample buffer. When finally writing the sampled image onto the frame buffer, we take average of the corresponding pixel values to resolve to the final value. By capturing more detail about how a primitive covers a pixel, supersampling reduces the jagged edges (aliasing) that would otherwise appear when a pixel’s single sample is forced into a binary decision.

Below are two examples of the rendering results of `basic/test4.svg` under different parameters.

| Samplerate = 1                                               | Samplerate = 4                                               | Samplerate = 16                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![Q2_1](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ2_1.png) | ![Q2_4](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ2_4.png) | ![Q2_16](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ2_16.png) |

| Samplerate = 1                                               | Samplerate = 4                                               | Samplerate = 16                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![Q2_1-2](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ2_1-2.png) | ![Q2_4-2](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ2_4-2.png) | ![Q2_16-2](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ2_16-2.png) |

We observed that, as the sample rate goes up, the thin angle becomes less sharp but more continuous, and better represents the "real" triangle. This is because a higher sample rate increases the resolution of the sampled data, allowing for a more accurate approximation of the true shape of the triangle. At lower sample rates, the sampled points are more sparsely distributed, leading to a jagged or stepped representation of sharp angles. As the sample rate increases, more points are captured along the edges, smoothing out the transitions and reducing aliasing effects, which results in a more continuous and precise representation of the original shape.



<div style="page-break-after: always;"></div>

# Question 3

We implemented the transform functions by generating corresponding transform matrix. We altered the svg so that the little people is now doing exercies by stretching their arms.

![Screenshot 2025-02-15 at 10.29.43 PM](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsScreenshot%202025-02-15%20at%2010.29.43%E2%80%AFPM.png)

The SVG file can be downloaded here. [Q3_svg](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/my_robot.svg)

![Q3_svg](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/my_robot.svg)

<div style="page-break-after: always;"></div>

# Question 4

## Explanation

Barycentric coordinates are a way of describing a point's position inside a triangle (or a higher-dimensional simplex) using weighted averages of the triangle’s vertices. Every point inside the triangle is represented by three numbers (weights), corresponding to the triangle's vertices A, B, C. The berycentric coordinates can be intepreted as the ratio of the area of a small triangles with respect to the area of the full triangle. The closer P is to a vertex, the larger the corresponding subtriangle’s area, and thus the larger the associated barycentric coordinate. See the picture below for a detailed illustration.

<img src="https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsimage-20250215224622798.png" alt="image-20250215224622798" style="zoom:33%;" />



Berycentric coordinates can effectively be used for intropolation. For a point with coordinates ($\lambda_A, \lambda_B, \lambda_C$), its color can be intepreted as ($\lambda_A \cdot c_A+ \lambda_B \cdot c_B +\lambda_C\cdot c_C$). See the picture below for a illustration of a triangle interpolated with berycentric coordinates. In real implementation though, we calculate the coordinates by calculating distance, which is more efficient.

<img src="https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ4_demonestration.png" alt="Q4_demonestration" style="zoom:50%;" />

## Screenshot

Here is a png screenshot of `svg/basic/test7.svg` with default viewing parameters and sample rate 1.

<img src="https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ4_color_wheel.jpg" alt="Q4_color_wheel" style="zoom:50%;" />

<div style="page-break-after: always;"></div>

# Question 5

## Explanation

Pixel sampling is the process of determining the color of a pixel by fetching texture data from a texture map. When implementing texture mapping, I used pixel sampling to fetch the correct texel color based on texture coordinates (u,v), which are continuous values ranging from 0 to 1. The range of u, v needs to be converted to the actual coordinates of the mipmap, or otherwise we will be constantly sampling the (0, 0) pixel. My inplementation can be factorized into two steps: Frrstly, we mapped the (u, v) coordinates to the texture image’s pixel grid. After that, colors can be effectively fetched from the uv coordinates.

Now we explain the difference of the two sample methods. The nearest method simply rounds the non-integer coordinates to the nearest integer values, while bilinear method interpolates color from the four nearest pixels. These can be formulated as follows:

**Nearest Neighbor Sampling**  
- Convert (u, v) to texture space:  
  $$
  x = \text{round}(u \times \text{texture width})
  $$
  $$
  y = \text{round}(v \times \text{texture height})
  $$
- Fetch the texel at $(x, y)$.  

**Bilinear Sampling**  

- Compute the surrounding texel indices and their fractional offsets:  
  $$
  x = u \times \text{texture width}, \quad y = v \times \text{texture height}
  $$
  $$
  x_0 = \lfloor x \rfloor, \quad x_1 = x_0 + 1
  $$
  $$
  y_0 = \lfloor y \rfloor, \quad y_1 = y_0 + 1
  $$
  
- Retrieve the four texels at $(x_0, y_0)$, $(x_0, y_1)$, $(x_1, y_0)$, $(x_1, y_1)$.  

- Perform linear interpolation first along x, then along y.  

## Comparison

Here are the example renderings of different methods. Notice the gradient int the transition, how bilinear is more smooth, and how supersampling further enhances the render results. Also, nearest  sampling renders a little faster.

|            | Nearest                                                      | Bilinear                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1 Sample   | ![Q5_N_1](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ5_N_1.png) | ![Q5_Bi_1](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ5_Bi_1.png) |
| 16 Samples | ![Q5_N_16](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ5_N_16.png) | ![Q5_Bi_16](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ5_Bi_16.png) |

In the following example, bilinear clearly defeats nearest. The color of nearest sampling has visible jaggies, while the bilinear methods is way more smooth.

| Bilinear                                                     | Nearest                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![Q5_Comparison_Bi](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ5_Comparison_Bi.png) | ![Q5_Comparison_N](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsQ5_Comparison_N.png) |

## Relative Differences

The difference of the two methods can be summed as follows:

- Nearest sampling is fast but produces rough, pixelated results.  
- Bilinear sampling smooths textures but requires more computation. 

For better quality, bilinear sampling is preferred, especially when textures are scaled. 

There will be a large difference when textures are magnified or skewed, in which nearest sampling will result in jaggies; also when there are high-frequency details, nearest will cause aliasing and bilinear will be a little better. In general, bilinear filtering improves visual quality in cases of scaling and transformation, while nearest sampling retains sharp texel boundaries but can lead to noticeable artifacts.


<div style="page-break-after: always;"></div>

# Question 6

## Explanation

Level sampling is a technique used in texture mapping to determine the appropriate level of detail (LOD) for a texture based on the screen-space size of the mapped surface. Instead of directly using a high-resolution texture for all cases (results in aliasing), level sampling selects a lower-resolution version of the texture when the object is far away or viewed at a sharp angle. This helps maintain visual quality and performance because it is equivalent to pre-apply a low-resolution box filter.

The idea is to precompute multiple levels of a texture where each level is a downsampled version of the original texture. During rendering, the appropriate level is chosen based on how much the texture is being scaled or minified.

Our realization can be broken down into three steps. Firstly, compute the correspodning mipmaps. Then, according to the transformation and scaling factor, chooes the mipmap to sample from. At last, we fetch the color from the correponding mipmaps.

## Comparison

Here’s a comparison of the tradeoffs between different sampling methods.

| **Technique**  | **Speed**                                                    | **Memory Usage**                            | **Antialiasing Power**                                   |
| -------------- | ------------------------------------------------------------ | ------------------------------------------- | -------------------------------------------------------- |
| Pixel Sampling | Fast (samples one texture level at a time)                   | Low (only needs one texture fetch)          | Low (prone to aliasing, especially at minified scales)   |
| Level Sampling | Moderate (needs to compute LOD and may interpolate between MIP levels) | Moderate (requires multiple MIP map levels) | Good (reduces aliasing caused by minification)           |
| Supersampling  | Slow (increases rendering cost significantly)                | High (multiple texture fetches per pixel)   | Excellent (best at reducing jagged edges and flickering) |

To sum up, Pixel Sampling is the fastest and most memory-efficient but can introduce aliasing, especially at extreme minifications or magnifications; Level Sampling balances performance and quality by using precomputed MIP maps, reducing aliasing without a massive performance hit, and Supersampling provides the highest quality but at the cost of significant performance and memory usage.

## Demonestration

Here is a set of comparison of for sampling methods where we mainly focused on antialiasing power with pixel inspector on the upper-right corner. (Photo credit: Armand Khoury from Unsplash).


|           | P_NEAREST                                                    | P_LINEAR                                                     |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| L_ZERO    | ![Screenshot 2025-02-16 at 12.01.07 AM](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsScreenshot%202025-02-16%20at%2012.01.07%E2%80%AFAM.png) | ![Screenshot 2025-02-16 at 12.01.11 AM](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsScreenshot%202025-02-16%20at%2012.01.11%E2%80%AFAM.png) |
| L_NEAREST | ![Screenshot 2025-02-16 at 12.01.00 AM](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsScreenshot%202025-02-16%20at%2012.01.00%E2%80%AFAM.png) | ![Screenshot 2025-02-16 at 12.00.49 AM](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsScreenshot%202025-02-16%20at%2012.00.49%E2%80%AFAM.png) |
