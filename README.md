# BeatSaber C# porting guide for Quest
This guide aims to do the following things:
- Be a complementary guide with [Laurie/Danrouse' modding guide](https://github.com/danrouse/beatsaber-quest-modding-guide) 
- Teach some _pointers_ on how to properly port mods (get it?)
- Discipline the reader with proper programming practices (no var p = 5)

This assumes that you have a basic understanding of C++ and a bit of C#. You will also be using codegen in this tutorial.
This guide will explain different practices in no particular order

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
#pragma once
#include "UnityEngine/GameObject.hpp"

UnityEngine::GameObject* go = UnityEngine::GameObject::Find(il2cpp_utils::createcsstr("name"));
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

    DECLARE_INSTANCE_FIELD_DEFAULT(float, floatVar, 0.1f); // default value 

    REGISTER_FUNCTION(OurClass,
      getLogger().debug("Registering OurClass!"); // May need to be removed
      REGISTER_METHOD(ctor);
      REGISTER_METHOD(Update);
    )
)
```
```cpp
// OurClass.cpp
#include "OurClass.hpp"

// Necessary
DEFINE_CLASS(OurNamespace::OurClass);

void OurNamespace::OurClass::ctor() {

}


void OurNamespace::OurClass::Update() {
  // Update method! YAY
}
```
** Remember to register your custom type, which should be done in the load method as follows: **
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

## Harmony Patch to BS-Hooks
TODO: Get examples and write actual code

## Diagnosing crashes
On Quest you'll notice that it's far less forgiving for mistakes. Your mod will crash even for the slightest error, and sometimes you might even spend hours scratching your head, wondering why your code isn't working when it works in the original mod.

One of the best ways to understand the problem is by using tombstones. Tombstones are files created when the game crashes and are stored in `sdcard/Android/data/com.beatgames.beatsaber/files`. In conjuction with the tombstone, we can use a script that will try it's best to parse it and refer back to your source code, making it look more like a C# stacktrace.
Simply use the `ndk-stack.ps1` script as follows:

`ndk-stack.ps1 /path/to/tombstone` 

and then you'll have `path/to/tombstone_unstripped.txt`. The new log _should_ reference your source code now, assuming your mod was to blame for the crash.

You will also notice that they contain something along the lines of `signal (Code) (ERROR_NAME)` or nullptr dereference as example:
`signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x7753b8faf0`

Here are some common codes and their meanings (though they often depend)
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

#### SafePtr
TODO: Wait for sc2ad to fix SafePtr

### SEGV_MAPERR (similar to ClassCastException or NullPtr) 
This usually means that you are assuming your variable is of type `B*` but in reality it is `A*`. Since you are assuming it's `B*`, the memory or functions you are trying to access do not exist therefore you get a MAP ERROR (memory isn't mapped as you'd expect) The best way to check your classes before assuming/casting them is to do a simple if check as follows:
```cpp
// left is the class
// right is the parent class
if (il2cpp_functions::class_is_assignable_from(objectA->klass, classof(B*))) {
    auto *objectB = reinterpret_cast<B*>(objectA);
}
```

However, it should be noted that if the GC yeets the memory in the pointer, it will usually throw a [SEGV_MAPERR and it cannot be at runtime checked (yet)](nullptr-dereference-segv_mapperr-but-this-code-shouldnt-be-null-what-gives). Instead of checking to see if it's been yeeted, you should instead try to avoid it altogether by making a custom-type or SafePtr for it.

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

You cannot fix it easily either. You either fix your CRASH_UNLESS condition or fix your C# method call, depending what is the cause.
You _can_ try to catch the exception, though this is undefined behaviour and we don't support it _yet_ in the community.

## Optimizations
There's many ways we can optimize our mod to be as fast, or even _faster_ than the PC counterpart.

### Recreate/Replace functions
If the function is simple, such as a math method, you are _**encouraged**_ to use a C++ implementation of it. Either by the `std` library or rewriting it yourself. While the code is functionally similar, you avoid the overhead il2cpp incurs and these minor changes make a big difference in the long run

### Use C++ alternative types
This is similar to the previous method, except instead of methods we use C++ native types or structs. 
#### List
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
