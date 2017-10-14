

# Performance tips for Unity 2d mobile


#### At the minute I'm looking into performance with our latest game (which I'll write about once it's out on iOS), so I figured I'd gather together all the different tips that I ended up using to get our game running back up at 60fps.

#### You see, it used to run at 60fps no problem, but as development continued and the Unity versions piled up, occasional hiccups became more and more obvious, so it was time to dig down and figure out what was going on behind the scenes. We're currently Android only (as it's a ton quicker to iterate and get everything working as it should), but the latest Unity version (5.3 as of writing) seemed to exacerbate the performance problems on this.


# Our game

There's no one silver bullet to optimisation, and what works for one person mightn't work for another, so with this in mind, this is our game constitutes of:
- It's a 2D golf game, for mobile, using basic Sprites
- It uses multiple scenes, and the new Unity UI
- In the game scene, the world map is built with individual tiles
- It's multiplayer, so you get CPU spikes as messages come and go
- We don't use any lights, multiple cameras, etc  


 It goes without saying that the performance issues were found using the profiler, so if you have it, it's something you should be using.  
 Also, these tips are mainly focused on 2D. Your mileage may vary.


# The dumb stuff

Let's get the dumb stuff out of the way first. These are the simple things you need to do, and don't need much discussion.

- Set your **Application.targetFrameRate** to 60. By default on mobile, it's 30
- Otherwise you can turn on **vSyncCount** in the quality settings. Apparently, on some Android devices, with Android 5, you can render at more than 60fps, so if you don't force vSync, you run the risk of an unresponsive device, as it's rendering too quickly, so it's possible that future versions of Unity will force this
- When testing, test on an actual device - you can run on the desktop to make sure it doesn't crash, but it won't tell you a tap about what's happeninig on the device
- If you're not using **[object pools](https://en.wikipedia.org/wiki/Object_pool_pattern)**, use object pools
- Use something like TexturePacker to pack your images into one, so you can get Dynamic Batching going
- Remove empty callbacks from components. If they're there, even if there's nothing in them, they'll get called, which slows you down
- Cache components you use regularly. Ditto for GameObjects. No GameObject.Find() when you can get away with it, especially not in Update/FixedUpdate
- Disable/Disactivate GameObjects/Components when you don't need them, especially if they have Update/FixedUpdate calls
- If using physics, don't move Rigidbody2D yourself (i.e. directly setting the postion), especially if they've been marked as static. The physics engine doesn't like and will let you know. If you need to, disable the GameObject, set the position using the Transform (there's currently a bug when setting the Rigidbody2D.position while the GameObject is deactivated), then re-activate it
- Watch your poly count. We had a pretty high-resolution ball, which was fine with 1/2 players, but we'll be releasing soon with 1v3/teams, so all of a sudden your poly count doubles
   
After that comes all the things that require a bit more effort on your part.  

Let's go a bit deeper.

# Control the garbage collector

#### One of the worst things that can happen in the middle of playing is the GC kicking in. You'll usually notice if it looks like your rendering is skipping every so often.
  
Now the important thing to remember is that the GC only kicks in when the application requests more memory. The default behaviour is to try and free some up before asking for more from the OS. This means that if you instantiate everything you need up front, and you have no other memory allocations during play, you'll never have a problem with the GC.  

If you're using object pools, easy, right?  

The killer is the hidden allocations. These can be big or small, but they build up until eventually your application requests another memory page, resulting in the GC crapping all over your game.
- Using a foreach will allocate 32B of memory for the enumerator. While it might not sound like much, a couple of these inside an Update/FixedUpdate and you'll feel it
- Adding or removing a callback from a delegate will allocate 104B of memory. I've not seen a way around this
- Strings allocate memory as you create them. Keep a special eye out for concatenation - "Hello " + "world" - as in this case you're actually allocating 3 strings here, one for each part, and one for the final result (strings are immutable). This is pretty easy to do if your calling Debug.Log() a lot. Using StringBuilder or the String.Format() where you need to
- Know the difference between classes and structs and when to use them. Classes are created on the heap, while structs are passed by value. Be especially careful of Lists of structs  

In the CPU Usage of the profiler, you can sort by GC Alloc column to find out who's allocating and when. Removing as many of these allocations as possible means your game will have a smoother ride. Some will be outside of your control (such as physics engine allocations), but we can live with those.  
In the Memory Area of the profiler, you can also take a snapshot of the current memory state, which you can use to help you see if there's a memory leak somewhere.  

