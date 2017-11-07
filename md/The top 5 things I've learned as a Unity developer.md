

Unity is a fantastic game development platform for many reasons, one of which being the clean, accessible way its component-oriented platform is designed.  For example, it's very easy to slam together some code and have a simple working prototype running in a day or two. Despite its versatility however, I've found over the years that there's a few practices that work particularly well with Unity. 

With my new game, The Fall, I've taken many of these practices and combined them. The result has been a very smooth development process that has been quick and easy to work in and has created relatively fewer bugs that my last games.  To help celebrate my announcement of The Fall and help plug its new Kickstarter campaign, I'm writing this article to share some of my favorite design practices that might help you, with your game. 


If you get some value out of this article, please check out [The Fall on Kickstarter!](http://kck.st/1awYFJc)


### 1. Think of your code as a long term company asset
A great programmer once told me that when you write code, you should consider not just its value as an asset to the project you're working on, but as an asset to your business. For example, in The Fall, I have a dialogue system that can display a conversation between two players with avatars:  
![](http://www.overthemoongames.com/SuportImages/Evaluator_Dialogue_Thumb.jpg)

This dialogue system is written in a way that's completely independent and requires very little set-up in Unity. I can, with very little work, take this dialogue system and plug it into any future game that I want to make. Simple adjustments to it's OnGUI() function could re-skin what is essentially a perfectly functional dialogue system that could theoretically work in most cases, right out of the box. Needless to say this saves a lot of time and money, and the increased speed and ease also aides the creative process as well.


Remember: If you're going to invest the time to write code, you should be adding value to your business and creative process as well!

Remember: If you're going to invest the time to write code, you should be adding value to your business and creative process as well!



### 2. Use global class instance variables

This design pattern has helped me the most in keeping my components separate and clean.  The idea here is for each or your major scripts, you can set a global variable that points to a specific instance of that script, so that you can access that instance anywhere without having to cross-wire scripts in the inspector.  Here's how it works, more or less:

Inside my main DialogueSystem.cs script, I have a variable at the top, that has the same type of it's containing class, written as follows:
``` C#
public static DialogueSystem instance
```

Then, in the script's Awake() function, you simply write:
``` C#
instance = this;
```

You then simply attach this script to any object in your scene, and when Unity loads that scene, that particular script instance will set DialogueSystem.instance to it's self. Why is this helpful? Well, it means that if I want to access my dialogue system, from anywhere in my game, all I have to do is type:
``` C#

DialogueSystem.instance.SomePublicFunctionName();

```

And Unity will know that I want to call a function on the particular instance of the DialogueSystem script in my scene. 

**Important note -- This design pattern intends that you only have one script instance in a given scene. Therefore, this works very well for large components, such as a dialogue system, but I would imagine it would be useless or problematic to use on objects that could be instanced multiple times in a scene. If you're interested in a more safe implementation of this pattern, you can Google "Singleton" - the implementation in this article is more simple, for accessibility's sake.**


### 3. Use Callbacks (delegates or Monobehaviour.SendMessage())
This one is a little more complicated if you're a beginner. Consider this:

Lets say we have an NIS (non-interactive sequence) where we want the player's character to walk a few steps, stop, enter dialogue with an NPC, and then return control to the player once the dialogue has ended.  Well, because of point #2 in this article, we know that we can simply activate the Dialogue System from an NIS script by calling:
``` C#
DialogueSystem.instance.SomeFunctionToActivateDialogue() 
```
But how do we know when the dialogue has finished, so that the NIS script can return control to the player, or do something else that we want? Well, we can use a callback. When we first call DialogueSystem.instance.Whatever(), we can pass that function some parameters that it can use when it's finished to notify the script that called it in the first place. There are a few ways of doing this. I very much prefer using Delegates in C#, because they're a little cleaner, but if you use Javascript, they're not an option.

##### Delegates(C#)
Think of delegates as variables that basically point to a function. Using delegates, you can pass function A to function B as a parameter, so that function B can call function A when function B completes. I won't get into the details of creating delegates here, but it's a fairly simple process and this is a good place to get started learning:
http://msdn.microsoft.com/en-us/library/ms173171(v=vs.80).aspx
If I get some requests, I'd be happy to write a more detailed tutorial.

##### Monobehaviour.SendMessage(string)
Unity provides some callback functionality using its SendMessage function, that can basically call a function in a script, with a string that is the function's name. For example, lets say we have a script named "Foo" with a function named "FooFunction". Inside another script, we'd need a variable that points to an instance of our "Foo" script, lets say we call that variable "ourFooVariable". We could then call:
``` C#
ourFooVariable.SendMessage("FooFunction");

```
We can use this for callbacks because we can pass an instance of a script and a function name to another function. For example, in Javascript:
``` js
//Script A:
//-----------
 function Start(){ 
    ScriptB.MyFunction(this, "MyCallback"); // 'this' means this instane of Script A
 }
 function MyCallback(){
    //This will get fired after script B is finished.
 }
//------------
//Script B:
//------------
 function MyFunction(callback : MonoBehaviour, cbName : String){ 
     //do some stuff 
     //..... 
     //call back to script A: 
     callback.SendMessage(cbName);
 }
//----------- 
```
Using callbacks in conjunction with global class instance variables can help you create components that are more or less independent. In The Fall, when the player is in an NIS and the dialogue system needs to be triggered, the NIS simply calls the dialogue system and sends it a callback, and then does nothing as the dialogue system works. When it's finished, the dialogue system runs the callback function in the original NIS and the NIS then goes about it's business. Nothing needs to be cross-wired. No work in the inspector is required to get these systems to play nicely together. No NIS scripters need to mess with the dialogue system to get it to work nicely with their code. It just works.


### 4. Get scripts to disable and enable themselves.
Lets look back again to our dialogue system example. In The Fall, when the dialogue system is enabled, it draws to the GUI, all the time. I have no checks in the script to see weather or not it should be displaying the dialogue, it simply always does it. This works because the dialogue system is able to enable and disable it's self, and this works well because of points #2 and 3.

When I activate a dialogue in my dialogue system, the first line of code is for the dialogue system to enable it's self:
``` C#
this.enabled = true;
```
When the dialogue has completed and the callback has been sent, the last line of code disables it's self:

``` C#
this.enabled = false;
```
This way the dialogue system doesn't take much in the way of resources and simply sits there and shuts its mouth, so to speak, until it's needed.








### 5. Try to get away from using Update for everything -- try using coroutines.

A very common design pattern with unity development is to put simple code in a function that runs every frame, when it really only needs to be run once in a while.  For example, [there's a popular Fade-In-and-Out solution](http://answers.unity3d.com/questions/341350/how-to-fade-out-a-scene.html) that checks, every frame, weather or not it should be making the screen black, and then moves a number between 0 and 1. It does this every single frame, always, even though the vast, vast, vast, vast majority of the time, this doesn't need to be considered at all.

There are lots of times when you want a script to wait for a long period of time, and then do something gradually on command. I have found that Coroutines are often the best way of making this happen.

[Here is a simple primer on Coroutines in unity](http://docs.unity3d.com/Documentation/ScriptReference/index.Coroutines_26_Yield.html)

Basically, with Coroutines, you can get unity do something over several frames, and exit when it's done. Instead of having a fade-in-and-out script that's always updating, you could create a coroutine to fade the screen in or out, which would simply stop on completion, so Update isn't running constantly like the faucet in the kid's bathroom.

In JavaScript, a simple coroutine might look like this:
``` js
var fadeAlpha : float = 0;
function FadeIn(seconds : float){ 
     while(fadeAlpha != 1){ 
        fadeAlpha = Mathf.MoveTowards(fadeAlpha, 1, Time.deltaTime*(1/seconds)); 
        yield; 
     }
}
```
If this function gets run, it would change the value of fadeAlpha to 1 over how ever many seconds you tell it. You would then use that fadeAlpha value in a similar fashion to the script in the above link, in an OnGUI function.



### Puting it all together

Lets consider a FadeInOut script, using all of these ideas. We create a FadeInOut script and plunk only one instance into a scene and disable it. Make sure the script has a static variable called instance, and when the script runs it's Awake() function, it sets the instance variable to it's self. 

Next, we create a few functions in the script similar to the one above, only that enable the script in the first line of FadeIn(), and disables the script in the last line of FadeOut();

If we want, we can implement a callback system, but this script is probably too simple to warrant it.

Then, if we ever want the game screen to fade out, we just call FadeInOut.instance.FadeOut(1); and the game will fade out over one second. When we want it to fade in again, We can call FadeInOut.instance.FadeIn(1) and the game will fade back in over 1 second.

This way, we've created a simple FadeInOut script, that works completely independently, doesn't waste resources, and can be plunked into any scene of any unity game and work with no setup or cross-wiring in the inspector what so ever.


### Conclusion

Creating modular bits isn't just good for the sake of organization, but it also creates value for yourself over time. I'm sure there are lots of other design patterns to aide this process.  I hope that you got some value out of this and that my descriptions weren't too confusing. If you liked what you read, or have a question, please leave a comment and I'll do my best to respond. 
