# CornerCulling
Fast and maximally accurate culling method. Proof of concept in C++ and UE4.
Wallhack Penicillin.  
Calcualtes lines of sight from possible (due to latency) locations of players to the corners of the bounding volumes of enemies,
determining if they interesct with the bounding volumes of occluding objects. Analystical approach to raycasts. Speed gains from heuristics and caching.

Runtime demonstration:
https://youtu.be/SHUXDR0hleU

Accuracy demonstration:
https://youtu.be/tzrIXcdYQJE

An alternative implementation could fetch all objects potentially along each line of sight,
using an efficient bounding volume hiearchy. In a game with a large number of static objects,
this method's logarithmic complexity could result in huge speedups.

## Non-Technical Pitch

While not as rage-inducing as a blatant aim-botter, a wallhacker inflicts great mental strain on their victims, making them constantly uneasy and suspicious. Many serious FPS players, including myself, have spent hours staring at replays, trying to catch them red-handed. But at some point, it all becomes too exhausting.

Furthermore, wallhacks are often too subtle to detect with simple heuristics, making detection and punishment strategies costly and ineffective.

So, instead of detecting wallhacks, some games have adopted a prevention strategy. With this strategy, the game server independently calculates if a player can see an enemy. If the player cannot see the enemy, then the server does not send the enemy location to the player. Even if a malicious player tampers with their game, they will not be able to find information that they did not recieve.

Unfortunately, modern implementations of this idea are inaccurate or slow. I have developed a solution that is both perfectly accurate and faster than currently deployed solutions--utilizing ideas developed in the past 25 years of graphics research.

## Technical Details

Instead of using slow ray marches or fast approximations like PVS, we geometrically calculate if each potential line of sight is blocked by large, occluding polyhedra in the scene. A huge speed gain comes from caching recent polyhedra--if a corner blocked a line of sight a few milliseconds ago, it is likely to block the same line of sight now. If so, we can skip checking all other polyhedra. Ideally, the system would also implement a scene heiarchy like Binary Space Partition or Bounding Volume Heiarchy for logarithmic object lookup on cache misses. This improvement is unecessary for small 5v5 shooters like CS:GO, but it might be for military and battle-royale games. Furthemore, a poentially visible set check petween players (but not to bounding volumes if scene heiarchy is implemented) could also increase speed 

Also, we must account for latency to prevent popping. To do so, one should calculate angles not from the player's last known position, but from the most aggressive positions they could peek. A practical method to check all such positions is to, on the plane normal to the line from player to enemy, calculate the four corners of the rectangle that contains all possible peeks. Then let an object cull an enemy only when lines of sight from all four corners are blocked.

By accounting for latency, we can also afford to speed up average culling time by a factor of K if we cull every K ticks. Compared to a 100 ms ping, the added delay of culling every 30 ms instead of 10 ms is minimal--but results in a 3x speedup. I have not tested whether it is ideal to spread out the culling over multiple ticks for all gameserver instances running on a single CPU, or if it is better to stagger the full culling cycle of each game instance. I suspect that the latter will have better cache performance.

The last big tip is to keep enemies revealed for a few culling cycles. It is expensive when all polyhedra failed to occlude an enemy, especially if many of them barely failed. Keeping enemies revealed for ~200 ms does not confer a big advantage to wallhackers, but could save cpu cycles. This timer can adapt to server load.

## Major Task:  
Big refactor ¯\\\_(ツ)_/¯  
Move occlusion logic to OcclusionController class.  
Disentangle VisibilityPrisms from players and occluding objects.  

Refactor design doc:  
    Culling controller handles all culling. New culling loop.  
    
```python
# Get line segments needed to calculate LOS between each player and their enemies.
for player_i in player_indicies:  
    for enemy_i in player_indicies:
        if (get_team(player_i) != get_team(enemy_i) and
            (almost_visible(player_i, enemy_i) or
             ((cull_this_tick() and
              potentially_visible(player_i, enemy_i)
             )
            )
           ):  
            LOS_segments_queue.push(LOS_segments(player_i, enemy_i))

# Try to block segments with occluding planes in each segments' player's cache.
# This case should be common and fast.
for segments in LOS_segments_queue:
    for plane in occluding_plane_caches[segments.player_i]:  
        if (intersects(segments.left, plane) or
            intersects(segments.right, plane)
           ):
            blocked_queue.push(segments)
            break
    LOS_segment_queue_2.push(segments)

# Get occluding planes for all potentialy visible occluders.
for player_i in player_indicies:  
    for occluder in occluding_prisms:  
        if potentially_visible(player_i, occluder):  
            occluding_plane_queue.push(get_occluding_finite_plane(player_i, occluder))

# Check remaining LOS segments against all occluding planes in the queue.
for segments in LOS_segments_queue_2:
    for plane in occluding_plane_queue:  
        if (intersects(segments.left, plane) or
            intersects(segments.right, plane)
           ):
            blocked_queue.push(segments)
            update_cache_LRU(occluding_plane_caches[segments.player_i], plane)
            break

for segments in blocked_queue:
    hide(segments.player_i, segments.enemy_i)
reset_queues()
```
               
