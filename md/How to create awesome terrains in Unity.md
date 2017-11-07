

Today we wanna give a short insight on how we create terrains for planet surfaces. Though Unity has some terrain shaping tools, they are somewhat limited.

We’re using a couple of tools to receive a nice, realistic look.

The basic shape of the terrain is done in Mudbox or any other good sculpting software that can export heightmaps. We will use the heightmap in World Machine, a procedural terrain creation software that will help us add realistic details like erosion. You can get a free version of World Machine here.

Once we have a nice looking terrain we can export a new heightmap for Unity, a colormap, a splatmap… you name it! To import our maps in Unity we’ll use the awesome ats Colormap ULTRA terrain shader.

### Creating the basic shape

First we need to import a simple plane model into Mudbox. We will use a plane with 512 x 512 units (cm in Maya) and 8 subdivisions as OBJ file. You can use any 3d modeling software to create that plane or [**download the plane here**](http://www.bitsalive.com/blogfiles/simple_plane.zip).  


### Working on the details

Open Mudbox! 

Before we jump in, let’s set some parameters.
- Adjust the FOV parameter of the Perspective layer to match the FOV value on your camera in Unity (60 per default). This way you have a better judgement of scale and an equal perspective.
- Set the Sculpt tool’s direction to Y only. Since the heightmap stores height information, we want polygons to only move upwards in the Y direction. 
- Choose a nice Stamp Image for the Sculpt tool. You can do this after you formed your basic shape. The bw_cliffFace.tif stamp works best for rock surfaces.

![image](http://www.bitsalive.com/wp-content/uploads/2013/10/mudbox_settings.png)

Start sculpting your terrain until you’re satisfied with what you got. When you’re done your terrain could look something like this:   

![image](http://www.bitsalive.com/wp-content/uploads/2013/10/mudbox_terrain.png)

Now let’s export a heightmap of our terrain to use in World Machine! 

For this we re-import the simple plane OBJ file into the scene to project the high-poly terrain data onto the flat low-poly plane.

After that go to _UV & Maps → Extract Texture Maps → New Operation_

Select Displacement Map and adjust the following parameters:
- *Target Models (low resolution mesh)  → Add your low-poly plane*
- *Source Models (high resolution mesh) → Add your high-poly terrain*
- *Search Distance → Try with “Best Guess” and adjust if you get artifacts in the heightmap*
- *Image Size → Let’s start with a 2048 x 2048 resolution*
- *Base File Name → Select your filename and choose TIFF 16 bit Integer (RGBA)*
- *Click Extract to bake your hightmap*


![image](http://www.bitsalive.com/wp-content/uploads/2013/10/export.png)

When the baking process is done you will have a 2048 x 2048 heightmap image.


### Adding procedural terrain details

Let’s open World Machine and set up some nodes! It’s best to set the resolution to a lower value (513 x 513) while fooling around with parameters. This speeds up build-time. Once you’re happy with your terrain you can increase the resolution to 2049 x 2049. You can change the resolution in the Project Settings menu. World Machine comes with a lot of presets for every node. Just try out different settings!

Creating a new heightmap

- *First we will need a File Input node from the Generator tab to load our heightmap. Double-click the node and choose your TIF heightmap file.*
- *You can connect various modification nodes (like Erosion from the Natural tab) to the heightfield output of the File Input module*
- *Finally connect a Height Output node from the Output tab to the heightfield output of the last module from your modificator chain.*
- *Double click on the Height Output node and choose a filename. Select RAW16 as file format and change the file extension of the previously selected filename from .r16 to .raw*
- *Click Write output to disk!*

![iamge](http://www.bitsalive.com/wp-content/uploads/2013/10/heightmap_settings.png) 

Creating a splatmap

A splatmap is used for texturing the terrain depending on it’s height-level. Therefore we have to split the terrain’s height information into separate channels to get RGBA values. We’ll use the *Select Slope* and the *Select Height* module from the *Selector* tab.

Our nodes could look something like this:

![image](http://www.bitsalive.com/wp-content/uploads/2013/10/splatmap_settings.png)

Slope Selectors can be configured to only select terrain data from a given height range. You get the best results if the Maximum and Minimum values overlap with the other Slope Selectors.

![image](http://www.bitsalive.com/wp-content/uploads/2013/10/slope_settings.png)

Since the Selectors only have one heightfield output, we have to use a Channel Combiner from the Combiners tab and connect the heightfields to the Red, Green and Blue inputs of the Channel Combiner. Finally we connect a Bitmap Output node from the Outputs tab to export our splatmap as a 16 bit PNG image.

Creating a normal map

A normal map is great way to add structure to your terrain. To export a normal map we need to add a *Normal-Map Maker* node from the Converter tab and connect it to a Bitmap Output.

![image](http://www.bitsalive.com/wp-content/uploads/2013/10/normal_settings.png)


Creating a color map

Finally we need to bake a color map using the Colorizer node from the Converter tab. To add some more detail we will export a curvature map using the Select Convexity node from the Selector tab. To export the image files we will need to connect a Height output and a Bitmap Output node.  

![image](http://www.bitsalive.com/wp-content/uploads/2013/10/settings_full.png)

We will combine the curvature map with the color map using Photoshop. Just fool around with transparency levels and blend modes (overlay seems to work best) until you're happy with the result.

if you wanna dig further into creating terrains using Mudbox and World Machine, there is a wonderful in-depth [tutorial](http://youtu.be/p0o3bqoM0Qg) for CryEngine and UDK. Since the 2nd part deals with importing the maps into CryEngine continue reading - we'll conver the Unity part in the following chapters!

When you're done you should have 4 images and a heightmap as RAW data file:
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/colormap.jpg)  
Color Map


![image](http://www.bitsalive.com/wp-content/uploads/2013/10/curvaturemap.jpg)  
Curvature Map


![image](http://www.bitsalive.com/wp-content/uploads/2013/10/normal_map.jpg)  
Normal Map


![image](http://www.bitsalive.com/wp-content/uploads/2013/10/splatmap.jpg)  
Splat Map


### Importing the Terrain & Maps into Unity

Open up Unity and add your maps to the project. Additionally add at least 4 textures which you wanna use on your terrain (e.g. grass, rock, ...) We will be using the [ats Colormap ULTRA terrain shader](https://www.assetstore.unity3d.com/#/content/4722) which you can get for only 10$ from the Unity Asset Store. It's worth it!

Create a new terrain with *GameObject → Create Other → Terrain*. Go to the Settings tab of your newly created terrain object and set the *Heightmap Resoulution to 2049* (the size we used to export maps from World Machine) and import your heightmap RAW file by clicking the "*Import Raw*" button.  
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/terrain_settings.png)  

*Warning:* You have to choose "*Windows*" as the Byte Order in case it's set to "Mac"! Otherwise your terrain will be totally messed up.  
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/heightmap_import.png)  

### Using the ats Colormap ULTRA terrain shader

Setting up the ats Colormap shader is a bit complicated, but the documentation which is contained in the package is really good. The best way to get your terrain up and running is to follow the steps from the manual.

Anyway. here's a quick overview of how to setup the shader:
- Add at least 4 terrain textures to the terrain’s paint tool  
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/add_textures.png)  
- Add the “CustomTerrainScriptColorMapUltraU4.cs”-script (when using Unity 4) to your terrain.
- Assign your colormap and the normalmap using the new editor that appeared on the terrain when you added the Colormap script.  
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/add_colormap.png)

- Add your splatmap and hit “Apply Splat Map” In case you have troubles with the splatmap’s format, you can change it the import settings of the image file in Unity.  
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/add_splatmap.png)


- Create combined normalmaps by adding your terrain texture’s normal maps to the slots and export the newly combined normalmaps
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/combine_normals.png)

- Generate average colors for seamless blending for each texture  
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/avg_color.png)


- Create a new terrain material  
![image](http://www.bitsalive.com/wp-content/uploads/2013/10/create_material.png)


Now you should have an awesome terrain in Unity! If you encounter any problems with ats Colormap ULTRA terrain shader please check the user manual that’s included in the package. It’s really detailed and you should be able to solve any issues.


Here’s the terrain we’ve just created:

![image](http://www.bitsalive.com/wp-content/uploads/2013/10/terrain_final.jpg)