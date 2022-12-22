---
title: Coding your mod - QMM
nav_order: 5
parent: Your first Below Zero mod
---

# Coding your mod

![](..\images\qmm.png) 

In order to activate your mod, QModManager needs to know a bit about it. This is where the `mod.json` file comes in. Every mod has to have a `mod.json` file, so right click your project, select `Add new item...` and in the name, use `mod.json`. If Visual Studio has picked a template, don't worry, just select all and delete the content that it's added.

Now, enter in the details about your mod:

```json
{
  "Id": "KnifeDamageMod_BZ",
  "DisplayName": "Knife Damage Mod",
  "Author": "Oli Ollerenshaw",
  "Version": "1.0.0",
  "Enable": true,
  "AssemblyName": "KnifeDamageMod_BZ.dll",
  "VersionDependencies": {
    "SMLHelper": "2.9.6"
  },
  "Game": "BelowZero"
}
```

The important items here are the Assembly name, dependencies and game.

-   **Assembly name** - must match the name of the DLL that you are building

-   **Version Dependencies** - must call out other dependent mods and the version supported

-   **Game** - can be either BelowZero or Subnautica

**NOTE:** It's a good idea to keep on top of this, as well as the details of your Assembly, as mods like VersionChecker and QModManager itself, can make good use these values.

Okay, we're looking good. Now we can write some C#. Just to note, each time we add some code, I'll show the full content of the file at that point in time. It will make this section seem longer, but it will hopefully remove any confusion of what goes where and when.

What I do, and this is completely down to personal preference, is to have a single class to handle registering the mod and managing options. I always call this class `QMod`, but you can call it whatever you want. We'll stick with this for the tutorial.

**NOTE:** When I refer to `your project`, don't confuse that with `your solution`! Your solution is `Subnautica Mods`, your project is `MyFirstSubnauticaMod_BZ`.

By default, the template will have created a class for you. Right click `Class1.cs` within your project and choose rename. Give the class a name such as `QMod`. Visual Studio will helpfully rename the class for you, as well as the file.

You\'ll want to refer to those lovely references we added, so first up, add these statements. This will also set you up with a great feature of QMod, which is a log file that you can write to for debugging:

```c#
using System.Reflection;
using HarmonyLib;
using QModManager.API.ModLoading;
using Logger = QModManager.Utility.Logger;
```

We'll need to tell Harmony to patch our code, so add this into a `Patch()` method with the appropriate Harmony annotations in place:

```c#
using System.Reflection;
using HarmonyLib;
using QModManager.API.ModLoading;
using Logger = QModManager.Utility.Logger;

namespace KnifeDamageMod_BZ
{
    [QModCore]
    public static class QMod
    {
        [QModPatch]
        public static void Patch()
        {
            var assembly = Assembly.GetExecutingAssembly();
            var modName = ($"<someuniquevalue>_{assembly.GetName().Name}");
            Logger.Log(Logger.Level.Info, $"Patching {modName}");
            Harmony harmony = new Harmony(modName);
            harmony.PatchAll(assembly);
            Logger.Log(Logger.Level.Info, "Patched successfully!");
        }
    }
```

The `modName` generated above needs to be unique across mods - this prevents mods interfering or otherwise messing with each other. The `<someuniquevalue>` should be replaced with something, well, unique. I tend to use my Discord / Github username, but it's entirely up to you what you use:

```c#
var modName = ($"mroshaw_{assembly.GetName().Name}");
```

You may also note the use of the namespace. This, again, gives you an opportunity to ensure there is no ambiguity over which `QMod` class you're referring to in code. If you keep your namespace unique, clear and consistent, you reduce the risk of calling someone else's class methods in error.

**NOTE:** You'll see some calls to the Logger here. These are super useful to help debug your code. Just remember to use `Level.Info` for high level logging only. For detailed logging, especially in code that gets called a lot, remember to use `Level.Debug`. The last thing you need is to fill players log files with millions of lines of debug text. This can be especially troublesome if you're logging in an `Update` method that's getting executed every frame!

Great stuff. This gives us what we need to load a mod, patch some code, and do some good!

So, we probably want to do something with the game code. Again, personal preference, but I like to create a new class for each class that I'm impacting. For the tutorial, we'll just go with something simple, and create a new class file to hold our custom knife code. Do that by right clicking on your project and selecting `Add item...`. Select `Class` and give it a sensible name. For this tutorial, we'll go for `KnifeDamageMod.cs`.

At this stage, your Visual Studio project should look a bit like this:

![](.\media\firstmodclassproject.png)

Open up new class file and add these using statements. This gives us the basics of what we need to tell Harmony what we're up to, and for us to reference game objects within our mod code:

```{csharp}
using HarmonyLib;
using Logger = QModManager.Utility.Logger;
```

You'll see the `Logger` in there again, giving us immediate access to that fantastic static QModManager goodness.

We're going to patch the "Knife", and make it do mega damage. So, we'll tell Harmony to patch the `Start` method of the `Knife` class:

