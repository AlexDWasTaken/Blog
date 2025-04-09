# CS184 HW4 Writeup



# Homework 4: Clothism

&gt; https://alexdwastaken.github.io/Blog/posts/189hw4/
&gt; **Group members: Xize Duan, Phoenix Ye**

## Overview
In this project, we build a realistic cloth simulation from the ground up. First, we model the cloth as a mass-spring system, discretizing it into point masses and connecting them with structural, shearing, and bending springs (Part 1). Then, we implement numerical integration to animate these masses over time (Part 2) and handle collisions with both external objects (Part 3) and the cloth itself (Part 4). Finally, we apply various shading techniques (Part 5), including diffuse, texture, and displacement mapping, to achieve visually appealing renderings of the simulated cloth in real-time.

## Part 1: Masses and Springs
In this part, we build a grid of point masses connected by three types of springs (structural, shearing, and bending). This establishes the foundation for our cloth simulation.

- **Structural**: connects each mass to its immediate neighbor(s) horizontally or vertically.
- **Shearing**: connects diagonally adjacent masses.
- **Bending**: connects every other mass (two steps away) horizontally or vertically.

We populate `point_masses` by creating `num_width_points * num_height_points` masses:
- If **horizontal orientation**, set `y = 1.0` and vary `(x,z)`.
- If **vertical orientation**, vary `(x,y)` and set a small random `z` offset (between `-1/1000` and `1/1000`).

Each point mass is marked `pinned` if its `(row, col)` index is in the pinned list.  
Once all masses are in point_masses, we create:
- **Structural springs**, connecting each mass to `(r, c-1)` &amp; `(r-1, c)`.
- **Shearing springs**, connecting diagonals `(r, c)` to `(r-1, c-1)` &amp; `(r-1, c&#43;1)`.
- **Bending springs**, connecting `(r, c)` to `(r, c-2)` &amp; `(r-2, c)`.

Below, we show the cloth in scene/pinned2.json from two viewing angles where you can clearly see the cloth wireframe to show the structure of our point masses and springs.

![Pinned 2 Angle 1](part1/part1.1.png)
Pinned 2 Angle 1 depicting structure of point masses

![Pinned 2 Angle 1](part1/part1.2.png)
Pinned 2 Angle 2 depicting structure of point masses (After cloth falls)

![Without any shearing constraints](part1/part1.3.png)
Without any shearing constraints

![With only shearing constraints](part1/part1.4.png)
With only shearing constraints

![With all constraints](part1/part1.5.png)
With all constraints

---

## Part 2: Simulation via Numerical Integration
In Part 2, we animate the cloth over time by computing the forces on each point mass and then integrating to update their positions. We use:
- **External forces**
- **Spring forces** (from structural, shearing, and bending springs)
- **Verlet integration** to determine new positions
- **Provot’s constraint** to limit spring over-stretching.

At the start of each timestep, we reset each point mass’s `forces` to zero. Then we:
1. **Add external forces**  
   - For each external acceleration in `external_accelerations`, we apply $ F = m \times a $ to every point mass.
2. **Add spring correction forces**  
   - For each spring (if enabled by the current cloth parameters), we use Hooke’s law:  
    ${F}_{spring} = -k (\|f{p_a} - f{p_b}\| - L_0)\,\hat{d}$
     where $\hat{d}$ is the unit direction from one mass to the other, $L_0$ is `rest_length`, and $k$ is the spring constant.  
   - Bending springs are typically weaker, so we multiply $k$ by a smaller factor.

We use **Verlet integration** to update positions. For each point mass:
1. If it’s **pinned**, skip all updates (remain fixed).  
2. Otherwise, given the current acceleration $f{a} = f{F}/m$, and damping factor $d$, we do:
    
    $$f{x}\_{new} = f{x}\_{current}
       &#43; (1 - d) (f{x}\_{current} - f{x}\_{previous})
       &#43; f{a}  \Delta t^2
    $$
3. Store `current_position` in `last_position` after updating.

