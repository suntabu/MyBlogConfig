

Say you're making a Unity game that takes place in a large landscape dotted with windmills, and some of these windmills have tunnels that lead underground. But in Unity, the terrain collider is generated from heightmap data: it's essentially one giant bumpy plane. You can't punch holes in it.


Can we hack it to achieve the same result? Yep. Here's one way, there are two parts to it:
- Hiding a piece of terrain geometry with a "depth mask" shader
- Disabling the collider so the player (or whatever) can pass through the hole, but collides with terrain other times.


If you need more detail, here's my specific implementation:
To decide what to draw and when, 3D engines draw meshes in a certain order (Unity calls this the "[**render queue**](http://docs.unity3d.com/Documentation/Components/SL-SubshaderTags.html)"), so it can figure out what blocks what from the camera view. This is what happens when, say, a particle system sprays steam particles into a wall, and the wall cuts off part of the sprite; in that case, the wall came earlier in rhe render queue, so it blocks the part of the sprite that is stuck inside the wall. Why draw something that's behind a wall?


So if we can trick the engine into thinking there's a wall, it won't draw anything behind it, even if we don't render the wall, meaning we can selectively cut out pieces of models. That's what we do with a "depth mask" -- it's an invisible wall that hides whatever is behind it.

Here's what my depth mask shader looks like:
``` shader
Shader "Depth Mask Simple (Terrain)" {
  SubShader {
    Tags {"Queue" = "Geometry+10" } // earlier = hides stuff later in queue
    Lighting Off
    ZTest LEqual
    ZWrite On
    ColorMask 0
    Pass {}
  }
}

```

I've also overridden my default Unity terrain shader, just to make it render later in the queue, at "Geometry+100" which is AFTER the depth mask ("Geometry+10"), and I do this because I want to cover the edges of the hole with a tunnel model ("Geometry"). So my render queue now looks like this: brick tunnel >> depth mask >> terrain. 

If you don't modify the queues in the shaders, or you need a more complicated setup, then you might be better off writing a script that changes render queues on a per-object basis using [**Material.renderQueue**](http://docs.unity3d.com/Documentation/ScriptReference/Material-renderQueue.html). (For me, though, this setup is good enough because my terrain is simple and flat.)

Now we need a hole model. You're trying to model out the "mass of empty space" that the hole would otherwise occupy.

![image](http://3.bp.blogspot.com/-1d1wb7ztcJA/UB8JBlrtQnI/AAAAAAAABK4/OwnF5RT9cKQ/s1600/terrainhole2.jpg)

I used an 8-sided cylinder, just barely thicker than the terrain, so that it just barely blocks it. It's textured with a material using the depth mask shader, in the picture below, and you can see the blue skybox underneath:

![image](http://2.bp.blogspot.com/-6GPK1JEaNEU/UB8JRni2GtI/AAAAAAAABLA/p5bU0v7leKg/s1600/terrainhole1.jpg)


Now we need to turn off the collision for our hole, so the player can fall through it. I chose to make a sphere collider trigger over the hole that disables collisions between the player and the terrain when the player is inside the trigger:
``` C#
using UnityEngine;
using System.Collections;

public class TerrainHole : MonoBehaviour {

    public Collider player; // assign in inspector?
    public TerrainCollider tCollider; // assign in inspector?
    
    void OnTriggerEnter (Collider c) {
      if (c.tag == "Player") {
         Physics.IgnoreCollision(player, tCollider, true);
      }
    }
    
    void OnTriggerExit (Collider c) {
      if (c.tag == "Player") {
         Physics.IgnoreCollision(player, tCollider, false);
      } 
    }

}
```

... and then I put it all together and cover the edges of the hole with my tunnel model and boom: grassy landscape on the right, deep brick abyss on the left. Done!

![](http://2.bp.blogspot.com/-r57N99tUfC8/UB8LkjbB6eI/AAAAAAAABLI/paAM9NoDbLc/s1600/terrainhole3.jpg)


I imagine you can use this technique for making caves, craters, whatever. There are also [**some strong augmented reality uses**](http://pixelplacement.com/2011/02/15/masking-in-unity/) for this. Now get digging!