## Other Tasks (in no order):
- Implement (or hack together) potentially visible sets to pre-cull enemies and occluding objects.
- Consider using bounding volume heiarchy or binary space partition to only check objects
  along each line of sight  
- Account for Z axis with more general line of sight check. Outlined below.
- Implement UE4 polyhera imports by creating convex hulls from blue-print editable lists of points.
- Reach out to graphics experts (professors, article/book/library writers, graphics/CAD engine creators)
- Reach out to more FPS game developers.  
- Continue researching graphics community state of the art.  
- What to do about a wallhacking Jet with a lag switch? Cull harder based on trust factor?  
- Make enemy lingering visibility adaptive only when server is under load.  
- Test LRU, k-th chance, and random replacement algorithms. I suspect LRU is optimal due
  to small cache sizes and light overhead compared to checking operations  
- Design doc opimizations for large Battle Royale type games.  
  No culling until enough players die. PVS filter players and occluders. Only cull accurately up close.  
- Consider ways to partially occlude enemies, trimming down their bounding boxes.
  Currently, if two objects each occlude 99% of an enemy, the enemy is still visible because a sliver
  of their left is visible to one box, and a sliver of their right is visible to another.
  One idea is to divide player bounding boxes into 4 along the X and Y axis.
  I don't know if it's worth it.

### General blocking LOS check:
- Take as input a player camera location, the enemy bounding volume(s), and a possibly occluding convex polyhera.
- Returns if the occluder completely blocks the enemy.
- Get the player-facing polygons of each potentialy occluding volume
  by dot producting the polygons' normals with a line from the player position to a vertex of the polygon.
- Possibly collapse polygons together with weighted average of normals, but should be unecessary with simple geometry. 
- Check that the enemy is on the opposite side of all player-facing polygons. If not, return blocked = false.
- Get the perimiter edges of the polyhedral mesh formed by these surfaces by XORing in edges of all polygons.
  Interior edges are added an even numbner of times, and thus excluded.
- Create occlusion planes from the player position to the edges of the perimiter.
- Check that the enemy is on the inside of all occlusion planes. Return the result.

### Calculating potential player locations when aggressively peeking an enemy:
- Given a player with latency and thus uncertian position, draw a line from
  the player camera's most recently known position to the known center of an enemy.  
- In the plane that is normal to that line and contains the player, return the corners of
  rectangle that bounds the possible positions of the player.  

## Research

### Occlusion Culling:  
http://www.cs.unc.edu/~zhangh/hom.html  
https://www.gamasutra.com/view/feature/131388/rendering_the_great_outdoors_fast_.php?page=3  

### Improved bounding boxes (k-dops):  
https://www.youtube.com/watch?v=h4GBU-NXJ1c  

### Faster raytracing:  
http://www0.cs.ucl.ac.uk/staff/j.kautz/teaching/3080/Slides/16_FastRaytrace.pdf  
https://www.cs.cmu.edu/afs/cs/academic/class/15462-s09/www/lec/14/lec14.pdf
https://hwrt.cs.utah.edu/papers/hwrt_siggraph07.pdf
http://webhome.cs.uvic.ca/~blob/courses/305/notes/pdf/Ray%20Tracing%20with%20Spatial%20Hierarchies.pdf

"a large custom static mesh with no instancing, such as an urban scene, or a complex indoor environment, will typically use a BSP-Tree for improved runtime performance. The fact that the BSP-Tree splits geometry on node-boundaries is helpful for rendering performance, because the BSP nodes can be used as pre-organized triangle rendering batches. The BSP-Tree can also be optimized for occlusion, avoiding the need to draw portions of the BSP-Tree which are known to be behind other geometry."  
https://stackoverflow.com/questions/99796/when-to-use-binary-space-partitioning-quadtree-octree

### Note
Unsurprisingly (and fortunately), graphics researcher are decades ahead. My idea is basically shadow culling,  
which graphics researchers documented in 1997. <br />  
https://www.gamasutra.com/view/feature/3394/occlusion_culling_algorithms.php?print=1 <br />  
[Coorg97] Coorg, S., and S. Teller, "Real-Time Occlusion Culling for Models with Large Occluders", in Proceedings 1997 Symposium on Interactive 3D Graphics, pp. 83-90, April 1997.  
[Hudson97b] Hudson, T., D. Manocha, J. Cohen, M. Lin, K. Hoff and H. Zhang, "Accelerated Occlusion Culling using Shadow Frusta", Thirteenth ACM Symposium on Computational Geometry, Nice, France, June 1997.  

## Graphics Libraries:  
https://docs.unrealengine.com/en-US/API/Runtime/Core/Math/FMath/index.html  
https://www.cgal.org/  
https://www.shapeop.org/  
https://www.geometrictools.com/  
