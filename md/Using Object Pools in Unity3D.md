### [Using Object Pools in Unity3D:Why and How.](http://freakycreations.net/using-object-pools-in-unity3d-why-and-how-kind-of/)

So, if you are a game programmer and don’t want to share that fate, please read on, as I will share some thoughts about why you should use this and then the design and implementation of our own Object Pool class, that, while not perfect, it has helped us greatly to optimize our game.

If you are not a game programmer but you find this helpful or intersting or you are just curious, read on too! but know that I might get a little bit technical at the beginning and a lot in the last part.


#### The Purpose:

Imagine you are developing a game about things that throw throwable things to other things; lets say for example, a dinosaur(You) that throws cellphnoes to a neverending horde of mutant rabbits. Let’s also say that once a cellphone gets thrown and hits something, it explodes, and if it hits a rabbit, it explodes as well.

So, if we talk about programming, what does all of this mean? Well, it means that we have to make appear on our game scene lots and lots of instances of the same objects (cellphones and rabbits). That is simple to accomplish, we just have to instantiate our objects every time we need to, that is every time you press the ‘throw’ button or everytime you want to make a new rabbit appear, and destroy them everytime they explode. Easy, huh?

Now, there is a very big problem with that implementation. I ask to you: What if you are making a bigger version of that same game where you have to spawn hundreds of rabbits and cellphones in very short lapses of time? Even the best PCs and game consoles won’t run that game smoothly, as there is a large overhead in instatiating and destroying objects, (remember that this means allocating and freeing large quantities of memory, and doing this indiscriminately may bring chaos and destruction through issues like Garbage Collection or Heap Fragmentation and those things Gustavo likes to talk about).

#### Finding a Solution:

Very smart people designed a solution for this kind of issue a while ago: It is called “Object Pooling”. It consists on instantiating a fixed quantity of the needed objects while the program, app, game or whatever is loading; and setting them into a “Stand By” state; then when they are needed, are changed to an “Active” state, then when they are no longer needed, instead of destroying the objects, we recycle them, changing them again to the “Stand By” state.

The container that holds all these pre-instantiated objects is called a “Pool”, which is also in charge of managing the states of the objects.

Note that using this implementation there is no constant Instantiate/Destroy overhead (which was the source of the problem) as all the objects will be in the scene all the time, but the ones in “Stand By”, will not be visible, and will not interact with the game world or the player.

#### 3. Implementing a Specific Solution:

Ok. Let’s get our hands dirty now. The following is a specific implementation for the Unity3D game engine using C# as its scripting language.