We interpret the **damping** parameter as a percentage. For instance, if damping = 0.02 (2%), we use $1 - 0.02$ to slightly reduce the velocity term, simulating friction/energy loss.


After integration, each spring is checked to ensure it’s not stretched beyond **110%** of its rest length. If the distance between `pm_a` and `pm_b` exceeds $1.1 \times \text{restlength}$:
1. Compute the excess distance and direction.
2. Correct half of it on each mass, unless one is pinned (then apply the entire correction to the other).

This prevents the cloth from being unnaturally stretched.

To see the cloth’s behavior under different parameters, we tested:

## **Spring Constant**

We tested ks from 500 N/m to 50000N/m. At very low ks, the cloth is extremely loose and it strethes a lot and takes longer to settle into a shape. At very high ks, the cloth behaves more rigidly, barely deforms.

![KS = 500](part2/part2.1.png)
Cloth with ks = 500 $N/m$

![KS = 5000](part2/part2.2.png)
Cloth with ks = 5000 $N/m$

![KS = 5000](part2/part2.3.png)
Cloth with ks = 50000 $N/m$

## **Density**
We tested density from 1 $g/cm\^2$ to 50 $g/cm\^2$. With higher density, the cloth is heavier and falls faster under gravity. Springs may stretch more, leading to larger displacements before settling.  With lower density, it yields a lighter cloth, drifting more gently and bouncing around in collisions.

![Density = 1](part2/part2.4.png)
Cloth with a density of 1 $g/cm^2$

![Density = 15](part2/part2.5.png)
Cloth with a density of 15 $g/cm^2$

![Density = 50](part2/part2.6.png)
Cloth with a density of 50 $g/cm^2$

## **Damping**
We tested damping from 0.0% to 0.2%. With high damping, the cloth loses energy quickly, so it settles faster and doesn’t bounce much. With low damping, the cloth swings and bounces longer before coming to rest.

![Damping = 0](part2/part2.7.png)
Cloth with damping set at 0.0%

![Damping = 0.2](part2/part2.8.png)
Cloth with damping set at 0.2%

## **Pinned 4**
Below is a final resting shot of the cloth from `scene/pinned4.json`.

![pinned4](part2/part2.9.png)

---

## Part 3: Handling collisions with other objects

In Part 3, we extend our cloth simulation to collide with external primitives—in this case, **spheres** and **planes**. Whenever a point mass intersects or crosses these objects, we resolve the collision by “bumping” the point mass back outside the object’s surface. This prevents the cloth from passing through the object and allows it to drape realistically.

To handle collisions with a **sphere** of radius $r$ at center ${o}$:
1. **Detection**:  
After each time step, we check whether a point mass $f{p}$ is within the sphere: $\|{p} - {o}\| &lt; r$
2. **Resolution**:  
We find the intersection/tangent point on the sphere surface and we then compute a correction vector from the point mass’s `last_position` to ${t}$.  Finally, we update `position`. By doing so, the point mass ends up exactly on the sphere’s surface (slightly adjusted by friction). 

For Collision with **Planes**
1. **Detection**:
We check if the point mass moved from one side of the plane to the other between last and current position. If the dot products with the plane’s normal differ in sign, the point mass has crossed the plane.
2. **Resolution**:
We compute the intersection/tangent point $t$ where the segment crosses the plane.
Then we bump the point mass a small offset $ϵ$ above $t$ (in the plane’s normal direction) to avoid numerical issues.
Similar to the sphere, we use friction scaling $(1−f)$ when applying the correction vector.

## Results
Using scene/sphere.json, we pinned two corners of the cloth and dropped it onto the sphere. For Default ks = 5000, Cloth drapes over the sphere fairly stiffly but still shows some bending.
For Lower ks = 500, Cloth is much looser and sags more around the sphere. For Higher ks = 50000, Cloth appears almost rigid, hugging the sphere with minimal visible folds.

![collison ks = 500](part3/part3.1.png)
Collision with Sphere at ks = 500 N/m

