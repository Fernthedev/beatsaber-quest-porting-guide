# BeatSaber C# porting guide for Quest

This guide aims to do the following things:

- Be a complementary guide with [Laurie/Danrouse' modding guide](https://github.com/danrouse/beatsaber-quest-modding-guide)
- Teach some _pointers_ on how to properly port mods (get it?)
- Discipline the reader with proper programming practices (no var p = 5)

This assumes that you have a basic understanding of C++ and a bit of C#. You will also be using codegen in this tutorial.
This guide will explain different practices in no particular order

## PRs are welcome!

You can make PRs to this repo, though your new documentation should have the following requirements:

- [x] Properly describe the practice and it's use cases. Specify when to/not to use said practice
- [x] The reader should be aware of the consequences/advantages of said method
- [x] Show examples from C# identical/similar code if applicable

## Objects and Codegen

To understand why we use codegen, we have to understand how il2cpp works behind the scenes. BeatSaber uses the Unity engine which has 2 different ways of compiling: Mono and il2cpp. The PC version is compiled in Mono which allows mods to be created in C#, while also having a garbage collector and JIT optimizations. However, the Quest version uses il2cpp. Why you may ask?

Simply, the Quest (and even Quest 2) are not very powerful compared to their PC counterparts, so running BeatSaber in Mono will be pretty bad performance (even on Q2). So, what is Unity's solution to this dilemma? Il2cpp is a program developed by Unity that allows IL (the code that C# is compiled to which Mono or another runtime interprets) and converts it to C++. This allows C++ compilers such as clang/LLVM to further optimize the code to an extreme and even allows Unity games to run on platforms that do not support Mono altogether. The problem with this? Quest Modding can't access these classes the same way PC can, we have to paradigms similar to reflection. Take a look at this:

```cpp
static const MethodInfo* method = CRASH_UNLESS(il2cpp_utils::FindMethodUnsafe("UnityEngine", "GameObject", "Find", 1));
// nullptr since it's a static method, otherwise you provide the instance
Il2cppObject* object = il2cpp_utils::RunMethod(nullptr, method, il2cpp_utils::createcssstr("someObject"))
```

As you can see, this is problematic in many ways. For one, this has no type-safety therefore a compile doesn't guarantee a run. Second, this assumes that your code is even correct and can break easily between updates without notice. The code you may need to change may be even hard to test which may cause a crash you don't realize until it's too late.

Thanks to sc2ad's wonderful work on codegen, we can avoid avoid all these issues. Simply put, this is the code with codegen installed:

```cpp
#include "UnityEngine/GameObject.hpp"

UnityEngine::GameObject* go = UnityEngine::GameObject::Find("someObject");
```

This code is safe, compact and easy to read. Codegen has the added side-benefit of making mods easier to port from C# since the code can be very similar to identical in some ways.

Codegen behind the scenes wraps around Il2CppObject* and gives easy access to it's fields and methods. Every codegen object is actually a `Il2cppObject*`

## Pointers

In C++, we use pointers due to the way il2cpp works. In C#, you knowingly or unknowingly are (most of the time) using pointers in your code.

For example, C#

```csharp
GameObject go = UnityEngine.GameObject.Find("name");
```

while on C++ we use the following (assuming you are using codegen)

```cpp
#include "UnityEngine/GameObject.hpp"

UnityEngine::GameObject* go = UnityEngine::GameObject::Find(il2cpp_utils::newcsstr("name"));
```

## Custom types and classes

In the PC mods, you will know that it's quite easy to create your own MonoBehaviours. However, on Quest we cannot just use C++ classes to extend C# classes or create Unity components. Instead, we rely on the useful library called [custom-types by sc2ad](https://github.com/sc2ad/Il2CppQuestTypePatching). This library allows us (albeit still more work than our PC counterpart) to create C# classes. Take the following for example, which is a simple MonoBehaviour:

```hpp
// OurClass.hpp
#pragma once

#include "custom-types/shared/types.hpp"
#include "custom-types/shared/macros.hpp"

#include "UnityEngine/MonoBehaviour.hpp"

DECLARE_CLASS_CODEGEN(OurNamespace, OurClass, UnityEngine::MonoBehaviour,
  public:
    DECLARE_METHOD(void, Update);
    DECLARE_CTOR(ctor);
    DECLARE_SIMPLE_DTOR();

    DECLARE_INSTANCE_FIELD(float, floatVar);

    REGISTER_FUNCTION(
      REGISTER_SIMPLE_DTOR();
      REGISTER_METHOD(ctor);
      REGISTER_METHOD(dtor);
      REGISTER_METHOD(Update);
      REGISTER_FIELD(floatVar);
    )
)
```

```cpp
// OurClass.cpp
#include "OurClass.hpp"

// Necessary
DEFINE_TYPE(OurNamespace::OurClass);

void OurNamespace::OurClass::ctor() {
  floatVar = 1.0f
  // Constructor!
}


void OurNamespace::OurClass::Update() {
  // Update method! YAY
}
```

This MonoBehaviour has a constructor, destructor, a method called "Update" and an instance field called `floatVar`.
It should be known that you cannot use non-il2cpp types in custom-types methods or fields (DECLARE\_ methods or fields), such as C++ structs or classes. You can use pointers, other custom-types and convertible value types though.

You should also know that you can use C++ methods and fields like any normal C++ class, but no constructors.

You also should not directly call the `ctor` method, and instead you should use `il2cpp_utils::New<OurNamespacce::OurClass*>(parametersHere);` (you technically can call ctor, but that won't actually construct the instance)

It is also important to know that the C++ destructor is never called when the type is freed by the GC, so if you have for example a `std::vector` or anything that needs deletion (such as std::unordered_map or anything that isn't [trivially constructed](https://en.cppreference.com/w/cpp/language/default_constructor)), it will never get freed and thus causes a memory leak. Thankfully, we can call the destructor ourselves very easily. This is a simple example as a solution to this kind of problem: (kindly provided by sc2ad)

```hpp
// type.hpp
#pragma once

DECLARE_CLASS_CODEGEN(Does, Stuff, Il2CppObject,
  std::vector<int> aCppVec;

  // This requires either INVOKE_CTOR or placement new to actually initialize the variable with the default value.
  DECLARE_INSTANCE_FIELD_DEFAULT(float, floatVar, 1.2f);

  DECLARE_CTOR(ctor);
  DECLARE_SIMPLE_DTOR();
  REGISTER_FUNCTION(
    REGISTER_METHOD(ctor);
    REGISTER_SIMPLE_DTOR();
  )
)
```

```cpp
// type.cpp
DEFINE_TYPE(Does::Stuff);

void Does::Stuff::ctor() {
  // you should only use this is if your constructor is non-trivial or contains non-trivial constructible fields such as vectors. very tiny performance impact
  INVOKE_CTOR();
}
```

**Remember to register your custom type, which should be done in the load method as follows:**

```cpp
load() {
    custom_types::Register::RegisterTypes<OurNamespace::OurClass*>();
}
```

Voila, we're done.

What about adding MonoBehaviours now? Well, it's simple too.

```cpp
UnityEngine::GameObject* go = getGameObjectSomehow(); // This is a placeholder, you'll be fixing this in your code

go->AddComponent<OurNamespace::OurClass*>(); // The * is necessary
```

And now our component is in the game.

Note that because we are defining a new method for construction (the C# ctor), we are not calling our C++ constructor. This means that our fields are uninitialized, including all calls to `DECLARE_INSTANCE_FIELD_DEFAULT` and non trivially constructible fields. We can fix this by calling the C++ constructor ourself, IN our C# one (ctor), via INVOKE_CTOR.

What does `INVOKE_CTOR` and `DECLARE_SIMPLE_DTOR` do behind the scenes? Well, first `INVOKE_CTOR` calls your C++ constructor at the cost of a _**very tiny**_ performance impact to initialize your fields. You do not need this call if you do not have [non-trivial constructible fields](https://en.cppreference.com/w/cpp/language/default_constructor) such as `std::vector`. `DECLARE_SIMPLE_DTOR` on the other hand causes the C++ destructor to be called by the C# destructor, which _in theory_ should have no memory leaks 🤞. Of course, if you have manually allocated data, you'll need your own destructor.

> Tip: If you plan on making a C# constructor just to invoke `INVOKE_CTOR`, you can use `DECLARE_DEFAULT_CTOR` and `REGISTER_SIMPLE_CTOR()` the same way you would use `DECLARE_DEFAULT_DTOR` and `REGISTER_DEFAULT_DTOR()` respectively.

You should **not** call either C# or C++ destructor outside of the destructor itself (e.g, just calling it anywhere in your code)
The GC will call it for you when it is no longer needed (do note that this does not apply to manually managed il2cpp created types)

### Custom destructor

This is a simple example using a custom destructor: (kindly provided by sc2ad)

```hpp
// type.hpp
#pragma once

DECLARE_CLASS_CODEGEN(Does, Stuff, Il2CppObject,
  std::vector<int> aCppVec;
  DECLARE_CTOR(ctor);
  DECLARE_DESTRUCTOR(dtor);
  REGISTER_FUNCTION(
    REGISTER_METHOD(ctor);
    REGISTER_METHOD(dtor);
  )
)
```

```cpp
// type.cpp
DEFINE_TYPE(Does::Stuff);

void Does::Stuff::ctor() {
  // create vector
  aCppVec = std::vector<int>();
}

void Does::Stuff::dtor() {
  // explicitly call the destructor, this almost always shouldn't be done outside of the destructor itself
  aCppVec.~vector();
}
```

## Harmony Patch to BS-Hooks

As you may already know, on Quest we do not have such things as post fix or pre fix like HarmonyPatches. So, how do we follow the same behaviour?

We use `MAKE_HOOK_OFFSETLESS(hookName, returnType, instance, parameters...)` to define the hook and it's code and `INSTALL_HOOK_OFFSETLESS(getLogger(), hookName, il2cpp_utils::FindMethodUnsafe("NameSpaceOfClass or empty if GlobalNamespace", "Class", "Method", parameterCount));` to register the hook.
You should NOT uninstall hooks, and beware for methods too small to be hooked. Ocassionally, there are methods in the game you CANNOT hook since they are too small, however you can workaround it by hooking other methods that ARE big enough.

Hook names are inconsequential, however I personally believe hook names should be as so: ```Class_Method```. If you hook the same method with different parameters, just suffix with a random number.

My recommended hook structure for mods is as follows (though always feel free to follow your own):
- include folder
  - ModName.hpp
    ```hpp
    #pragma once

    namespace ModName {
      namespace Hooks {
        void Class();
      }

      void InstallHooks();
    }
    ```
- src folder
  - hooks folder
    - Class.cpp
      ```cpp
      #include ModName.hpp
      
      MAKE_HOOK_OFFSETLESS(Class_Method, returnType) {

      } 

      void ModName::Hooks::Class() {
        INSTALL_HOOK_OFFSETLESS(getLogger(), Class_Method, il2cpp_utils::FindMethodUnsafe("NameSpaceOfClass or empty if GlobalNamespace", "Class", "Method", parameterCount))
      }
      ```
  - ModName.cpp
    ```cpp
    #include "ModName.hpp"

    void ModName::InstallHooks() {
      ModName::Hooks::Class();
    }
    ```

P.S I learned this from StackDoubleFlow, thanks buddy ;)

## Pre fix

TODO:

### Post Fix

Let's take a look at Kaitlyn's SoundExtensions mod as an example (thanks for letting me use your mod for this)

Let's look at this [post fix harmony patch](https://github.com/nyamimi/SoundExtensions/blob/518624e607c7660ae40b03ce10a25859d377b9f7/SoundExtensions/HarmonyPatches/StandardLevelScenesTransitionSetupDataSO.cs)

Essentially, what this patch does is the following in order:

- Hooks into `StandardLevelScenesTransitionSetupDataSO.Init`
- After `StandardLevelScenesTransitionSetupDataSO.Init` is called, grabs the instance field parameters and `SoundExtensionsController.Instance.Init(difficultyBeatmap, previewBeatmapLevel);` is called

Now, to recreate this in C++ is quite simple (we'll ignore the fact we don't have the SoundExtensionsController class)
First, we will look at `GlobalNamespace/StandardLevelScenesTransitionSetupDataSO.hpp` and find the Init method.

[Here](https://github.com/sc2ad/BeatSaber-Quest-Codegen/blob/fc7ad7e6e7c65210251f2665f628740d612c5157/include/GlobalNamespace/StandardLevelScenesTransitionSetupDataSO.hpp#L118), we see that it has 10 parameters. As you may already know, we have to make our hook have ALL 10 parameters. It also returns void, making it easy to do a post fix (though with methods that don't it's still easy)

```cpp
#include "main.hpp" // Include main for hooks, avoid problems by including the headers directly

#include "GlobalNamespace/StandardLevelScenesTransitionSetupDataSO.hpp"

MAKE_HOOK_OFFSETLESS(StandardLevelScenesTransitionSetupDataSO_Init, void, GlobalNamespace::StandardLevelScenesTransitionSetupDataSO* self,
                    Il2CppString* gameMode, GlobalNamespace::IDifficultyBeatmap* difficultyBeatmap, GlobalNamespace::IPreviewBeatmapLevel* previewBeatmapLevel, GlobalNamespace::OverrideEnvironmentSettings* overrideEnvironmentSettings, GlobalNamespace::ColorScheme* overrideColorScheme, GlobalNamespace::GameplayModifiers* gameplayModifiers, GlobalNamespace::PlayerSpecificSettings* playerSpecificSettings, GlobalNamespace::PracticeSettings* practiceSettings, ::Il2CppString* backButtonText, bool useTestNoteCutSoundEffects
                     ) {
  // Anything here is pre fix since it is running before the original method

  // We call the original method by putting the name of our hook, self (which is the instance) as the first parameter and the rest of the parameters.
  StandardLevelScenesTransitionSetupDataSO_Init(self, gameMode, difficultyBeatmap, previewBeatmapLevel, overrideEnvironmentSettings, overrideColorScheme, gameplayModifiers, playerSpecificSettings, practiceSettings, backButtonText, useTestNoteCutSoundEffects);
  // Anything after this is running after the original method runs, just as post fix would.
  SoundExtensionsController::Instance.Init(difficultyBeatmap, previewBeatmapLevel);
}


void InstallHook() {
  // This will register our hook
  INSTALL_HOOK_OFFSETLESS(getLogger(), StandardLevelScenesTransitionSetupDataSO_Init, il2cpp_utils::FindMethodUnsafe("", "StandardLevelScenesTransitionSetupDataSO", "Init", 10));
}
```

This does the following in order:

- Installs the hook
- When the hook runs, it will call the original method.
- Once the original method runs, you will run your code

What if the method you're hooking does NOT return void? It's simple to post fix too (and even change the result)

```cpp
#include "main.hpp" // Include main for hooks, avoid problems by including the headers directly

#include "GlobalNamespace/TrackLaneRingsRotationEffect.hpp"

MAKE_HOOK_OFFSETLESS(TrackLaneRingsRotationEffect_GetFirstRingRotationAngle, float, GlobalNamespace::TrackLaneRingsRotationEffect* self) {
  // Anything here is pre fix since it is running before the original method

  // We call the original method by putting the name of our hook, self (which is the instance) as the first parameter and the rest of the parameters.
  float result = TrackLaneRingsRotationEffect_GetFirstRingRotationAngle(self);
  // Anything after this is running after the original method runs, just as post fix would.

  // However, since this hook returns a variable, we cannot return null. So what can we do?

  float someVariable = 2;

  if (weWantToManipulateReturn) {
    // We can just add +2 to the return because we want to.
    // This does actually change the result of the original method and hooks that run after yours.
    return result + 2;
  } else {
    // Return the original
    return result;
  }
}


void InstallHook() {
  // This will register our hook
  INSTALL_HOOK_OFFSETLESS(getLogger(), TrackLaneRingsRotationEffect_GetFirstRingRotationAngle, il2cpp_utils::FindMethodUnsafe("", "TrackLaneRingsRotationEffect", "GetFirstRingRotationAngle", 0));
}
```

## Coroutines

In games, we might need to run code on the main thread later or at a specific interval. Usually, we keep track using state management such as counters or switches, though this can sometimes be hard to safely implement or just plain annoying. Unity attempts to alleviate the issue by using coroutines, which are "fake threads" as some call them. Your code will run in an Update method and every `yield return` will pause the method until the next Update. Your coroutine will end at the function's end or `yield break;`.
For more information about coroutines, take a look at the [Unity Coroutine docs](https://docs.unity3d.com/Manual/Coroutines.html)

However, as you may already know, not everything is as easy in C++ like C#. Luckily, this is one of those moments where C++ is on-par with syntax sugar and performance. Let's take a look at the following coroutine in C#:

```csharp
IEnumerator coroutine() {
  for (int i = 0; i < 30; i++) {
    // Timer
    secondsPassed++;
    doSomethingEverySecond();
    yield return new WaitForSeconds(1);
  }
}
```

Then you'd call it as `monoBehaviour.StartCoroutine(coroutine());`
This is a simple way to track time without having to measure it in every Update call yourself. We can also do this in C++, thanks to the wonderful work of `custom-types` with similar but different syntax. The following is a direct port of the C# coroutine from above, notice how it is very similar:

```cpp
#include "System/Collections/IEnumerator.hpp"
#include "custom-types/shared/coroutine.hpp"
#include "UnityEngine/WaitForSeconds.hpp"

custom_types::Helpers::Coroutine coroutine() {

    for (int i = 0; i < 30; i++) {
      // Timer
      secondsPassed++;
      doSomethingEverySecond();
      co_yield reinterpret_cast<System::Collections::IEnumerator*>(CRASH_UNLESS(WaitForSeconds::New_ctor(1)));
    }

    co_return;
}
```

and you'd run `monoBehaviour->StartCoroutine(reinterpret_cast<System::Collections::IEnumerator*>(CoroutineHelper::New(coroutine())));` akin to the C# code above

Congrats, you've just made a coroutine in C++!

What exactly are `co_return` and `co_yield` in C++ though? Well `co_return` in C++ is the direct port of `yield break;` for C#. `co_yield`, as it's name suggests, is a direct port to `yield return` for C#. You cannot use `return` in coroutines (which is a C++ 20 reason I don't want to delve into here)

_Do note that if your coroutine does NOT contain a `co_yield` or `co_return`, you WILL have LOTS of strange behaviour (believe me, I know it myself all too well). But you shouldn't use a coroutine if your method doesn't contain a `co_yield` or `co_return` to begin with, unless you're trying to run code in the Update method once (which would be weird)._

**Do note that the general rule of coroutines still apply here. You should avoid heavy work such as I/O or web requests on the main thread and instead use il2cpp threads (if you need to run il2cpp/Unity code in the thread) or use C++ threads for better performance. Coroutines are still on the main thread.**

## SafePtr

Il2Cpp is very fast, though don't fool yourself. One of it's major reasons it's fast on the Quest devices is due to the way it's GC works and how agressive it is. This is good for game developers since it allows them to not worry about memory management while using il2cpp, but it does bring an issue to the table for modders: how do _we_ as modders ensure the GC does not delete the memory we are using? We come to the problem by having stored pointers that lose their references and get GC'ed. Another cause for this problem is when we instantiate an object but it gets GC'ed before it even finishes instantiating such as it is with `UnityEngine::ScriptableObject::CreateInstance<>();` for example.

We **have** to tell the GC there is a reference to it existing so it doesn't get freed. One way to do this is to declare the pointer, register and store it in `custom_types`. This might be tedious and annoying, especially when you only need to pass around the variable through functions or less places.

SafePtr is smart pointer similar to `shared_ptr` and `unique_ptr` which can alleviate this problem. It does this by forcing a reference in il2cpp so the GC never frees it. Once SafePtr goes out of scope and it's destructor is called, this reference is freed and therefore gone, allowing the GC to free the pointer if there are no other references anymore.

There are some caveats however that you should be aware of:

- This does NOT stop the pointer from being assigned nullptr, that is fundamentally different from it being freed. A pointer points to a region of memory in C++, which means that if it is freed, it still points to that memory. It is just now invalid, and there is currently no way to check if a pointer is still valid. When a pointer is assigned nullptr, that memory is not freed. It just means that the pointer now points to nothing in memory.
- Do not use SafePtr everywhere, especially in static variables with infinite lifetime if you're not careful. This is a perfectly good use case where you may want to ensure that the memory will never be freed, however this is technically a memory leak and if you're not careful you'll cause more problems than you'll solve.

Be sure to use a SafePtr carefully, it is not a holy grace or a silver bullet. It is a specific tool for solving specific problems, be wise with it.

It should also be known that components and game objects shouldn't be used with SafePtr as those are fundamentally supposed to have their lifetimes tied to the game itself and destroyed when needed. Nothing is stopping you from using it, it's just bad practice fundamentally.


## Diagnosing crashes

On Quest you'll notice that it's far less forgiving for mistakes. Your mod will crash even for the slightest error, and sometimes you might even spend hours scratching your head, wondering why your code isn't working when it works in the original mod.

One of the best ways to understand the problem is by using tombstones. Tombstones are files created when the game crashes and are stored in `sdcard/Android/data/com.beatgames.beatsaber/files`. In conjuction with the tombstone, we can use a script that will try it's best to parse it and refer back to your source code, making it look more like a C# stacktrace.
Simply use the `ndk-stack.ps1` script as follows:

`ndk-stack.ps1 /path/to/tombstone`

and then you'll have `path/to/tombstone_unstripped.txt`. The new log _should_ reference your source code now, assuming your mod was to blame for the crash.

You will also notice that they contain something along the lines of `signal (Code) (ERROR_NAME)` or nullptr dereference as example:
`signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x7753b8faf0`

You can read the [Android NDK docs](https://source.android.com/devices/tech/debug/native-crash) for more detailed information about crashes and their meaning:

Here are some common codes and their meanings (though they often depend)

**Do note, that you should be skeptical with these explanations. Some crashes can be caused for other reasons such as ACCERR caused by nullptr for example**

### Nullptr dereference (the equivalent of NullPointerException on C#)

It means that you tried to use a ptr which is equal to nullptr

A very simple way to check if it's null is:

```cpp
A* ptr = nullptr;
if (ptr) {
  // YAY IT'S NOT NULLPTR
} else {
  // NO IT'S NULLPTR!!!!
}
```

### Nullptr dereference/SEGV_MAPERR (but this code shouldn't be null, what gives?)

One problem we have in Quest modding is that while il2cpp may be compiled to C++, it does bring a garbage collector. This garbage collector is very agressive due to the limitations of the Quest devices. As modders, this causes a problem in which if we try to store references to pointers, there is no guarantee the GC (garbage collector) won't delete the memory later while we use it. So what's our solution to this problem?

#### Custom-types

One of the answers lies again in custom-types. Even if you don't plan on extending a C# class, you can use custom-types to store pointers to C# classes. The GC will recognize the fields and won't delete them from memory as long as the instance is still alive.

#### SafePtr to avoid GC

[SafePtr](#SafePtr)

### SEGV_MAPERR (similar to ClassCastException or NullPtr)

This usually means that you are assuming your variable is of type `B*` but in reality it is `A*`. Since you are assuming it's `B*`, the memory or functions you are trying to access does not exist therefore you get a MAP ERROR (memory isn't mapped as you'd expect).

SEGV_MAPERR strictly speaking is a crash that occurs due to a pointer dereference to an unmapped region of memory. [You can read the Android crash docs for more details.](https://source.android.com/devices/tech/debug/native-crash#lowaddress)

Another reason why this crash may occur is due to the [GC which is explained more thoroughly here](#nullptr-dereferencesegv_maperr-but-this-code-shouldnt-be-null-what-gives).

The best way to check your classes before assuming/casting them is to do a simple if check as follows:

```cpp
// right is the class
// left is the parent class
if (il2cpp_functions::class_is_assignable_from(classof(B*), objectA->klass)) {
    auto *objectB = reinterpret_cast<B*>(objectA);
}
```

However, it should be noted that if the GC yeets the memory in the pointer, it will usually throw a [SEGV_MAPERR and it cannot be at checked at runtime (yet)](nullptr-dereference-segv_mapperr-but-this-code-shouldnt-be-null-what-gives). Instead of checking to see if it's been yeeted, you should instead try to avoid it altogether by making a custom-type or SafePtr for it.

### SIGABRT (intentional crash)

There are two main ways this crash occur (though there are many others, these are the ones I'll focus on here)

- An exception was thrown in the C# code (either called by your mod or another mod) and wasn't caught
  - Usually contains a `cxa_throw` in the callstack (though it varies)
  - Can be caught using `Il2CppExceptionWrapper&` in C++ code; though you should most of the time attempt to fix what causses it, not avoid it.
- A C++ exception was thrown and wasn't caught
  - Usually contains a `std::terminate` in the callstack (though it varies)
  - Most of the time (though not limited to) done intentionally by `CRASH_UNLESS/SAFE_ABORT`

This crash is unique in that a tombstone alone won't help you (it will show you the location, but not the reason)

When the crash occurs, a crash message will be printed before it explaining why it crashed (maybe a C# exception for example). This message is not captured by the tombstone (and it's a system service so we can't do it ourselves). You will have to manually log using `adb logcat` and reproduce the bug to get the message.

You cannot fix it easily either. You either fix your CRASH\_UNLESS condition or fix your C# method call, depending what is the cause.
You _can_ try to catch the exception, though this is undefined behaviour and we don't support it _yet_ in the community.

### ACCERR

While this crash is a rare error, it can occur by attempting to access memory which is not allowed. The GC is a probable cause, though it can also be bad casting. For example, it can be caused by accesing memory in the kernel spacce or `.rodata` section, which is usually prohibited.

_This is rather vague since I'm not very familiar with the crash myself_

## Optimizations

There's many ways we can optimize our mod to be as fast, or even _faster_ than the PC counterpart.

### Recreate/Replace functions

If the function is simple, such as a math method, you are _**encouraged**_ to use a C++ implementation of it. Either by the `std` library or rewriting it yourself. While the code is functionally similar, you avoid the overhead il2cpp incurs and these minor changes make a big difference in the long run.

### Use C++ alternative types

This is similar to the previous method, except instead of methods we use C++ native types or structs.

#### List/Arrays

We may use `std::vector` instead of a C# array list or array.

#### Hash Maps

We can use a `std::unordered_map` for hash maps (though they are not insertion order)

### C++ threads

Do not use if you need to access Unity/il2cpp in the thread
TODO:

### Lazy Initialize

In C++, we have the ability of doing the following:

```cpp
void func() {
  static auto someExpensiveVar = someExpensiveMethod();
}
```

What does this do? It initializes the variable the first time the method runs, and keeps it for every other run. This makes the variable constant however it is very helpful for improving performance. Avoid using it unless you are certain the lifetime of the object lasts as long or longer than you need.
