# C# Memory Managment for Unity Developers(part 1 of 3)

_**The following blog post, unless otherwise noted, was written by a member of Gamasutra’s community.
The thoughts and opinions expressed are those of the writer and not Gamasutra or its parent company.**_


-------------------------

[Note: This post presupposes 'intermediate' knowledge of C# and Unity.]  


I'm going to start this post with a confession. Although raised as a C/C++ developer, for a long time I was a secret fan of Microsoft's C# language and the .NET framework. About three years ago, when I decided to leave the Wild West of C/C++-based graphics libraries and enter the civilized world of modern game engines, Unity stood out with one feature that made it an obvious choice for myself. Unity didn't require you to 'script' in one language (like Lua or UnrealScript) and 'program' in another. Instead, it had deep support for Mono, which meant all your programming could be done in any of the .NET languages. Oh joy! I finally had a legitimate reason to kiss C++ goodbye and have all my problems solved by automatic memory managment. This feature had been built deeply into the C# language and was an integral part of its philosophy. No more memory leaks, no more thinking about memory management! My life would become so much easier.

 ![image](http://www.gamasutra.com/db_area/images/blog/203841/Unity_out_of_memory.png)
 
 _Don't show this to your players._  
 
 
 
 If you have even the most casual acquaintance with Unity and/or game programming, you know how wrong I was. I learned the hard way that in game developement, you cannot rely on automatic memory managment. If your game or middleware is sufficiently complex and resource-demanding, developing with C# in Unity is therefore a bit like walking a few steps backwards in the direction of C++.  Every new Unity developer quickly learns that memory management is a problematic issue that cannot simply be entrusted to the Common Language Runtime (CLR). The Unity forums and many Unity-related blogs contain several collections of tips and best practices concerning memory. Unfortunately, not all of these are based on solid facts, and to the best of my knowledge, none of them are comprehensive. Furthermore, C# experts on sites such as Stackoverflow often seem to have little patience for the quirky, non-standard problems faced by Unity developers. For these reasons, in this and the following two blog posts, I try to give an overview and hopefully some in-depth knowlege on **Unity-specific memory management issues in C#.**  
 
 
 - This first post discusses the fundamentals of memory management in the garbage-collected world of .NET and Mono. I also discuss some common sources of memory leaks.
 - The [second](http://www.gamasutra.com/blogs/WendelinReich/20131119/203842/C_Memory_Management_for_Unity_Developers_part_2_of_3.php) looks at tools for discovering memory leaks. The Unity Profiler is a formidable tool for this purpose, but it's also expensive. I therefore discuss .NET disassemblers and the Common Intermediate Language (CIL) to show you how you can discover memory leaks with free tools only.
 -  The [third](http://www.gamasutra.com/blogs/WendelinReich/20131127/203843/C_Memory_Management_for_Unity_Developers_part_3_of_3.php) post discusses C# object pooling. Again, the focus is on the specific needs that arise in Unity/C# development.  


I'm sure I've overlooked some important topics - mention them in the comments and I may write them up in a post-script.


### Limits to garbage collection  

Most modern operating systems divide dynamic memory into stack and heap ([1](http://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap), [2](http://vikashazrati.files.wordpress.com/2007/10/stacknheap.png)), and many CPU architectures (including the one in your PC/Mac and your smartphone/tablet) support this division in their instruction sets. C# supports it by distinguishing value types (simple built-in types as well as user-defined types that are declared as enum or struct) and reference types (classes, interfaces and delegates). Value types are allocated on the stack, reference types on the heap. The stack has a fixed size which is set at the start of a new thread. It's usually small - for instance, .NET threads on Windows default to a stack size of 1Mb. This memory is used to load the thread's main function and local variables, and subsequently load and unload functions (with their local variables) that are called from the main one. Some of it may be mapped to the CPU's cache to speed things up. As long as the call depth isn't excessively high or your local variables huge, you don't have to fear a stack overflow. And you see that this usage of the stack aligns nicely with the concept of [structured programming](http://en.wikipedia.org/wiki/Structured_programming).

If objects are too big to fit on the stack, or if they have to outlive the function they were created in, the heap comes into play. The heap is 'everything else' - a section of memory that can (usually) grow as per request to the OS, and over which the program rules as it wishes. But while the stack is almost trivial to manage (just use a pointer to remember where the free section begins), the heap fragments as soon as the order in which you allocate objects diverges from the order in which you deallocate them. Think of the heap as a Swiss cheese where you have to remember all the holes! Not fun at all. Enter automatic memory management. The task of automated allocation - mainly, keeping track of all the holes in the cheese for you - is an easy one, and supported by virtually all modern programming languages. Much harder is the task of automatic deallocation, especially deciding when an object is ready for deallocation, so that you don't have to.

This latter task is called garbage collection (GC). Instead of you telling your runtime environment when an object's memory can be freed, the runtime keeps track of all the references to an object and is thereby able to determine - at certain intervals - when an object cannot possibly be reached anymore from your code. Such an object can then be destroyed and it's memory gets freed. GC is still actively researched by academics, which explains why the GC architecture has changed and improved significantly in the .NET framework since version 1.0. However, Unity doesn't use .NET but it's open-source cousin, Mono, which continues to lag behind it's commercial counterpart. Furthermore, Unity doesn't default to the latest version (2.11 / 3.0) of Mono, but instead uses version 2.6 (to be precise, 2.6.5 on my Windows install of Unity 4.2.2 [EDIT: the same is true for Unity 4.3]). If you are unsure about how to verify this yourself, I will discuss it in the next blog post.

One of the major revisions introduced into Mono after version 2.6 concerned GC. New versions use generational GC, whereas 2.6 still uses the less sophisticated Boehm garbage collector. Modern generational GC performs so well that it can even be used (within limits) for real-time applications such as games. Boehm-style GC, on the other hand, works by doing an exhaustive search for garbage on the heap at relatively 'rare' intervals (i.e., usually much less frequently than once-per-frame in a game). It therefore has an overwhelming tendency to create drops in frame-rate at certain intervals, thereby annoying your players. The Unity docs recommend that you call System.GC.Collect() whenever your game enters a phase where frames-per-second matter less (e.g., loading a new level or displaying a menu). However, for many types of games such opportunities occur too rarely, which means that the GC might kick in before you want it too. If this is the case, your only option is to bite the bullet and manage memory yourself. And that's what the remainder of this post and the following two posts are about!


### Becoming your own Memory Manager

Let's be clear about what it means to 'manage memory yourself' in the Unity / .NET universe. Your power to influence how memory is allocated is (fortunately) very limited. You get to choose whether your custom data structures are class (always allocated on the heap) or struct (allocated on the stack unless they are contained within a class), and that's it. If you want more magical powers, you must use C#'s unsafe keyword. But unsafe code is just unverifiable code, meaning that it won't run in the Unity Web Player and probably some other target platforms. For this and other reasons, don't use unsafe. Because of the above-mentioned limits of the stack, and because C# arrays are just syntactic sugar for System.Array (which is a class), you cannot and should not avoid automatic heap allocation. What you should avoid are unnecessary heap allocations, and we'll get to that in the next (and last) section of this post.

Your powers are equally limited when it comes to deallocation. Actually, the only process that can deallocate heap objects is the GC, and its workings are shielded from you. What you can influence is when the last reference to any of your objects on the heap goes out of scope, because the GC cannot touch them before that. This limited power turns out to have huge practical relevance, because periodic garbage collection (which you cannot suppress) tends to be very fast when there is nothing to deallocate. This fact underlies the various approaches to object pooling that I discuss in the third post.


### Common causes of unnecessary heap allocation

**Should you avoid** foreach **loops**?

A common suggestion which I've come across many times in the Unity forums and elsewhere is to avoid foreach loops and use for or while loops instead. The reasoning behind this seems sound at first sight. Foreach is really just syntactic sugar, because the compiler will preprocess code such as this:

``` c#
    foreach (SomeType s in someList)
    s.DoSomething();

```
...into something like the the following:

```c#
using (SomeType.Enumerator enumerator = this.someList.GetEnumerator())
{
    while (enumerator.MoveNext())
    {
        SomeType s = (SomeType)enumerator.Current;
        s.DoSomething();
    }
}
```

In other words, each use of foreach creates an enumerator object - an instance of the System.Collections.IEnumerator interface - behind the scenes. But does it create this object on the stack or on the heap? That turns out to be an excellent question, because both are actually possible! Most importantly, almost all of the collection types in the System.Collections.Generic namespace (List<T>, Dictionary<K, V>, LinkedList<T>, etc.) are smart enough to return a struct from from their implementation of GetEnumerator()). This includes the version of the collections that ships with Mono 2.6.5 (as used by Unity).

[EDIT] Matthew Hanlon pointed my attention to an unfortunate (yet also very interesting) discrepancy between Microsoft's current C# compiler and the older Mono/C# compiler that Unity uses 'under the hood' to compile your scripts on-the-fly. You probably know that you can use Microsoft Visual Studio to develop and even compile Unity/Mono compatible code. You just drop the respective assembly into the 'Assets' folder. All code is then executed in a Unity/Mono runtime environment. However, results can still differ depending on who compiled the code! foreach loops are just such a case, as I've only now figured out. While both compilers recognize whether a collection's GetEnumerator() returns a struct or a class, the Mono/C# has a bug which 'boxes' (see below, on boxing) a struct-enumerator to create a reference type.

So should you avoid foreach loops?  
- **Don't use** them in C# code that you allow Unity to compile for you.
- **Do use** them to iterate over the standard generic collections (List<T> etc.) in C# code that you compile yourself with a recent compiler. Visual Studio as well as the free .NET Framework SDK are fine, and I assume (but haven't verified) that the one that comes with the latest versions of Mono and MonoDevelop is fine as well.

What about foreach-loops to iterate over other kinds of collections when you use an external compiler? Unfortunately, there's is no general answer. Use the techniques discussed in the second blog post to find out for yourself which collections are safe for foreach. [/EDIT]


**Should you avoid closures and LINQ?**

 You probably know that C# offers anonymous methods and lambda expressions (which are almost but not quite identical to each other). You can create them with the delegate keyword and the => operator, respectively. They are often a handy tool, and they are hard to avoid if you want to use certain library functions (such as List<T>.Sort()) or LINQ.
 
 Do anonymous methods and lambdas cause memory leaks? The answer is: it depends. The C# compiler actually has two very different ways of handling them. To understand the difference, consider the following small chunk of code:
 ``` c#
    int result = 0;
    
void Update()
{
    for (int i = 0; i < 100; i++)
    {
        System.Func<int, int> myFunc = (p) => p * p;
        result += myFunc(i);
    }
}
 
 ```

As you can see, the snippet seems to create a delegate myFunc 100 times each frame, using it each time to perform a calculation. But Mono only allocates heap memory the first time the Update() method is called (52 Bytes on my system), and doesn't do any further heap allocations in subsequent frames. What's going on? Using a code reflector (as I'll explain in the next blog post), one can see that the C# compiler simply replaces myFunc by a static field of type System.Func<int, int> in the class that contains Update(). This field gets a name that is weird but also revealing: f__am$cache1 (it may differ somewhat on you system). In other words, the delegator is allocated only once and then cached.

Now let's make a minor change to the definition of the delegate:
``` c#
 System.Func<int, int> myFunc = (p) => p * i++;
```

By substituting 'i++' for 'p', we've turned something that could be called a 'locally defined function' into a true closure. Closures are a pillar of functional programming. They tie functions to data - more precisely, to non-local variables that were defined outside of the function. In the case of myFunc, 'p' is a local variable but 'i' is non-local, as it belongs to the scope of the Update() method. The C# compiler now has to convert myFunc into something that can access, and even modify, non-local variables. It achieves this by declaring (behind the scenes) an entirely new class that represents the reference environment in which myFunc was created. An object of this class is allocated each time we pass through the for-loop, and we suddenly have a huge memory leak (2.6 Kb per frame on my computer).

Of course, the chief reason why closures and other language features where introduced in C# 3.0 is LINQ. If closures can lead to memory leaks, is it safe to use LINQ in your game? I may be the wrong person to ask, as I have always avoided LINQ like the plague. Parts of LINQ apparently will not work on operating systems that don't support just-in-time compilation, such as iOS. But from a memory aspect, LINQ is bad news anyway. An incredibly basic expression like the following:

``` c#
int[] array = { 1, 2, 3, 6, 7, 8 };

void Update()
{
    IEnumerable<int> elements = from element in array
                    orderby element descending
                    where element > 2
                    select element;
    ...
}
```
... allocates 68 Bytes on my system in every frame (28 via Enumerable.OrderByDescending() and 40 via Enumerable.Where())! The culprit here isn't even closures but extension methods to IEnumerable: LINQ has to create intermediary arrays to arrive at the final result, and doesn't have a system in place for recycling them afterwards. That said, I am not an expert on LINQ and I do not know if there are components of it that can be used safely within a real-time environment.


**Coroutines**  
If you launch a coroutine via StartCoroutine(),  you implicitly allocate both an instance of Unity's Coroutine class (21 Bytes on my system) and an Enumerator (16 Bytes). Importantly, no allocation occurs when the coroutine yield's or resumes, so all you have to do to avoid a memory leak is to limit calls to StartCoroutine() while the game is running.

**Strings**  
No overview of memory issues in C# and Unity would be complete without mentioning strings. From a memory standpoint, strings are strange because they are both heap-allocated and immutable. When you concatenate two strings (be they variables or string-constants) as in:
``` C#
void Update()
{
    string string1 = "Two";
    string string2 = "One" + string1 + "Three";
}
```
... the runtime has to allocate at least one new string object that contains the result. In String.Concat() this is done efficiently via an external method called FastAllocateString(), but there is no way of getting around the heap allocation (40 Bytes on my system in the example above). If you need to modify or concatenate strings at runtime, use System.Text.StringBuilder.

**Boxing**  
Sometimes,data have to be moved between the stack and the heap, For example, when you format a string as in:
``` C#
    string result = string.Format("{0}={1}",5,5.0f);
```
...you are calling a method with the following signature:

``` C#
public static string Format(
	string format,
	params Object[] args
)


```
In other words, the integer "5" and the floating-point number "5.0f" have to be cast to System.Object when Format() is called. But Object is a reference type whereas the other two are value types. C# therefore has to allocate memory on the heap, copy the values to the heap, and hand Format() a reference to the newly created int and float objects. This process is called boxing, and its counterpart unboxing.

This behavior may not be a problem with String.Format() because you expect it to allocate heap memory anway (for the new string). But boxing can also show up at less expected locations. A notorious example occurs when you want to implement the equality operator "==" for your home-made value types (for example, a struct that represents a complex number). Read all about how to avoid hidden boxing in such cases [here](http://stackoverflow.com/questions/10390782/why-cant-we-override-equals-in-a-value-type-without-boxing).


**Library methods**  
To wind up this post, I want to mention that various library methods also canceal implicit memory allocations,The best way to catch them is through profilling,Two interesting cases which I've recently come across are these:
- I wrote earlier that a foreach-loop over most of the standard generic collections does not result in heap allocations. This holds true for Dictionary<K,V> as well. However, somewhat mysteriously, Dictionary<K,V>.KeyCollection and Dictionary<K,V>.ValueCollection are classes, not structs, which means that "foreach（K key in myDict.Keys）..." allocates 16 Bytes. Nasty!
- List<T>.Reverse() uses a standard in-place array reversal algorithm. If you are like me, you'd think that this means that it doesn't allocate heap memory. Wrong again, at least with respect to the version that comes with Mono 2.6. Here's an extension method you can use instead which may not be as optimized as the .NET/Mono version,but at least manages to avoid heap allocation. Use it in the same way you would use List<T>.Reverse():

```C#
public static class ListExtensions
{
    public static void Reverse_NoHeapAlloc<T>(this List<T> list)
    {
        int count = list.Count;

        for (int i = 0; i < count / 2; i++)
        {
            T tmp = list[i];
            list[i] = list[count - i - 1];
            list[count - i - 1] = tmp;
        }
    }
}
```
There are other memory traps which could be written about. However, I don't want to give you any more fish, but teach you how to catch your own fish, Thats what the next post is about!

----------------------------

# C# Memory Management for Unity Developers (part 2 of 3)  

_The following blog post, unless otherwise noted, was written by a member of Gamasutra’s community.
The thoughts and opinions expressed are those of the writer and not Gamasutra or its parent company._

_[The [first](http://www.gamasutra.com/blogs/WendelinReich/20131109/203841/C_Memory_Management_for_Unity_Developers_part_1_of_3.php) installment of this three-part series discussed the **basics of memory management** in .NET/Mono and Unity, and offered some tips for avoiding unnecessary heap allocations. The [third](http://www.gamasutra.com/blogs/WendelinReich/20131127/203843/C_Memory_Management_for_Unity_Developers_part_3_of_3.php) dives into **object pooling**. All parts are intended primarily for **'intermediate'-level** C# developers.]_


Let's now take a close look at two paths to **finding unwanted heap allocations** in your project. The first path - the Unity profiler - is almost ridiculously easy to use, but has the not-so-minor drawback of costing a considerable amount of money, as it only comes with Unity's commercial 'Pro' version. The second path involves disassembling your .NET/Mono assemblies into Common Intermediate Language (CIL) and inspecting them afterwards. If you've never seen disassembled .NET code before, read on, it's not hard and it's also free and extremely educational. Below, I intend to teach you just enough CIL so you can investigate the real memory allocation behavior of your own code.


### **The easy path: using Unity's profiler**
Unity's excellent profiler is chiefly geared at analyzing the performance and the resource demands of the various types of assets in your game: shaders,textures,sound,gameobjects,and so on. Yet the profiler is equally useful for digging into the memory-related behavior of your C# code - even of external .NET/Mono assemblies that don't reference UnityEngine.dll! In the current version of Unity(4.3), this functionality isn't accessible from the Memory profiler but from the CPU profiler. When it comes to your C# code, the Memory profiler only shows you the **Total size** and the **Used amount ** of the Mono heap.

![image](http://www.gamasutra.com/db_area/images/blog/203842/Unity_memory_profiler.png)

This is too coarse to allow you to see if you have any memory leaks stemming from your C# code. Even if you don't use any scripts, the 'Used' size of the heap grows and contracts continuously, As soon as you do use scripts, you need a way to see where allocations occur, and the CPU profiler gives you just that.

Let's look at some example code. Assume that the following script is attached to some GameObject.
``` C#
using UnityEning;
using System.Collections.Generic;

public class MemoryAllocatingScript : MonoBehaviour
{
    void Update()
    {
        List<int> iList = new List<int>(new int[] { 072, 101,
            108, 108, 111, 032, 119, 111, 114, 108, 100, 033 });
        string result = "";

        foreach (int i in iList.ToArray())
            result += ((char)i).ToString();

        Debug.Log(result);
    }
}

```
All it does is building a string ("Hello world!") from a bunch of integers in a circuitous manner, making some unnecessary allocations along the way, How many? I'm glad you asked, but as I'm lazy, let's just look at the CPU profiler. With "Deep Profile" checkd at the top of the window, it traces the call tree as deeply as it can at every frame.
![profiler](http://www.gamasutra.com/db_area/images/blog/203842/Unity_CPU_profiler.png)

As you can see, heap memory is allocated at five different places during our Update(). The initialization of the list, it's redundant conversion to an array in the foreach loop, the conversion of each number into a string and the concatenations all require allocations. Interestingly, the mere call to Debug.Log() also allocates a huge chunk of memory - something to keep in mind even if it's filtered out in production code.

if you don't have Unity Pro, but happen to own a copy of Microsoft Visual Studio, note that there are alternatives to the Unity Profiler which have a similar ability to drill into the call tree. [Telerik](http://www.telerik.com/) tells me that their **JustTrace Memory Profiler** has a similar functionality ([see here](http://www.telerik.com/help/justtrace/snapshots-memory-snapshots-call-tree.html)). However, I do not know how well it replicates Unity's ability to record the call tree at each frame. Furthermore, although remote-debugging of Unity projects in Visual Studio(via UnityVS) is possible, I haven't succeeded in bring JustTrace to profile assemblies that are called by Unity.


**The only slightly harder path: disassembling your own code**

**Background on CIL**

If you already own a .NET/Mono disassembler, fire it up now, otherwise I can recommend ILSpy. This tool is not only free, it's also clean and simple, yet happens to include one specific feature which we need further below.


You probably know that the C# compiler doesn't translate your code into machine language, but into the Common Intermediate Language. This language was developed by the original .NET team as a low-level language that incorporates two features from higher-level languages. On one hand, it is hardware-independent, and on the other, it includes features that might best be called 'object-oriented', such as the ability to refer to modules (other assemblies) and classes.

CIL code that hasn't been run through a code obfuscator is surprisingly easy to reverse-engineer. In many cases, the result is almost identical to the original C# (VB, ...) code. ILSpy can do this for you, but we shall be satisfied to merely disassemble code (which ILSpy achieves by calling ildasm.exe, which is part of .NET/Mono). Let's start with a very simple method that adds two integers.

``` C#
int AddTwoInts(int first, int second)
{
    int result = first + second;
        
    return result;
}
```
If you wish, you can paste this code into the MemoryAllocatingScript.cs file from above. Then make sure that Unity compiles it, and open the compiled library Assembly-Csharp.dll in ILSpy (the library should be in the directory Library\ScriptAssemblies of your Unity project). If you select the AddTwoInts() method in this assembly, you'll see the following.

![image](http://www.gamasutra.com/db_area/images/blog/203842/ILSpy_AddTwoInts.png)

Except for the blue keyword hidebysig, which we can ignore, the method signature should look quite familiar. To get the gist of what happens in the method body, you need to know that CIL thinks of your computer's CPU as a stack machine as opposed to a register machine. CIL assumes that the CPU can handle very fundamental, mostly arithmetic instructions such as "add two integers", and that it can also handle random access of any memory address. CIL also assumes that the CPU doesn't perform arithmetic directly 'on' the RAM, but needs to load data into the conceptual 'evaluation stack' first. (Note that the evaluation stack has nothing to do with the C# stack that you know by now. The CIL evaluation stack is just an abstraction, and presumed to be small.) What happens in lines IL_0000 to IL_0005 is this:
- The two integer parameters get pushed on the stack.
- add get's called and pops the first two items from the stack, automatically pushing it's result back on the stack.
- Lines 3 and 4 can be ignored because they would be optimized away in a release build.
- The method returns the first value on the stack (the added result).


**Finding memory allocations in CIL** 

The beauty of CIL-code is that it doesn't conceal heap allocations. Instead, heap allocations can occur in exactly the following three instructions, visible in your disassembled code.

- newobj <constructor>: This creates an uninitialized object of the type specified via the constructor. If the object is a value type (struct etc.), it is created on the stack. If it is a reference typ (class etc.) it lands on the heap. You always know the type from the CIL code, so you can tell easily where the allocation occurs.
- newarr <element type>: This instruction creates a new array on the heap. The type of elements is specified in the a parameter.
- box <value type token>: This very specialized instruction performs boxing, which we already discussed in the first part of this series.


Let's look at a rather contrived method that performs all three types of allocations.
``` C#
void SomeMethod()
{
    object[] myArray = new object[1];
    myArray[0] = 5;

    Dictionary<int, int> myDict = new Dictionary<int, int>();
    myDict[4] = 6;

    foreach (int key in myDict.Keys)
        Console.WriteLine(key);
}

```

The amount of CIL code generated from these few lines is huge, so I'll just show the key parts here:

``` IL
IL_0001: newarr [mscorlib]System.Object
...
IL_000a: box [mscorlib]System.Int32
...
IL_0010: newobj instance void class [mscorlib]System.
    Collections.Generic.Dictionary'2<int32, int32>::.ctor()
...
IL_001f: callvirt instance class [mscorlib]System.
    Collections.Generic.Dictionary`2/KeyCollection<!0, !1>
    class [mscorlib]System.Collections.Generic.Dictionary`2<int32,
    int32>::get_Keys()
```

As we already suspected, the array of objects (first line in SomeMethod()) leads to a newarr instruction. The integer '5', which is assigned to the first element of this array, needs a box. The Dictionary<int, int> is allocated with a newobj.

But there is a fourth heap allocation! As I mentioned in the first post, Dictionary<K, V>. KeyCollection is declared as a class, not a struct. An instance of this class is created so that the foreach loop has something to iterate over. Unfortunately, the allocation happens in a special getter method for the Keys field. As you can see in the CIL code, the name of this method is get_Keys(), and its return value is a class. Looking through this code, you might therefore already suspect that something fishy is going on. But to see the actual newobj instruction that allocates the KeyCollection instance, you have to visit the mscorlib assembly in ILSpy and navigate to get_Keys().


As a general strategy for finding memory leaks, you can create a CIL-dump of your entire assembly by pressing Ctrl+S (or File -> Save Code) in ILSpy. You then open this file in your favourite text editor and search for the three mentioned instructions. Getting at allocations that occur in other assemblies can be hard work, though. The only strategy I know is to look carefully through your C# code, identify all external method calls, and inspect their CIL code one-by-one. How do you know when you're done? Easy: your game can run smoothly for hours, without producing any performance spikes due to garbage collection.


PS: In the previous post, I promised to show you how you could verify the version of Mono installed on your system. With ILSpy installed, nothing's easier than that. In ILSpy, click Open and find your Unity base directory. Navigate to Data/Mono/lib/mono/2.0 and open mscorlib.dll. In the hierarchy, go to mscorlib/-/Consts, and there you'll find MonoVersion as a string constant.

--------------------------------------------

# C# Memory Management for Unity Developers (part 3 of 3)

_The following blog post, unless otherwise noted, was written by a member of Gamasutra’s community.
The thoughts and opinions expressed are those of the writer and not Gamasutra or its parent company._

_[The **[first](http://www.gamasutra.com/blogs/WendelinReich/20131109/203841/C_Memory_Management_for_Unity_Developers_part_1_of_3.php)** installment of this three-part series discussed the basics of memory management in .NET/Mono and Unity, while the **[second](http://www.gamasutra.com/blogs/WendelinReich/20131119/203842/C_Memory_Management_for_Unity_Developers_part_2_of_3.php)** dived into the Unity Profiler and CIL to discover unwanted memory allocations in your C# code.]_


This third post is about **object pooling**. We've hitherto focused on heap allocations. Now we also want to avoid unnecessary deallocations, so that while our game is running, the garbage collector (GC) doesn't create those ugly drops in frames-per-second. Object pooling is ideal for this purpose. I will present complete code for three kinds of object pools. (You can also find them as a **[gist on Github](https://gist.github.com/wrh/7342325)**.)

### **Starting with a very simple pool class**

The idea behind object pooling is extremely simple. Instead of creating new objects with the new operator and allowing them to become garbage later, we store used objects in a pool and reuse them as soon as they're needed again. The single most important feature of the pool - really the essence of the object-pooling design pattern - is to allow us to acquire a 'new' object while concealing whether it's really new or recycled. This pattern can be realized in a few lines of code:

``` C#
public class ObjectPool<T> where T : class, new()
{
    private Stack<T> m_objectStack = new Stack<T>();

    public T New()
    {
        return (m_objectStack.Count == 0) ? new T() : m_objectStack.Pop();
    }

    public void Store(T t)
    {
        m_objectStack.Push(t);
    }
}
```

Simple, yes, but a perfectly good realization of the core pattern. (If you're confused by the "where T..." part, it is explained below.) To use this class, you have to replace allocations that make use of the new operator, such as here...

``` C#
void Update()
{
    MyClass m = new MyClass();
}

```
... with paired calls to New() and Store():

``` C#
ObjectPool<MyClass> poolOfMyClass = new ObjectPool<MyClass>();

void Update()
{
    MyClass m = poolOfMyClass.New();

    // do stuff...

    poolOfMyClass.Store(m);
}
```

This is annoying because you'll need to remember to call Store(), and do so at the right place. Unfortunately, there is no general way to simplify this usage pattern further because neither the ObjectPool class nor the C# compiler can know when your object has gone out of scope. Well, actually, there is one way - it is called automatic memory managment via garbage collection, and it's shortcomings are the reason you're reading these lines in the first place! That said, in some fortunate situations, you can use a pattern explaind under "A pool with collective reset" at the end of this article. There, all your calls to Store() are replaced by a single call to a ResetAll() method.


### **Adding complexity to the ObjectPool class**

I'm a big fan of [**simplicity**](http://kensegall.com/insanely-simple-book/) in code as well as life, the universe and everything, but the ObjectPool class is perhaps a bit too simple in its current state. If you search around for object pooling libraries in C#, you will find a variety of solutions, some of them rather sophisticated and complex. So let's take a step back and think about what additional functions we might - or might not - like to find in a generic object pooling class.

- Many types of objects need to be 'reset' in some way before they can be reused. At a minimum, all member variables may be set to their default state. This can be handled transparently by the pool, rather than by the user. When and how to reset is a matter of design that relates to the following two distinctions.
    -  Resetting can be eager (i.e., executed at the time of storage) or [**lazy**](http://en.wikipedia.org/wiki/Lazy_evaluation) (executed right before the object is reused).
    -  Resetting can be managed by the pool (i.e., transparently to the class that is being pooled) or by the class (transparently to the person who is declaring the pool object).
- In the example above, the object pool 'poolOfMyClass' had to be declared explicitly with class-level scope. Obviously, a new such pool would have to be declared for each new type of resource (My2ndClass etc.). Alternatively, it is possible to have the ObjectPool class create and manage all these pools transparently to the user.  
- Several object-pooling libraries you find [**out**](http://www.codeproject.com/Articles/188215/Object-Pool-Class-for-C-NET-Applications) , [**there**](http://xnaxen.codeplex.com/wikipage?title=Object%20Pooling%20in%20Xen&referringTitle=Home) aspire to manage very heterogeneous kinds of scarce resources (memory, database connections, game objects, external assets etc.). This tends to boost the complexity of the object pooling code, as the logic behind handling such diverse resources varies a great deal.
- Some types of resources (e.g., database connections) are so scarce that the pool needs to enforce an upper limit and offer a safe way of failing to allocate a new/recycled object.
- If objects in the pool are used in large numbers at relatively 'rare' moments, we may want the pool to have the ability to shrink (either automatically or on-demand).
- Finally, the pool can be shared by several threads, in which case it would have to be thread-safe.

 
Which of these are worth implementing? Your answer may differ from mine, but allow me to explain my own preferences.

- Yes, the ability to 'reset' is a must-have. But, as you will see below, there is no point in choosing between having the reset logic handled by the pool or by the managed class. You are likely to need both, and the code below will show you one version for each case.
- Unity imposes **[limitations on your multi-threading](http://stackoverflow.com/questions/10959832/make-background-thread-in-unity3d)**- basically, you can have worker threads in addition to the main game thread, but only the latter is allowed to make calls into the Unity API. In my experience, this means that we can get away with separate object pools for all our threads, and can thus delete 'support for multi-threading' from our list of requirements.
- Personally, I don't mind too much having to declare a new pool for each type of object I want to pool. The alternative means using the singleton pattern: you let your ObjectPool class create new pools as needed and store them in a dictionary of pools, which is itself stored in a static variable. To get this to work safely, you'd have to make your ObjectPool class multi-threaded. None of the multi-threaded object pooling solutions I've seen so far strike me as 100% safe, however...
- In line with the scope of this three-part blog, I'm only interested in pools that deal with one type of scarce resource: memory. Pools for other kinds of resources are important, too, but they're just not within the scope of this post. This really narrows down the remaining requirements.
    -  The pools presented here do not impose a maximum size. If your game uses too much memory, you are in trouble anyway, and it's not the object pool's business to fix this problem.
    -  By the same token, we can assume that no other process is currently waiting for you to release your memory as soon as possible. This means that resetting can be lazy, and that the pool doesn't have to offer the ability to shrink.

###  A basic pool with initialization and reset 

Our revised ObjectPool<T> class looks as follows:
``` C#
public class ObjectPool<T> where T : class, new()
{
    private Stack<T> m_objectStack;

    private Action<T> m_resetAction;
    private Action<T> m_onetimeInitAction;

    public ObjectPool(int initialBufferSize, Action<T>
        ResetAction = null, Action<T> OnetimeInitAction = null)
    {
        m_objectStack = new Stack<T>(initialBufferSize);
        m_resetAction = ResetAction;
        m_onetimeInitAction = OnetimeInitAction;
    }

    public T New()
    {
        if (m_objectStack.Count > 0)
        {
            T t = m_objectStack.Pop();

            if (m_resetAction != null)
                m_resetAction(t);

            return t;
        }
        else
        {
            T t = new T();

            if (m_onetimeInitAction != null)
                m_onetimeInitAction(t);

            return t;
        }
    }

    public void Store(T obj)
    {
        m_objectStack.Push(obj);
    }
}
``` 
  
This implementation is very simple and straightforward. The parameter 'T' has two constraints that are specified by way of "where T : class, new()". Firstly, 'T' has to be a class (after all, only reference types need to be object-pooled), and secondly, it must have a parameterless constructor.

The constructor takes your best guess of the maximum number of objects in the pool as a first parameter. The other two parameters are (optional) closures - if given, the first closure will be used to reset a pooled object, while the second initializes a new one. ObjectPool<T> has only two methods besides its constructor, New() and Store(). Because the pool uses a lazy approach, all work happens in New(), where new and recycled objects are either initialized or reset. This is done via two closures that can optionally be passed to the constructor. Here is how the pool could be used in a class that derives from MonoBehavior.

``` C#
class SomeClass : MonoBehaviour
{
    private ObjectPool<List<Vector3>> m_poolOfListOfVector3 =
        new ObjectPool<List<Vector3>>(32,
        (list) => {
            list.Clear();
        },
        (list) => {
            list.Capacity = 1024;
        });

    void Update()
    {
        List<Vector3> listVector3 = m_poolOfListOfVector3.New();

        // do stuff

        m_poolOfListOfVector3.Store(listVector3);
    }
}
```
If you've read the first post of this series, you know that the two delegates used in the definition of the ListOfVector3-pool are 'OK' from a memory standpoint. On one hand, they are not true closures but mere 'locally defined functions', and on the other hand, it doesn't even matter because the pool has class-level scope.

### A pool that lets the managed type reset itself

The basic version of the object pool does what it is supposed to do, but it has one conceptual blemish. It violates the principle of encapsulation insofar as it separates the code for initializing / resetting an object from the definition of the object's type. This leads to tight coupling, and should be avoided if possible. In the SomeClass example above, there is no real alternative because we cannot go and change the definition of List<T>. However, when you use object pooling for your own types, you may want to have them implement the following simple interface IResetable instead. The corresponding class ObjectPoolWithReset<T> can hence be used without specifying any of the two closures as parameters (which I left in for the sake of flexibility).

``` C#
public interface IResetable
{
    void Reset();
}

public class ObjectPoolWithReset<T> where T : class, IResetable, new()
{
    private Stack<T> m_objectStack;

    private Action<T> m_resetAction;
    private Action<T> m_onetimeInitAction;

    public ObjectPoolWithReset(int initialBufferSize, Action<T>
        ResetAction = null, Action<T> OnetimeInitAction = null)
    {
        m_objectStack = new Stack<T>(initialBufferSize);
        m_resetAction = ResetAction;
        m_onetimeInitAction = OnetimeInitAction;
    }

    public T New()
    {
        if (m_objectStack.Count > 0)
        {
            T t = m_objectStack.Pop();

            t.Reset();

            if (m_resetAction != null)
                m_resetAction(t);

            return t;
        }
        else
        {
            T t = new T();

            if (m_onetimeInitAction != null)
                m_onetimeInitAction(t);

            return t;
        }
    }

    public void Store(T obj)
    {
        m_objectStack.Push(obj);
    }
}
```

### A pool with collective reset

Some types of data structures in your game may never persist over a sequence of frames, but get retired at or before the end of each frame. In this case, when we have a well-defined point in time by the end of which all pooled objects can be stored back in the pool, we can rewrite the pool to be both easier to use and significantly more efficient. Let's look at the code first.

``` C#
public class ObjectPoolWithCollectiveReset<T> where T : class, new()
{
    private List<T> m_objectList;
    private int m_nextAvailableIndex = 0;

    private Action<T> m_resetAction;
    private Action<T> m_onetimeInitAction;

    public ObjectPoolWithCollectiveReset(int initialBufferSize, Action<T>
        ResetAction = null, Action<T> OnetimeInitAction = null)
    {
        m_objectList = new List<T>(initialBufferSize);
        m_resetAction = ResetAction;
        m_onetimeInitAction = OnetimeInitAction;
    }

    public T New()
    {
        if (m_nextAvailableIndex < m_objectList.Count)
        {
            // an allocated object is already available; just reset it
            T t = m_objectList[m_nextAvailableIndex];
            m_nextAvailableIndex++;

            if (m_resetAction != null)
                m_resetAction(t);

            return t;
        }
        else
        {
            // no allocated object is available
            T t = new T();
            m_objectList.Add(t);
            m_nextAvailableIndex++;

            if (m_onetimeInitAction != null)
                m_onetimeInitAction(t);

            return t;
        }
    }

    public void ResetAll()
    {
        m_nextAvailableIndex = 0;
    }
}
```


The changes to the original ObjectPool<T> class are substantial this time. Regarding the signature of the class, the Store() method is replaced by ResetAll(), which only needs to be called once when all allocated objects should go back into the pool. Inside the class, the Stack<T> has been replaced by a List<T> which keeps references to all allocated objects even while they're being used. We also keep track of the index of the most recently created-or-released object in the list. In that way, New() knows whether to create a new object or reset an existing one.