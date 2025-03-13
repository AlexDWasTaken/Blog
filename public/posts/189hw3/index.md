# CS184 HW3 Writeup



# CS184 Homework 3 Write-Up

&gt; https://alexdwastaken.github.io/Blog/posts/189hw3/
&gt; **Group members: Xize Duan, Phoenix Ye**

## Overview

## Part1: Ray Generation and Scene Intersection

In this part, we generated rays according to the positions on the screen space, and did perspecive projections to transform these rays into camera space. We first generated pixel sample points on $(x, y) \in [0, w] \times [0, h]$ ,  and generated two randoms samples from $X, Y \sim\text{Unif}(0, 1)$, and eventually got random pixel samples $(x&#43;X, y&#43;Y)$. After that, we generate rays from the origin and test intersections along the rays. 

The first key is to distinguish the different space present in this case

- **Pixel space, image space, sensor space** and **camera space** are different. 
- By normalizing 2D **pixel space**, we get the **image spac**e. 
- The **image space** can be transformed into the **sensor space** by aliging the borders (since both are rectangular shapes). Specifically, (0, 0) goes to $(-\tan (0.5 \cdot hfov), -\tan(0.5.vhov))$ and $(1, 1)$ goes to $(\tan (0.5 \cdot hfov), \tan(0.5.vhov))$.
- The **sensor space** sits on $(0, 0, -1)$ of the **camera space** and is perpendicular to $z$ direction. Appending (-1) to a new dimension of the sensor space gives the position of the point in camera space.
- A ray is casted from the origin to the **camera space** sample point.

The second key is to cast rays, and properly set the ray up.

- Ray equation is $\vec o &#43; t \vec d$, and to ensure the consistency of each ray, $\vec d$ has to be normalized! (a lot of bugs encountered!)
- There are two fields `ray.min_t` `ray.max_t`. These are used to determine where the ray stars and when it ends. This caused a lot of bugs.
- `c2w` matrix converts camera space to world space. We do not have to invert this! (a lot of bugs encountered!)

To test the triangle intersection, we adopt the algorithm proposed in class. A short idea is to obtain the intersection of the ray and the plane, and calculate the berycentric coordinates with respect to the triangle. The coordinates can be calculated by doing cross products (since the cross product of two edges equals to the area of the triangle created by the two edges, and we can obtain berycentric coordinates by area). If the berycentric coordinates $\alpha, \beta$ and $\gamma$ are all positive, then the point lies in the triangle. An detail of implementation is that we can use dot product of components to avoid re-calculating same things.

To test the sphere intersection, we solved a quadratic function for `t1` and `t2`. The important thing is to **set `ray.min_t` `ray.max_t` after every valid intersection!; If not, we cannot detect the occlusion relation of an object correctly**. Becuase we didn&#39;t set the `t` values correctly, the balls won&#39;t render because triangle is tested last.

Here are some of the example renders.

| Example 1: Empty                                             | Example 2: Spheres                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![Part1_CBempty](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsPart1_CBempty.png) | ![Part1_CBspheres](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsPart1_CBspheres.png) |

 

## Part 2: Bounding Volume Hierarchy

BVH partitions the objects into different bounding boxes organized in a tree structure to accelerate ray-scene intersection, and it&#39;s very useful in ray tracing because it shrinks the complexity from $O(n)$ to $O(\log n)$ with a good policy.

**Construction**

In our implementation, bvh is seperated along the longest axis with median centroid. The first step is to determine whether the node is a leaf node by comparing the current `element_count` with `max_leaf_size`. If it already is, fill in `start` and `end` field and return directly.

If the node is not a leaf node, then we find its median and partition accordingly (with a functional programming style!)

```c&#43;&#43;
std::nth_element(start, mid, end, 
    [splitDim](const Primitive* a, const Primitive* b) {
      return a-&gt;get_bbox().centroid()[splitDim] &lt; b-&gt;get_bbox().centroid()[splitDim];
    });
```

This function partitions without sorting completely. After that, we create new BVH nodes and construct with the left half and right half, and link the new nodes to `l` and `r`.

A detail here is about the inclusiveness of &#39;start&#39; and &#39;end&#39;. In our implementation, we have to use

```c&#43;&#43;
auto leftStart = start;
auto leftEnd = mid;
auto rightStart = mid;
auto rightEnd = end;
```

where `mid` presents two times instead of one `mid` and one `mid&#43;&#43;`. We guess this is related to the implementation of iterators.

**Intersection**

Intersecting the bounding box is simple. We just record the last &#39;in&#39; time and first &#39;out&#39; time. If the &#39;in&#39; time is before &#39;out&#39; time, then a intersection is detected, and we can return accordingly.

One of the function only asks us to test whether there is a intersection; to solve that, write a simple recursion: if it is a leaf node, simply test every object inside; if not, return `has_intersection(ray, node-&gt;l) || has_intersection(ray, node-&gt;r)`. Another function asks us to record the hit position as well, in this case, just record the position in the `Intersection` object is fine.

We did not encounter any problem when realizing the intersection functions.

 **Results**

Here are some example images of large scenes with normal shading.

| Dragon                                                       | Lucy                                                         | Maxplanck                                                    |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![Part2_dragon](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsPart2_dragon.png) | ![Part2_lucy](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsPart2_lucy.png) | ![Part2_maxplanck](https://raw.githubusercontent.com/AlexDWasTaken/blog-pics/main/picsPart2_maxplanck.png) |

Here we analyze the rendering time with and without bvh. Here are the data:

| Scene | Primitives | Without BVH (Render) | With BVH (Render &#43; Build BVH) |
| ----- | ---------- | -------------------- | ----------------------------- |
| Gems  | 252        | 12.4182s             | 0.2220s&#43;0.0001s               |
| Empty | 12         | 0.5307s              | 0.1796s &#43; 0.0000s             |
| Cow   | 5856       | 80.9856s             | 0.0888s&#43;0.0036s               |
| Bunny | 28588      | Stuck                | 0.1170s&#43;0.3080s               |

The results demonstrate the significant impact of BVH acceleration on rendering performance. Without BVH, rendering times increase dramatically with scene complexity, with the Bunny scene failing to complete due to excessive computations. In contrast, BVH-based rendering consistently achieves great speedups, reducing rendering times by orders of magnitude. Even when factoring in BVH construction time, the total cost remains negligible. For example, in the Cow scene, rendering time drops from 80.99s to just 0.0888s, while BVH construction takes only 0.0036s. This highlights BVH&#39;s effectiveness in reducing ray intersection tests and optimizing traversal, making it indispensable for efficient ray tracing in complex scenes.

## Part 3: Direct Illumination



---

> Author:   
> URL: http://localhost:1313/Blog/posts/189hw3/  

