### [Unity3D: Change a script's execution order dynamically](http://gamedevrant.blogspot.com.es/2013/07/unity3d-change-scripts-execution-order.html?m=1)


So the other day I wanted to make sure one of my scripts always ran first, to check in it's init if certain conditions have been met, and if not, do some actions.

I didn't want to rely on manually setting each scene's execution order, and if it's the first level, OnLevelWasLoaded won't get called (no idea why).

So my solution - Create an editor class that made sure that my script's execution order is always before everything else:

``` C#
using UnityEditor;

[InitializeOnLoad]
public class ExecutionOrderManager : Editor
{
    static ExecutionOrderManager()
    {
        // Get the name of the script we want to change it's execution order
        string scriptName = typeof(MyMonoBehaviourClass).Name;

        // Iterate through all scripts (Might be a better way to do this?)
        foreach (MonoScript monoScript in MonoImporter.GetAllRuntimeMonoScripts())
        {
            // If found our script
            if (monoScript.name == scriptName)
            {
                // And it's not at the execution time we want already
                // (Without this we will get stuck in an infinite loop)
                if (MonoImporter.GetExecutionOrder(monoScript) != -100)
                {
                    MonoImporter.SetExecutionOrder(monoScript, -100);
                }
                break;
            }
        }
    }
}


```


Note: In order for this script to work it must reside in the Editor folder.