``` C#
//The MIT License (MIT)
 
//Copyright (c) 2013 Freaky Creations
 
//Permission is hereby granted, free of charge, to any person obtaining a copy
//of this software and associated documentation files (the "Software"), to deal
//in the Software without restriction, including without limitation the rights
//to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
//copies of the Software, and to permit persons to whom the Software is
//furnished to do so, subject to the following conditions:
 
//The above copyright notice and this permission notice shall be included in
//all copies or substantial portions of the Software.
 
//THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
//IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
//FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
//AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
//LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
//OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
//THE SOFTWARE.
 
using UnityEngine;
using System.Collections.Generic;
 
namespace Freaky.Unity {
    public class Assert {
 
        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        public static void Test ( bool assertion, string message ) {
            if (!assertion) throw new UnityException(message);
        }
    }
 
}
 
namespace Freaky.Unity.Utilities {
 
    /*
     * Invariants:
     * - recycled elements are not active
     * - spawned elements are active
     *
     * Author: Gustavo Totoy, Javier Ron, Pedro Lucas.
     */
    [System.Serializable]
    public class Pool<T> where T : Component {
 
        #region Interface
        public Pool( T prefab, Transform parent, int initialSize = 0, bool fakeMode = false ) {
 
            fakeMode_ = fakeMode;
            prefab_ = prefab;
            parent_ = parent;
 
            spawnedElements_ = new List<T>(initialSize);
            recycledElements_ = new Queue<T>( ( !fakeMode_ ) ? initialSize : 0 );
            if ( !fakeMode_ ) {
                for ( int i = 0; i < initialSize; ++i ) {
                    recycledElements_.Enqueue( Allocate( Vector3.zero, Quaternion.identity ) );
                }
            }
        }
 
        public T Spawn( Vector3 position, Quaternion rotation ) {
            if ( fakeMode_ ) {
                T newElem = Allocate( position, rotation);
                spawnedElements_.Add(newElem);
                newElem.gameObject.SetActive(true);
                return newElem;
            }
 
            T newSpawnedElement;
            if(recycledElements_.Count > 0){
                newSpawnedElement = recycledElements_.Dequeue();
                newSpawnedElement.transform.position = position;
                newSpawnedElement.transform.rotation = rotation;
            }else{
                newSpawnedElement = Allocate( position, rotation);
            }
            spawnedElements_.Add(newSpawnedElement);
            GameObject newGameObject = newSpawnedElement.gameObject;
            newGameObject.SetActive(true);
            return newSpawnedElement;
        }
 
        public T Spawn( Vector3 position ) {
            return Spawn(position, Quaternion.identity);
        }
 
        public T Spawn() {
            return Spawn(Vector3.zero, Quaternion.identity);
        }
 
        public void Recycle( T spawnedElement ) {
            if ( fakeMode_ ) {
                if ( spawnedElements_.Remove(spawnedElement) ) {
                    if ( spawnedElement != null ) FakeModeDeleter(spawnedElement);
                    return;
                } else {
                    throw new System.ArgumentException();
                }
 
            }
 
            if ( spawnedElements_.Remove(spawnedElement) ) {
                if ( spawnedElement != null ) PoolDeleter(spawnedElement);
            } else {
                throw new System.ArgumentException();
            }
 
        }
 
        public void Recycle(System.Predicate<T> recyclingCondition) {
 
            System.Action<T> deleter;
 
            if(fakeMode_) {
                deleter = FakeModeDeleter;
            } else {
                deleter = PoolDeleter;
            }
 
            int eraseFrom = RemoveIf(spawnedElements_, recyclingCondition, deleter);
 
            if(eraseFrom != spawnedElements_.Count) {
                Erase(eraseFrom, spawnedElements_);
            }
 
        }
 
        public void Clear () {
            for ( int i = 0; i < spawnedElements_.Count; ++i ) {
                GameObject.Destroy(spawnedElements_[i].gameObject);
            }
 
            T recycledElem;
            while ( recycledElements_.Count > 0 ) {
                recycledElem = recycledElements_.Dequeue();
                GameObject.Destroy(recycledElem.gameObject);
            }
 
            spawnedElements_.Clear();
            recycledElements_.Clear();
        }
 
        public int RecycledCount {
            get {
                return recycledElements_.Count;
            }
        }
 
        public int SpawnedCount {
            get {
                return spawnedElements_.Count;
            }
        }
        #endregion
 
        #region Private Member Functions
        T Allocate( Vector3 position, Quaternion rotation ) {
            var elem = GameObject.Instantiate(prefab_, position, rotation) as T;
            elem.transform.parent = parent_;
            elem.name = string.Concat(elem.name, " ", nAllocatedElements_);
            Assert.Test(!elem.gameObject.activeSelf, "The prefab must not be active.");
            ++nAllocatedElements_;
            return elem;
        }
 
        void FakeModeDeleter(T x) {
            GameObject.Destroy(x.gameObject);
        }
 
        void PoolDeleter(T x) {
            x.gameObject.SetActive(false);
            recycledElements_.Enqueue(x);
        }
 
        int RemoveIf(List<T> collection, System.Predicate<T> p, System.Action<T> deleter) {
            int first = collection.FindIndex(p);
            int last = collection.Count;
 
            if(first == -1) return last;
 
            if(first != last) {
                for(int i = first; ++i != last; ) {
                    if(!p(collection[i])) {
                        Move(collection, i, first, deleter);
                        ++first;
                    }
                }
            }
 
            return first;
        }
 
        void Erase(int eraseFrom, List<T> collection) {
            while(eraseFrom < collection.Count) {
                Recycle(collection[eraseFrom]);
            }
        }
 
        void Move(List<T> collection, int indexSource, int indexDestiny, System.Action<T> deleter){
            if ( collection[indexDestiny] != null ) {
                deleter(collection[indexDestiny]);
            }
            collection[indexDestiny] = collection[indexSource];
            collection[indexSource] = null;
        }
 
        #endregion
 
        #region Private Data Members
        T prefab_;
        Transform parent_;
        Queue<T> recycledElements_;
        List<T> spawnedElements_;
        bool fakeMode_;
        int nAllocatedElements_;
        #endregion
    }
}
```

This implementation is as Unity3D/C# friendly as we could make it, trying to avoid unnecessary allocations(Which is bad always, but very very bad in Unity), and implementing an interface that emulates the Instatiate/Destroy fuctions, which are more apropiatedly called Spawn/Recycle.

It is also implemented as a Generic Type, so you can just pass throught there your type of pooled object, in this case it could be a component attached to a Unity prefab that is the base for your pooled ‘GameObjects’.

I hope you found this useful and/or interesting, and I also hope to be able to write more about this things that you should consider doing when programming a game, and that you don’t share the same difficulties and issues that this programming team I know and told you about did…

Good-Bye for now! 😀