![collison ks = 5000](part3/part3.2.png)
Collision with Sphere at ks = 5000 N/m

![collison ks = 50000](part3/part3.3.png)
Collision with Sphere at ks = 50000 N/m


![Shaded cloth on plane](part3/part3.4.png)
Shaded cloth on plane

## Part 4: Handling self-collisions
In Part 4, we address **self-collisions**, enabling the cloth to fold on itself realistically rather than clipping or intersecting. We implement **spatial hashing** to efficiently detect when point masses are too close, then apply corrective forces to keep them separated.

## 1. Spatial Hashing Approach

1. **`hash_position(pos)`**  
   - We partition the space into 3D boxes whose dimensions depend on `width`, `height`, and a chosen scale factor (commonly 3 × the spacing).
   - We truncate the coordinates of `pos` to find which box it belongs to and convert that box coordinate into a unique hash value (e.g., using a combination of x, y, z indices).

2. **`build_spatial_map()`**  
   - Clear the map from previous frames.
   - For every `PointMass` in `point_masses`, compute its hash key using `hash_position`.
   - Insert a pointer to that point mass into the corresponding map bucket (an array or vector of `PointMass*`).

3. **`self_collide(pm)`**  
   - For a given point mass `pm`, look up its hash bucket in the map.
   - For each candidate `other` in that bucket:
     - Skip if `other == pm`.
     - Check if the distance between `pm` and `other` is below `2 × thickness`.
       - If yes, compute the correction vector to separate `pm` from `other`.
       - Accumulate these correction vectors, then apply the average to `pm.position`, scaled by `(1.0 / simulation_steps)`.
         - This avoids sudden large jumps in a single time step.

4. **Integrating with `simulate()`**  
   - After we do Part 2’s integration and collisions with other objects, we:
     1. Call `build_spatial_map()`.
     2. For each `PointMass` (that is not pinned), call `self_collide(pm)`.
   - This ensures cloth points repel each other if they become too close.

Below are **three screenshots** of our cloth folding onto itself, captured at different stages of the simulation:

![Early Self-Collision](part4/part4.1.png)
Early Self-Collision

![Intermediate Folding](part4/part4.2.png)
Intermediate Folding

![Rest Collision](part4/part4.3.png)
Rest Collision


## 3. Parameter Experiments

To see how **density** and **ks** affect folding:

- **Increasing Density**:  When heavier cloth falls faster, creating deeper folds or stretches. The self-collision hashing must respond more aggressively as more mass collides in a shorter time.  
- **Decreasing ks**:  A looser cloth that sags more on itself, producing more pronounced folds. With self-collision, it keeps separated layers, though the folds are larger and floppier.
- **Increasing ks**:  A stiffer cloth that folds less dramatically. Self-collision corrections still prevent intersections, but the cloth more rigidly holds shape.

**Screenshots**  
1. **High Density**  
![High Density Fold](part4/part4.4.png)  
High Density Fold

2. **Low ks**  
![Low ks](part4/part4.5.png)  
Low ks

## Part 5: Shaders

In Part 5, we move from CPU-based rendering to **real-time GPU shaders**, leveraging **vertex shaders** (for geometry transforms) and **fragment shaders** (for per-fragment lighting and material effects). We implement several key shading models: **Diffuse Shading**, **Blinn-Phong Shading**, **Texture Mapping**, **Bump &amp; Displacement Mapping**, **Mirror (Environment Mapped) Reflections**.

**Explanation of shader program**: A shader program is a small, specialized piece of code that runs on the GPU. It is typically composed of two main stages: Vertex Shader – Operates on each vertex in a 3D model. It can transform the vertex from model space to screen space (e.g., via model, view, and projection matrices) and compute any per-vertex data that want to pass along (like normals, UV coordinates, etc.). The vertex shader’s outputs are then interpolated across the surface of the triangle or polygon.
Fragment (Pixel) Shader – Runs on each fragment, which can think of as each pixel-sized piece of the drawn surface. It uses the interpolated data from the vertex shader (position, normal, UV) and calculates the final color of that fragment. By splitting work this way, the GPU can operate in parallel: first it processes each vertex, then “fills in” the space between vertices by interpolating and running the fragment shader for every pixel covered by the triangle. This pipeline allows real-time rendering with advanced lighting and material effects at interactive frame rates.