```c#
using HarmonyLib;
using Logger = QModManager.Utility.Logger;

namespace Mroshaw.KnifeDamageMod_BZ
{
    /// <summary>
    /// Class to mod the knife
    /// </summary>
    public static class KnifeDamageMod_BZ
    {
        [HarmonyPatch(typeof(Knife))]
        public static class Knife_Patch
        {
            [HarmonyPatch(nameof(Knife.Start))]
            [HarmonyPostfix]
            public static void Start_Postfix(Knife __instance)
            {
				// Our code goes here
            }
        }
    }
}
```

The `Start` method is a really useful `MonoBehavior` class method. If implemented in a class, it's something we can use that's always run only once when an instance of that class is created. So, if we want to tweak something when a knife instance is created, this is the place to do it. Something that you should be aware of is that the `start` method is executed for **every instance** of the class. This is important when your patching object instances that might be components in more than one parent. For example, `SeaTrackMotor` class instances are used in a number of game objects, including not only the "Cab", but also the Aquarium and Docking modules. So patching `Start`on the `SeaTruckMotor`class will impact all of those instances, which may or may not be what you want.

**NOTE:** What you call your classes isn't important. It's the `annotations`that matter - those are the values you see in square brackets within the code. You can find lots of information about these, and how Harmony works, in the [Harmony user guide](https://harmony.pardeike.net/articles/intro.html), but we'll talk a little about these below.

We're going to manipulate the properties of the Knife, after it's been started. At this point, we're going to tweak the damage field to make it super powerful. This is what the code looks like:

```c#
using HarmonyLib;
using Logger = QModManager.Utility.Logger;

namespace Mroshaw.KnifeDamageMod_BZ
{
    /// <summary>
    /// Class to mod the knife
    /// </summary>
    public static class KnifeDamageMod_BZ
    {
        [HarmonyPatch(typeof(Knife))]
        public static class Knife_Patch
        {
            [HarmonyPatch(nameof(Knife.Start))]
            [HarmonyPostfix]
            public static void Start_Postfix(Knife __instance)
            {
                // Double the knife damage
                float knifeDamage = __instance.damage;
                float newKnifeDamage = knifeDamage * 2.0f;
                __instance.damage = newKnifeDamage;

                // Write to the QMM log file
                Logger.Log(Logger.Level.Debug, $"Knife damage was: {knifeDamage}," +
                    $" is now: {newKnifeDamage}");
            }
        }
    }
}
```

You're probably wondering what those first two lines are:

```c#
[HarmonyPatch(typeof(Knife))]
[HarmonyPatch("Start")]
```

and this one:

`[HarmonyPostfix]`

These are what are called "annotations" and these form the basis of how Harmony hooks your code into the game. In the first two lines, you're telling harmony to patch the `Start` method of the `Knife` class, executing your `KnifeDamageMod`code.

And at what point should Harmony run your code? Well, that's where `[HarmonyPostfix]` comes in. You might remember in an earlier section where we talked about Harmony patching, and how we can run code before the game code (`[HarmonyPrefix`) or after the game code (`[HarmonyPostfix]`). Here, we want to tweak the `Knife` instance after it's been created, so we use `HarmonyPostfix`.

One other thing to consider here is the method definition. We have some options here around what parameters we specify in our `PostFix` method:

1. We can always include `__instance` in our method arguments, and this will always pass in the class instance on which the method was invoked.
2. We can also pass in parameters from the game method, if we want to manipulate or refer to those, either directly or by reference.
3. We can specify `__result` as a `ref` parameter of the same type used by the patched method, to override the return value of the patched method, if it happens to have a return value. For example, if we're patching a method that takes two `int` parameters and returns a `float`, and we wanted to manipulate that return value, we could write:

```c#
[HarmonyPostPreFix]
public static bool KnifeMethodWithReturn(Knife __instance, int param1, int param2, ref float __result)
{
    // Check the input params
    if(param1 > param2)
    {
        // Manipulate the float return
        __result = 0.0f;

        // Bypass the patched method
        return false;
    }
    else
    {
        // Continue to call the patched method
        return true;
    }
}
```

Note that in a scenario where you are using a `HarmonyPrefix` patch, you can use a return type of `bool` to determine whether to continue to execute the game class method (return `true`) or whether to skip the game class method (return `false`). That way, you can choose to do something first, then allow the method to continue or completely override the method altogether, bypassing the game code.

Things get quite involved at this point, as there are a number of rules and options that apply to your method definition, depending on what you want to do and what Harmony annotation you're using. The best place to find out more is in the [Harmony documentation](https://harmony.pardeike.net/articles/patching.html), which explains the concepts in detail.

Anyway, enough of this theoretical nonsense! Go back to your code and check it's all there. If so, we're good to go!

You can build this now by right clicking the project and selecting `Build`. All going well, you'll see `Build succeeded`. If not, go back through and check your code.

You can also find the full source for this part of the tutorial [in this GitHub repository](https://github.com/mroshaw/BeginnersGuideModSubnautica), so if you have any problems whatsoever, have a look at the source in the repo and compare it to what you have.

Your `Post Build Scripts` should also have run, creating a folder in your game location and copying in the DLL and json files, giving the game everything it needs to run your snazzy new mod!

Congratulations! It's a beautiful bouncing baby knife mod!
