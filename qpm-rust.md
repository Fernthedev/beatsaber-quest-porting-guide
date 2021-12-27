# QPM-Rust is here coming to a mod near you!
## What is qpm-rust?
QPM-Rust is a completely new implementation of the QPM CLI client built from the ground up in our beloved Rust language by RedBrumbler with contributions from others in the community. We consider it the future of QPM, calling it QPM v2. It is **not** compatible with projects designed with QPM v1 but it will feel very familiar and ergonomic.

QPM-Rust was built with the principle of improving everything about QPM v1 that made it uncomfortable. What new features and improvements does it bring? Well quite a few:
- Symlinks are much more reliable and work on Windows and Mac now
- Being written in rust, startup time is reduced to practically 0
- Statically linked and size optimized, now it is 3-6 MB instead of 70MB (including the runtime)
- Makes use of CMake and Ninja for much faster and reliable builds
- You can now generate qmod dependencies and data using `qpm-rust qmod create` and `qpm-rust qmod build`
- Supports providing NDK path through environment variable (`ANDROID_NDK_HOME`)
- Colored data for better readability and contrast
- Version range is more deterministic and strict
- 🐲

# How to migrate
To migrate, you'll have to download [qpm-rust](https://github.com/RedBrumbler/QuestPackageManager-Rust) in the GH Actions page.

Download [CMake](https://cmake.org/download/) and [Ninja](https://github.com/ninja-build/ninja/releases) and add them both to your enviroment PATH.

After that is done, copy the `CMakeLists.txt` from an existing mod repository. For the purpose of this guide, I'll be using the [Chroma CMake file](https://github.com/bsq-ports/Chroma/blob/main/CMakeLists.txt) however you can also use the [QMod template from Laurie](https://github.com/Lauriethefish/quest-mod-template/blob/main/template/CMakeLists.txt)

You'll also need to copy the `build.ps1`, `copy.ps1` and `buildQMOD.ps1` from preferrably Laurie's mod template. If you need Github Actions, you can copy them from these repositories:
- Header only: [Chroma](https://github.com/bsq-ports/Chroma/tree/main/.github/workflows)
- Linking: [Tracks](https://github.com/StackDoubleFlow/Tracks/tree/master/.github/workflows)
- Just Github release automation: [Noodle Extensions](https://github.com/StackDoubleFlow/NoodleExtensions/tree/master/.github/workflows)

Now run `qpm-rust restore` and attempt to build using `build.ps1`
If you have build errors related to `extern/`, run `qpm-rust cache legacy-fix`. You'll need to run restore again if you didn't use symlinks.

That should be everything to get you finished with the qpm-rust migration. For an example of what this looks like before/after, take a look at this [commit](https://github.com/bsq-ports/Chroma/commit/37853616e99fa572041fdbd11ad267fb8ed04ab4)