## Blinn-Phong Shading Model
The Blinn-Phong shading model is a more refined variant of the basic Phong reflection model. It breaks down reflected light into three main components: Ambient – A constant term, simulating indirect light bouncing around the scene. Diffuse – The “Lambertian” reflection that depends on the angle between the surface normal and the light direction. It makes surfaces appear brighter when they face the light and darker as they turn away. Specular – A highlight that simulates the mirror-like reflection on shiny surfaces. In Blinn-Phong, we compute a half-vector $h$ halfway between the view direction and the light direction. The specular term depends on $max(0,n⋅h)^p$, where $p$ is a “shininess” exponent controlling how sharp or wide the highlight is.

Below are screenshots of ambient only, diffuse only, specular only and combined Blinn-Phong Model. 

![Ambient](part5/part5.1.png)  
Ambient Only (ka = 1, kd = 0, ks = 0)

![Diffuse](part5/part5.2.png)  
Diffuse Only (ka = 0, kd = 1, ks = 0)

![Specular](part5/part5.3.png)  
Specular Only (ka = 0, kd = 0, ks = 0)

![Combined](part5/part5.4.png)  
Combined (ka = 0.2, kd = 0.5, ks = 0)

## Custom Texture
![Custom](part5/part5.5.png)  
Custom Texture

## BUMP AND DISPLACEMENT MAPS
In Bump.frag, we perturb normals at the fragment level:
Compute height differences $deltaU$, $deltaV$ from the height map.
Construct a local “bump” normal in tangent space, then transform it by the TBN matrix back to world space.
Use this modified normal in a Blinn-Phong-like lighting equation.

Below are screenshots of bump mapping on the cloth and sphere

![Bump on cloth](part5/part5.6.png)  
Bump on cloth

![Bump on sphere](part5/part5.7.png)  
Bump on sphere

![Bump on sphere on cloth](part5/part5.8.png)  
Bump on sphere on cloth

Below is a screenshot of displacement mapping on the sphere

![Displacement on sphere and cloth](part5/part5.9.png)  
Displacement on sphere and cloth

![Displacement on cloth on sphere](part5/part5.10.png)  
Displacement on cloth on sphere

In Displacement.vert, we actually move each vertex along its normal by an amount read from the height map: $p&#39;=p&#43;n×[h(uv)×u_height_scaling]$.
This changes the geometry, so higher mesh resolution (-o 128 -a 128) reveals more detail.

![Bump 16 on cloth and sphere](part5/part5.11.png)  
Bump(-o16 -a16) on cloth and sphere

![Bump 128 on cloth and sphere](part5/part5.12.png)  
Bump(-o128 -a128) on cloth and sphere

![Displacement 16 on cloth and sphere](part5/part5.13.png)  
Displacement(-o16 -a16) on cloth and sphere

![Displacement 128 on cloth and sphere](part5/part5.14.png)  
Displacement(-o128 -a128) on cloth and sphere

Below is a screenshot of our mirror shader on the cloth and on the sphere.

![Mirror on cloth and sphere](part5/part5.15.png)  
Mirror on cloth and sphere

![Mirror on cloth on sphere](part5/part5.16.png)  
Mirror on cloth on sphere

## Appendix
In completing this assignment, we utilized ChatGPT O1 for language refinement, grammar correction, and structural adjustments. Additionally, we used it to improve the clarity of Markdown formatting to ensure better organization and readability.

Through this process, we gained a deeper understanding of Markdown formatting, including how to use headers, lists, and emphasis for clearer structuring. Moreover, we improved my ability to concisely express ideas, making our writing more precise and polished.

---

> Author:   
> URL: http://localhost:1313/Blog/posts/189hw4/  