You can also call the GC yourself, using System.GC.Collect(). Do this when you can afford to. We do it, for example, at the end of a shot, when the ball has finished moving, but before the next player has taken their turn. We allow a collection here, which lowers the chance of it happening where it would be visible (such as when balls are moving).

# Logging


#### In relation to one of the previous points, you'll need to remove any calls to Debug.Log() etc that you have. This is because:

- LogStringToConsole() - the method called to print out your message is super slow. Even if you're not listening for it (i.e. have adb or whatever connected), it'll still get called
- As well as that, when profiling, you need to select Development build, so the log message also grabs a stack trace (similar to how you'd see it in the Unity editor), which is even slower. You'll see a spike for every message when profiling
- As well as that, creating the message to log will entail lots of string concatenation and memory allocation, especially if you've overridden ToString() (and you should), and your logs are something like Debug.Log( "Player " + player + " is doing something in game " + game );  
  

"No problem", you say, "I have my own log function that wraps Debug.Log(), so I only need to set a compilation constant inside there to early return out and not print anything!"   

Sure, that'll solve the call itself, but a call to an empty function will still allocate the memory for the string message passed, giving you eventual garbage collection problems.  

In the end, you have two three solutions (James from the comments showed a pretty neat way of removing Debug.Log() calls):
- Wrap each call to Debug.Log() in a compilation constant so that the call doesn't exist when you do a production build
- Before you build, do a Find and Replace or run a script to comment then all out
- Use the C# Conditional attribute. It's essentially a neater, more elegant way of doing conditional compliation. Inside a class, add the attribute to a static void function. If the define is declared, then the function, and all its calls are included. If not, it's like they never existed:
    
``` C#
    #define USE_LOGS // NOTE: changed in 5.5 - see note below
using System;
using System.Diagnostics;

public class Trace
{
    [Conditional("USE_LOGS")]
    public static void Msg(string msg)
    {
        Debug.Log(msg);
    }
}

// elsewhere in the code - if USE_LOGS is defined, you'll see this output, otherwise,
// it's stripped from the build
Trace.Msg( "Hello world" );
```
    
- NOTE: I'm leaving the previous code example up for historical reasons, but as of [Unity 5.5, they've changed the way this works](https://forum.unity3d.com/threads/conditional-attribute-ignores-defines-in-5-5.447160). Apparently the previous behaviour was a bug, and the #define now needs to be set where you *call your method*, not where it's declared. Because of the way Unity projects work, this leaves two options: 1) add a #define in every file, which is awkward, or 2) add your constants in the PlayerSettings > Scripting Define Symbols box. They go in the form USE_LOGS;SOME_OTHER_DEFINE.

We went with the second option as it's easier in terms of dev (as I didn't know about the third one, but we now use that one as it's amazing), and you only need to run it just before a build, then Git revert the changes. On a Mac, you can make a shell script that goes something like this:
```
grep -rl --null "Debug.Log" "./Assets/Scripts" | xargs -0 sed -i "" 's|Debug.Log|//Debug.Log|g'
```

which uses grep to list all the files containing Debug.Log() calls, and pipes the result to sed which comments them out, in place.  


There are a few things to bear in mind however: 
- You need to be careful about unclosed conditional statements and the like, otherwise:
```
if( something )
	Debug.Log( "Something happened" );
myObj.Foo();
```
can become
```
if( something )
	//Debug.Log( "Something happened" );
myObj.Foo();
```
which leads to unexpected behaviour. I have another script that pulls out all the logs and the line before, that I can feed to a JavaScript tool that prints any non-wrapped log, so I can fix it (JS is great for building quick, crappy tools)

- If you're using Application.logMessageReceived or Application.logMessageReceivedThreaded (e.g. if you're sending errors to the server), then watch out that you don't comment out the logs that you're interested in (you can tweak the regex to ignore LogError() calls, for example)
- Some code uses UnityEngine.Log.Debug() instead, so check for that as well (and first)
- If you're using the script, because it's changing the line in place, it's like typing with the Insert key selected, so Debug.Log( "Hello" ) actually becomes //Debug.Log"Hello" );. As you're commenting this out, it doesn't really matter, but it's something to keep in mind if you want to adapt the script for something else


