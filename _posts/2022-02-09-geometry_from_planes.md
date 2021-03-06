---
title:  "Journey into CSG: Generating 3d geometry from a list of planes"
#date:   2020-02-07
comments: true
math: true
---

Recently I started a new project of making a real-time CSG editor. Or, at least that's the very first starting point for the project. I didn't exactly know how CSG worked before, so I looked into [some articles](http://sandervanrossen.blogspot.com/2009/12/realtime-csg-part-1.html). I learned some very cool things, such as what a half-edge data structure is. Previously, it had been a big mystery to me how you would actually implement a 3d modeling operation such as extrusion, bevel, or inserting edge loops, but this very tiny idea really opened things up for me. It's all so clear now! You should first [read about it](https://www.flipcode.com/archives/The_Half-Edge_Data_Structure.shtml) if you're not familiar with it. You can easily convert a half-edge mesh to a traditional triangle-mesh that you'd feed to the GPU for instance.

<!--more-->

<p class="message-info">
NOTE: Instead of storing half-edges separately, I'm bundling them together in a single Edge - struct that stores both of them. This way we require one less pointer to worry about. Also, if we want to store data that is shared per each edge, such as sharpness/smoothness or UV-seam flags, it's stored right there in one place. A half-edge index is the edge index in the first 31 bits and the last bit indicating it's the first or the second half.
</p>

One smart idea of CSG is that brushes are not defined by a mesh, but by a list of planes that each cut the space in half. It is much easier to perform boolean operations with lists of planes, than it is with arbitrary triangle meshes. I'm currently on the [second part of the real-time CSG blog](http://sandervanrossen.blogspot.com/2009/12/realtime-csg-part-2.html), which presents two methods for converting a list of planes into a 3d mesh. The first method, which apparently is widely used, is to project a huge axis-aligned rectangle per each cut-plane to form a polygon, then slice that polygon per each plane, and finally merge all of the vertices. My instincts agree with the author in that this seems pretty ugly. We'd have to decide on a "big value" that covers everything. What would happen if this value is too large? And, we'd have to use epsilons for floating point comparisons for vertex-merging, and having to merge vertices is just annoying.

<p class="message-info">
NOTE: Instead of storing 3d planes as point + normal, store them by their mathematical coefficients ax + by + cz + d = 0, where [a, b, c] represent the normal and 'd' the signed distance of the origin from the plane. It's less data and less useless degrees of freedom, which is always good to minimize. Calculating the distance of a point to the plane is simply  dot(point, plane.normal) - plane.d
</p>

The second, "precise", method is to loop through all combinations of plane-triples to find the intersection vertices (that are also inside the shape), then generate geometry from that. My code for this looked something like this:

```c++
for (int i=0; i<box_planes_count-2; i++) {
   Plane plane_i = box_planes[i];
      for (int j=i+1; j<box_planes_count-1; j++) {
         Plane plane_j = box_planes[j];
         for (int k=j+1; j<box_planes_count; k++) {
            Plane plane_k = box_planes[k];
            
            Vector3 p;
            if (!intersection_point_three_planes(plane_i, plane_j, plane_k, &p))
               continue;
            
            bool vertex_is_on_the_brush = true;
            for (int l=0; l<box_planes_count; l++) { // jesus, do we have to go through it all AGAIN?
               if (l == i || l == j || l == k) continue;
               
               if (dot(p, box_planes[l].abc) + box_planes[l].d > 0.000001) {
                  // the vertex is on the positive side of the plane and thus outside the brush
                  // notice that we need to use an epsilon
                  vertex_is_on_the_brush = false;
               }
            }

            if (!vertex_is_on_the_brush) continue;

            // Add the vertex to a data-structure, but be careful! A vertex can have more than three faces surrounding
            // it, so this exact same vertex might already exist! So we need some sort of vertex merging code anyway.
            // In my code, I stored the vertex positions as three integers, so I could hash the coordinate and use a
            // hash-table to quickly lookup vertices by their coordinates. We'd also naturally get rid of issues with
            // floats by not using them. I realize now that we could also get rid of our previous epsilon by first
            // checking if the rounded vertex has already been added.
      }
   }
}
```

While implementing it, I was aware of how O(n^too-much) this was, but only after implementing I looked back at it and said, "Yeah, that's too much n." I'm not entirely sure if going through all combinations of plane-triples is how you're supposed to do it or what the article suggested, but that's how I interpreted it and can't think of another way. I don't want my editor to slow down with cylinders with many sides.

I started thinking about this problem and why the "clean" solution is more work than the first one, which is *only* O(n^2). Looking at a sketch I made, if we add another line (or a plane in 3d), we will have to intersect it with every single other line (even worse in 3d) and the vast majority of intersection points will be outside the shape and unnecessary work:

<img src="{{site.baseurl}}/assets/lots_of_verts.png">

Maybe we could keep a sort-of bounding box that we would shrink with each plane and discard planes and vertices that lie outside it? That seems pretty complex and like an after-thought. I looked into known methods for solving the [vertex enumeration problem](https://en.wikipedia.org/wiki/Vertex_enumeration_problem), such as [this](https://link.springer.com/chapter/10.1007/3-540-61576-8_77) "simple" and [this](http://cgm.cs.mcgill.ca/~avis/doc/avis/AF92b.pdf) "extremely simple" algorithm. Suffice to say, I didn't get far.

I was nearly finished with implementing the dumb projected-rectangle method until it hit me: why don't we start with a cube directly? I'd need to make it possible to cut any arbitrary mesh by a plane later on anyway, so I could just hand-craft the vertices, half-edges and faces for a box, scale it up, and use that as a starting point for cutting!

<img src="{{site.baseurl}}/assets/cube_half_edge_ref.png">

## The algorithm

The algorithm is quite simple: start by finding **any** edge that intersects the plane and deleting all of the faces that lie completely outside the plane. Then start going around one of its faces until you hit another intersecting (half) edge. Split both of these edges in two and connect them by an edge. Then hop into another side of the latter half-edge and repeat until you're back at the start.

This is a lot faster than any of the previously discussed methods. Per each plane, the worst case is you need to loop through all the existing geometry once as opposed to all of the other planes! You could have thousands of planes and easily process it all in less than a millisecond. The method is also as stable as it can be, it doesn't involve any epsilons. Hooray!

<iframe width="700" height="394" src="https://www.youtube.com/embed/EghUcSJ1qrg" frameborder="0" allowfullscreen></iframe>
<p><br></p>

Stability is a big reason I chose to go down the CSG path. Arbitrary mesh-vs-mesh booleans are really difficult and often unstable. If you try to boolean two complicated meshes together in blender/maya or if they have sides that barely touch, you can expect troubles. They have a reputation for being a bit wonky, which is probably one reason they're not used that much. Frankly, you can't really even define a good way for them to work in some situations. What **should** happen when you boolean a mesh with holes and flying geometry with another? But if we just set the constraint that at least one of them has to be a BSP brush (or tree) defined by planes, everything is really intuitive! Corner cases suck, I want zero of them in my editor. So far this seems doable, we'll see how it goes!