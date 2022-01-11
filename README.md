# hdtSMP for Skyrim Special Edition

Fork of [version](https://github.com/aers/hdtSMP64) by aers, from
[original code](https://github.com/HydrogensaysHDT/hdt-skyrimse-mods) by hydrogensaysHDT

## Changes 

+ Added maximum angle in ActorManager. A max angle can be used to specify physics on NPCs within a field of view.
  0 degrees represents straight in front. Default is 45 which is treated as + or - 45 degrees so 90 total degrees. 
  180 would be all around.
+ Added distance check in ActorManager to disable NPCs more than a certain distance from the player. This
  resolves the massive FPS drop in certain cell transitions (such as Blue Palace -> Solitude). Default
  maximum distance is 10000, which resolves that issue, but I recommend something around 2000 for better
  general performance in cities.
+ Added can-collide-with-bone to mesh definitions, as a natural counterpart to no-collide-with-bone.
+ Added new "external" sharing option, for objects that should collide with other NPCs, but not this one.
  Good for defining the whole body as a collision mesh for interactions between characters.
+ Significant refactoring of armor handling in ActorManager, to be much stricter about disabling systems and
  reducing gradual FPS loss.
+ Changed prefix mechanism to use a simple incrementing value instead of trying to use pointers to objects.
  Previously this could lead to prefix collisions with a variety of weird effects and occasional crashes.
+ Skeletons should remain active as long as they are in the same cell as (and close enough to) the player
  character. Resolves an issue where entering the Ancestor Glade often incorrectly marked skeletons as
  inactive and disabled physics.
+ Added "smp list" console command to list tracked NPCs without so much detail - useful for checking which
  NPCs are active in crowded areas. NPCs are now sorted according to active status, with active ones last.
+ New mechanism for remapping mesh names in the defaultBBPs.xml file, allowing much more concise ways of
  defining complex collision objects for lots of armors at once.
+ The code to scan defaultBBPs.xml can now handle the structure of facegen files, which means head part
  physics should now work (with limitations) on NPCs without having to manually edit the facegen data.
+ New bones from facegen files should now be added to the head instead of the NPC root, so they should now be
  positioned correctly if there is no physics for them or after a reset.

## Note about NPC head parts

Head parts work fine for NPCs without valid facegen data, but this isn't very useful because it triggers the
infamous dark face bug. Special restrictions apply to NPCs that do have facegen data:

+ Only one XML file can be used per face, even if was built from multiple physics-enabled parts.
+ NiStringExtraData nodes aren't automatically copied into the facegen file, so physics won't work
  automatically. Either do it manually in NifSkope or map one of the head parts to a file in defaultBBPs.xml.
+ Bones that aren't explicitly referenced by any mesh are removed when facegen data is generated, and can't
  be used as kinematic objects in constraints. Replace references to these with the NPC head (which should
  always be present). You may also need to set the frame of reference origin to the position of the missing
  bone relative to the head to get correct constraint behavior.

## Coming soon (maybe)

+ Reworked tag system for better compartmentalized .xml files.
+ More parallelism on collision checking.

## Known issues

+ Several options, including shape and collision definitions on bones, exist but don't seem to do anything.
+ Sphere-triangle collision check without penetration defined is obviously wrong, but fixing the test
  doesn't improve things. Needs further investigation.
+ Smp reset doesn't reload meshes correctly for some items (observed with Artesian Cloaks of Skyrim).
  Suspect references to the original triangle meshes are being dropped when they're no longer needed. We
  could keep ownership of the meshes, but it seems pretty marginal and a waste of memory. It also breaks
  some constraints for NPCs with facegen data.
+ It's not possible to refer to bones in facegen data that aren't used explicitly by at least one mesh. Most
  HDT hairs won't work as non-wig hair on NPCs without altering constraints. Probably possible but annoying
  to fix.
+ Probably any open bug listed on Nexus that isn't resolved in changes, above. This list only contains
  issues I have personally observed.

## Build Instructions

These should be complete instructions to set up and build the plugin, without assuming prior coding
experience. I'm checking everything out into D:\Dev-noAVX and building without AVX support, but of course you
can use any directory.

You will need:
+ Visual Studio 2019 (any edition)
+ Git
+ CMake

Open a VS2019 command prompt ("x64 Native Tools Command Prompt for VS2019"). Download and build Detours and
Bullet:

```
d:
mkdir HDT-SMP
cd HDT-SMP
git clone https://github.com/microsoft/Detours.git
git clone https://github.com/bulletphysics/bullet3.git
cd Detours
nmake
cd ..\bullet3
cmake .
```

If you want AVX support in Bullet, use `cmake-gui` instead of `cmake`, and check the `USE_MSVC_AVX` box
before clicking Configure then Generate. This should give a fairly significant performance boost if your CPU
supports it.
+ Note that AVX support in Bullet and the HDT-SMP plugin itself are independent configuration options. Enable
  it in both for maximum performance; disable it in both for maximum compatibility.

Open D:\HDT-SMP\bullet3\BULLET_PHYSICS.sln in Visual Studio, select the Release configuration, then 
Build -> Build solution.

Download skse64_2_01_05.7z and unpack into Dev-noAVX (source code is included in the official distribution),
then get the HDT-SMP source:

```
cd D:\HDT-SMP\skse64_2_01_05\src\skse64
git init
git remote add origin https://github.com/Karonar1/hdtSMP64.git
git fetch
git checkout master
```

Open D:\HDT-SMP\skse64_2_01_05\src\skse64\hdtSMP64.sln in Visual Studio. If you are asked to retarget
projects, just accept the defaults and click OK.

Open properties for the hdtSMP64 project. Select "All Configurations" at the top, and the C/C++ page. Add the
following to Additional Include Directories (just delete anything that was there before):
+ D:\HDT-SMP\Detours\include
+ D:\HDT-SMP\bullet3\src
+ D:\HDT-SMP\skse64_2_01_05\src

On the Linker -> General page, add the following to Additional Library Directories:
+ D:\HDT-SMP\bullet3\lib\Release
+ D:\HDT-SMP\Detours\lib.X64

Open properties for the skse64 project. Select General, and change Configuration Type from "Dynamic Library
(.dll)" to "Static Library (.lib)".

Make the following changes to the SKSE code:

at line 36 (the end of the list of class declarations), add:
```cpp
class BSDynamicTriShape;
class BSFadeNode;
```

In NiObjects.h, at line 81, delete:
```cpp
virtual void			* Unk_05(void);
```

and replace it with:
```cpp
virtual BSFadeNode		* GetAsBSFadeNode(void);
```

At line 88, delete:
```cpp
virtual void			* Unk_0C(void);
```

and replace it with:
```cpp
virtual BSDynamicTriShape	* GetAsBSDynamicTriShape(void);
```
And at line 172 (inside the `enum` definition), add:
```cpp
kNone =		0,
```

In GameEvents.h, at line 667, delete:
```cpp
EventDispatcher<void>								unk840;					//  840 - sink offset 0C8
```

and replace it with:
```cpp
EventDispatcher<TESMoveAttachDetachEvent>			unk840;					//  840 - sink offset 0C8
```

In GameMenus.h, befor line 1098 (just before the GetSingleton declaration), add:
```cpp
bool IsGamePaused() { return numPauseGame > 0; }
```

For SKSE64 2.1.5: open skse64.sln with a text editor, look for "common/common" and replace it by "common".

Close Visual Studio and open D:\HDT-SMP\skse64_2_01_05\src\skse64\hdtSMP64.sln in Visual Studio.
Open the properties for the different projects (Alt+F7), and for those with an Post-build event (skse64, skse64_loader, skse64_steam_loader), disable it by setting Yes to No.
Select Release and x64; then generate the solution.
For SKSE 2.1.5, ignore the errors concerning the projects with "loader" in their name.

Close Visual Studio and reopen the hdtSMP64.sln file.
Depending on your edition, CPU and GPU, select your configuration (ex: AE_CUDA_AVX), x64 and Build -> Rebuild.

## Credits

hydrogensaysHDT - Creating this plugin

aers - fixes and improvements

ousnius - some fixes, and "consulting"


