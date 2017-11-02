### [How to create collidable curves (trail, lines, whatever) in Unity3D](http://blog.playmedusa.com/how-to-create-collidable-curves-trail-lines-whatever-in-unity3d/)

A few weeks before Halloween we decided to create a minigame to polish and learn new tricks (and treats, ha!). With not much time left for the game, we came up with simple mechanics: pumpkins fall from the top of the screen and must land safely. If pumpkins fall too fast they break and don’t score. By drawing a line (which would look like a spiderweb), players guide healthy pumpkins gently to the ground while throwing rotten pumpkins away.  Both actions score points. The goal consists just on getting as many points as posible in 64 seconds. We called it Halloweee! and it finally looked like you see on the image (you may try it on Android, it’s free)

![image](http://blog.playmedusa.com/wp-content/uploads/2013/11/HG5-258x400.png)  
Halloweee! Pumpkins and candies for a Halloween game (Android, free)


The trickiest part was how to create a collidable and properly textured curve from a gesture drawn by the player. Thanks to Unity’s Mesh class it was not that hard.


Two classes handle it. SpiderwebCreator takes care of mouse input. When the mouse button is clicked a new Spiderweb is created. While the mouse button is clicked, it simply adds points to the current Spiderweb. When it’s released, we stop building the current Spiderweb. All that just requires some raycasting and checking whether hitpoints are too close or far away from the last added point, so that the final spiderweb doesn’t look too cluttered or stretched.

It’s the Spiderweb class the one that builds up and creates collidable and textured curves. It keeps both a list of nodes (points added by the user while drawing the web) and the arrays of vertices, colors, triangles, normals and uv required by the meshFilter to maintain a mesh. webWidths keeps track of the width of the web at each node – these widhts are random, so that the web looks a bit more dynamic. Yes, thickness would have been a more proper name.

``` C#
public class Spiderweb : MonoBehaviour {
  ...
  MeshFilter meshFilter;
  MeshCollider meshCollider;
  Mesh mesh;

  List nodeList;
  List webWidths;

  Vector3 []vertices;
  Color32 []colors;
  int []triangles;
  Vector3 []normals;
  Vector2 []uv;

  void Awake () {
    meshFilter = GetComponent();
    meshCollider = GetComponent();
    PrepareSpiderweb();
  }

  public void PrepareSpiderweb() {
     mesh = new Mesh();
     meshFilter.mesh = mesh;
     meshCollider.sharedMesh = null;

     nodeList = new List();
     webWidths = new List();

  }
  ...

```

So, whenever a new node is added to the current Spiderweb by the Spiderweb creator, it’s both added to the nodeList and a new webWidth is also computed. If we have more than one node in the list, we rebuild the SpiderWeb (just one node makes for a silly web)

``` C#
  public void AddWebNode(Vector3 p) {
    nodeList.Add (p);
    webWidths.Add (Random.Range (minWebWidth, maxWebWidth));
    if (nodeList.Count > 1) {
  	RebuildSpiderweb();
    }
  }

```
And RebuildSpiderweb does the magic. Here’s how, step by step, with comments:

``` C#
void RebuildSpiderweb() {
  mesh.Clear();

/* Each node turns into two vertices,
adding thickness (width) to the line. */

  int totalVertices = nodeList.Count * 2;

/* The spiderweb grows downwards. For each node
we add its vertex and a second one, right below it. 
We also define vertices colors.*/

  vertices = new Vector3[totalVertices];
  colors = new Color32[totalVertices];
  for (int i = 0; i < nodeList.Count; i++) {
    vertices[i*2] = nodeList[i] + Vector3.down * webWidths[i];
    vertices[i*2+1] = nodeList[i] ;
    colors[i*2] = new Color32(255, 255, 255, 255);
    colors[i*2+1] = new Color32(255, 255, 255, 255);
  }
  mesh.vertices = vertices;
  mesh.colors32 = colors;

/* UV coordinates are computed so that 
the same piece of drawing is repeated every 
four vertices (that's a quad). Even vertices 
get the top-left point of the texture, odd 
vertices the bottom-right point of the texture. */

  uv = new Vector2[totalVertices];
  for (int i = 0; i < totalVertices; i++) {
    if (i % 2 == 0) {
      uv[i] = new Vector2 (i * 0.5f, 0);
    }
    else {
      uv[i] = new Vector2 ((i-1) * 0.5f, 1);
    }
  }
  mesh.uv = uv;

/* Triangles are tricky. Depending on whether 
we are drawing the gesture from left to right 
or right to left, we must define them 
so that the clockwise rule applies!*/

  triangleCount = totalVertices - 2;
  triangles = new int[triangleCount * 3];
  int triangleIndex = 0;
  for (int t = 0; t < triangleCount; t++) { 			
    bool leftToRight = vertices[t+2].x > vertices[t].x;
    if (leftToRight) {
      if (t % 2 == 0) {
        triangles[triangleIndex] = t;
        triangles[triangleIndex+1] = t+2;
        triangles[triangleIndex+2] = t+1;
      }
      else {
         triangles[triangleIndex] = t;
         triangles[triangleIndex+1] = t+1;
         triangles[triangleIndex+2] = t+2;
      }
    }
    else {
      if (t % 2 == 0) {
         triangles[triangleIndex] = t;
         triangles[triangleIndex+1] = t+3;
         triangles[triangleIndex+2] = t+2;
      }
      else {
         triangles[triangleIndex] = t;
         triangles[triangleIndex+1] = t+2;
         triangles[triangleIndex+2] = t-1;
      }
    }
    triangleIndex+=3;
  }
  mesh.triangles = triangles;

/* This computes normals for us */

  mesh.RecalculateNormals();

/* And finally we define the mesh 
collider with the same mesh we've 
just created. */

  meshCollider.sharedMesh = null;
  meshCollider.sharedMesh = mesh;
}
```

Each Spiderweb is also able to track the time it’s been active and fade away after five seconds (or when a new one is created). But more or less, that’s it! We’ll certainly use these collidable curves on future projects.