# Remove Global Illumination
This was one of the more recent additions to Unity, and it seems to be turned on by default. As we're not doing anything relating to lighting, you can turn them all off (Precomputed Realtime GI, Based GI, Fog, everything). You can find the option in the Windows > Lighting panel. You'll need to do it for each of your scenes.  

Depending on your Unity version, you might still see some spikes of GI in the profiler. This is apparently a known bug and is already fixed in an upcoming alpha.

# Force OpenGLES 2.0  
Unity is optimised for OpenGLES 2.0, but will allow you to select 3.0. It's also set by default and kind of hidden in Player Settings > Other Settings, where you need to untick Automatic Graphics API and remove OpenGLES3 from the list.  

It'll be selected if your phone supports it, but your performance will tank, so it's better to force 2.0. This one will obviously improve with time though.

# Set your colour/depth buffer  
Make sure you're using the right sized colour/depth buffer. You shouldn't use the 32bit colour buffer unless you notice banding in your gradients. As our app is pretty much gradient free, it was an easy switch to gain a lot in memory. Ditto for the depth buffer. 2D games shouldn't need much in the way of a depth buffer, so you can bring it right down. You can find these in Player Settings > Resolution and Presentation.

**NOTE: Unity 5 [seems to have done away with the ability to set your depth buffer](https://unity3d.com/unity/whats-new/unity-5.0). While it seems like the default is 16-bit (though I haven't found any confirmation of that so don't quote me on it), you can now only opt to disable it (and the Stencil buffer) altogether. You can also get a small bit more control using the DepthTextureMode property of the camera.**


# Set your quality settings  
A few simple tweaks here can go a long way:  
- First of all, disable any selections higher than the defaults (Simple for mobile)
- Set Pixel Light Count to 0 as you're probably not using them
- Set your Texture Quality to as low as you can get away with. Even Half Res on mobile is mostly unnoticable, and you'll gain about 75% memory
- In a similar vein, disable mipmaps if you're not using them (i.e. scaling down), as this'll save you about 33% in memory
- Turn off Anisotropic Textures as you probably don't have any textures that you'll see at an oblique angle
- Turn Anti Aliasing off or set it to 2 passes, which should be enough to cover any jagged lines (only really noticable if you're using stuff like LineRenderers anyway, as most of the rest should be images
- Turn off Soft Particles. They're for blending properly into meshes, but that's a 3D problem
- Turn off Shadows  
The two big ones are texture quality and anti aliasing. Texture quality directly impacts your memory and how long it takes to upload images to the GPU, while anti aliasing is applied to the entire screen, so on higher resolution devices it's going to be more expensive. The less passes for that, the better.  


# Canvas and the EventSystem  
We're using the new UI (and you should be as well), and every UI element needs a Canvas parent. By default, one is created when you add a new UI element, and by default, each Canvas adds it's own Graphics Raycaster so that you can hittest against it. More than one of these however, and you can expect your performance to tank, so remove any that you don't need. If you have the profiler, you'll see this come up as EventSystem.  

Speaking of UI, unless you specifically want to click on an element, make sure to untoggle raycastTarget, otherwise that object'll be included for interaction. On UI heavy scenes, this can easily add up (we gained 10ms on our shop scene). If you're coming from flash, this is the equivalent of mouseEnabled.  

I've also read that you can get a performance hit from not having a material on UI elements. I didn't particularly notice any difference, but there are some default UI materials included, so you can add them if you want.

# Use the right shaders

Speaking of materials, make sure you're using the right shader on your materials. The default Sprite-Default for 2D sprites handles transparency, which is not ideal. Transparency on mobile is a real buzz kill, especially if you have it on large areas of the screen. On the profiler you'll see this come up as something like renderForwardTransparent(). As we're using tiles, changing the default for a lot of them to a basic unlit brought the rendering time down significantly.  

This will obviously only work for solid sprites (i.e. square), and the fact of having a different material will split up Dynamic Batching, but the gains are worth it.

# Watch out for Unity UI Text

Unity UI Text is pretty powerful, but it's not the fastest, and you need to be very careful how you use it. We have a lot of text on the screen, with different effects such as outline, and it was hitting performance bad. We've recently made the switch to the Text Mesh Pro asset, and aside from the first frame where it's generating the texture, it's takes about half the time to render. This was a huge gain on some scenes. It's not a drop-in replacement though; you'll need to go through and replace your textfields one by one, but it's definitely worth it.  

If you want to stick with Unity Text anyway, then make sure:

- raycastTarget is turned off; it's on by default, but you're probably hit-testing against a background anyway
- Rich Text is turned off, unless you're rendering HTML etc
- Don't use Best Fit unless you really have to, or at most limit the min-max font size and only have 1 or 2 per screen. If your min font size is 10, and your max is 30, then what happens is Unity goes through the entire render cycle at 30, and checks if it fits. If it doesn't, then it goes down a notch and tries again, and so on. As you can imagine, with each render until you get the right size, this is pretty slow. Either fix your UI so you don't need to do this, or have one or two sizes in code and change them manually
- This one comes from Jaysama in the comments: Avoid dynamic fonts if you can. Dynamic fonts can be useful if you're using a common font and trying to save on download size, or working with asian languages, where the resulting font texture can be quite large. Unity won't pre-generate a font texture, but will use the underlying OS to render the text on the fly. This obviously has performance implications, especially if your text changes often.

**Update: [Unity have announced that TextMeshPro will become an official part of Unity!](https://blogs.unity3d.com/2017/03/20/textmesh-pro-joins-unity/)**


# Watch out for overdraw  
Overdraw is where the same pixel on the screen is hit multiple times. Aside from being redundant, when you combine it with transparent areas of sprites (and multiples thereof), it'll soon start to hurt. In an ideal world, you'd write to each pixel once, but this can be hard, especially with UI. In the Scene view, there's a dropdown that will show you the overdraw for the current scene, so you know where to apply your efforts.  

Unity has a polygon mode which will replace your square sprites with much tighter outlines. You have more verticies and polygons, but less overdraw, so it can be a big help. If you're using TexturePacker (and you should be), [then this is already built in for you.](https://www.codeandweb.com/texturepacker/tutorials/using-spritesheets-with-unity)    
Another thing to keep in mind is draw ordering. Normally, we'd be used to drawing a large background covering the screen, then drawing other elements over it. On mobile, it can be better to draw your front elements first, then use the depth buffer when drawing the background. A depth check is much quicker than overwriting a pixel.  


# Draw calls, Dynamic Batching, etc  
By default Dynamic Batching should be activated (if not, you can find it in Player Settings > Other Settings). Basically it gathers up all sprites using the same material and renders them all in one draw call. There are a number of things that break batching, such as differing scales, tints, textures etc, so you might need to reorganise how you draw your elements, or combine textures into a spritesheet. A Text UI element will probably not be rendered with UI sprites, and for one example that we had, simply moving it from behind one Sprite to in front of it brought our draw calls from 3 to 7. Changing the Pos Z of the element will also probably break it.  

Each draw call you have means a context switch for the GPU, which is expensive on mobile. I'd say any more than 10 is probably too many, and more than 20 will give you problems. Basically get it as low as you can, either through batching or combining your models/textures.  

Static batching gives a better performance boost, but it means that you can't move, scale, or rotate anything that it's applied to. Unfortunately it also doesn't seem to work yet with SpriteRenderers, so it's only for meshes at the minute.  

# Set up your audio properly

An easy thing to overlook is your AudioClip settings.
- Force To Mono: I'd always set this unless you have a specific reason not to. Unless the user is listening with headphones, most speakers are mono anyway
- Load In Background: Set this for every AudioClip that you don't need to play immediately. It'll push it on to another thread so it doesn't block the main one
- Preload Audio Data: Specifies whether it should be preloaded, or loaded when you first play it. For short SFX, set it, otherwise ignore.
- Load Type: Decompress On Load will decompress it immediately, Compressed in Memory will keep it compressed, but load it anyway, and Streaming will only load what's necesssary to play, when you're playing it. For short SFX, I use Decompress On Load, while setting music files to Streaming  
- Compression Format: PCM is the best, but largest, memory-wise, ADPCM is probably the best for short SFX, while Vorbis is the best for music. ~~On mobile, everything is loaded as Vorbis anyway~~ 

Update 20/04/2017: As pointed out by Min Shin in the comments, I was actually reading the [AudioFiles Manual page](https://docs.unity3d.com/Manual/AudioFiles.html) wrong. The actual text is:
> When audio is encoded in Unity the main options for how it is stored on disk is either PCM, ADPCM or Compressed. [...] The default mode is Compressed, where the audio data is compressed with either Vorbis/MP3 for standalone and mobile platforms, or HEVAG/XMA for PS Vita and Xbox One.  

which I took to mean that only Vorbis/MP3 is used on mobile, but it's rather that Vorbis/MP3 is used on mobile, *if you choose Compressed*.  

Making a few simple changes brought our app time-to-Home-screen down from around 30s to under 15s, as well as freeing up a lot of memory.  


# Enable Multithreaded rendering  

This final point is Android only, but in Player Settings > Other Settings you'll see a toggle for Multithreaded Rendering. This puts all rendering on another thread, freeing up your main thread for application logic, obviously a good thing.  

It comes with it's own caveat though, as depending on what you're going with your project, especially if you interacting with textures, it can give graphic glitches. The only thing to do is try it for your project and see if you get away with it. It might also not work on some devices, though I'm not sure if Unity turns if off automatically or not.  

If you see problems with it on certain devices, I almost say just disable that device, as the benefits are pretty awesome.  


# Scrolling


Doing any scrolling in your game menus? The new Unity UI has a scroll component built in, but its performance can be horrible. Solution: scroll the camera itself - if necessary create a new camera with the right necessary rect. Moving a camera is a simple matrix change, so the rendering is the same, while moving via a ScrollRect seems to modify all the RectTransforms all the way down, meaning everything has to be recalculated, and if you've half-way complicated UI, this can be a huge drain. Scrolling the camera alone can easily make it twice as quick. You might need to jump through hoops to get mouse coordinates working properly, but a bit of extra code to get rid of janky scrolling is doable.  

Update 08/2016: TFG Studios got in touch through the comments to show off their optimised ScrollView. It looks pretty nice, but you can [check it out yourself on YouTube](https://www.youtube.com/watch?v=FOQV8AA50Rw)



# Know when to break from Unity  

When you first come into Unity, everything is component-this and component-that, but sometimes you need to know when to break from Unity and do things a simpler, more direct way. As an example, when we first started making the map for the games, we did a naive approach where we use tiles and each tile has a component that helped with its behaviour; one for the tile properties (drag, etc), one for the physics, one for special considerations (e.g. flashing, or affecting the ball).  

This works for quick projects, but pretty soon you realise that you've about 10k objects in the game, and most of them are useless. The properties were split off into static variables, the physics were removed and a collision/trigger mesh generated by our level tool instead (so one PolygonCollider2D instead of one for each tile that needs it), and any special cases were merged where possible, so we'd have one loop rather than 50 Update() methods running.  

For the rendering, we also made use of a RenderTexture and Camera culling masks, so all those tiles would get rendered exactly once to the texture, then hidden. For some tiles we couldn't do this (e.g for the ones that break or flash), but for the vast majority we could, which meant that our map and border tiles are one simple mesh. And because the semi-transparent tiles are baked into the texture, we can apply one super fast non-transparent material to the mesh to get the quickest rendering possible.  


# Use threads where you need to

Because our game is multiplayer, we use sockets via TcpClient to send and receive messages. Originally we were doing a synchronous write and an async read. As the EndRead() method is on another thread, there was no problem when deserialising/decompressing the received message. But that left the problem of sending. Normally it wouldn't be too bad, but sometimes you would get a message being sent out while the ball is moving, and compressing/serialising to JSON are pretty slow operations, so you're looking at minimum 1 dropped frame. Even sending the shoot message as you fired would introduce a tiny bit of lag on the client, which meant the experience wasn't great.  

A simple threading system is now in place which means that all that is pushed off the main thread, so there shouldn't be any impact because of it. Some of the logic in the game had to be rewritten to accomodate that - because the writes were synchronous before, we could reuse objects, which we can't do with async - but the performance gains are worth it. [Joseph Albahari has a great resource on threading in C#](http://www.albahari.com/threading/).


# Conclusion

Optimisation is a pretty big subject, especially when so much of it is guesswork or slowed by the constant change > build > test cycle, which on mobile isn't quick. I also find the profiler on Unity not as good compared to others, such as Scout. Your history extends as far as the window, you can't select multiple frames to get averages (even though they have this information), and you can't save the results, so before and after comparisons either work through your memory, or screenshots.  

Still, it's what we have to work with, so if you have it, use it, as sometimes, what's causing your slowdown isn't what you think it is. You don't need to profile all the time, premature optimisation being the devil and all, but you should do it regularly.  

If there's anything I've missed, let me know! Happy hunting.