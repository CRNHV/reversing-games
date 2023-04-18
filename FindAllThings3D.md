# Old School RuneScape Steam version - Finding all things 3D | Reversing Games
  
Welcome to the third part of my OSRS reversing series, today we're finding all the things, kinda.

We'll be looking at actually iterating what I've called the render list, which seems to be the best approach for finding most things with the notable exception of players, NPCs, and most likely projectiles. They're all present in the list of course, but at least externally (i.e. reading process memory rather than injecting code and hooking stuff) the data is very inconsistent due to the objects being cleared and added again every frame. Ground items and other objects on the other hand are only added once and are quite static until removal.

This means that externally this is a great source for world objects that aren't updated on every frame and internally it might be your one stop shop for literally everything you need if you hook the right part of the game.

I recommend looking at the first entry in this series to see how you can use ReClass to inspect data and how to navigate Ghidra a bit if it's unfamiliar.

Tools used: [ReClass.NET](https://github.com/ReClassNET/ReClass.NET), [x64dbg](https://x64dbg.com/), [Ghidra](https://ghidra-sre.org/) and a modified [coltonons D2DOverlay for prototyping](https://github.com/coltonon/D2DOverlay)

Contents of this blogpost are based on the 26.2.2021 build of osclient.exe

Previous parts:

[Part 1: Old School RuneScape Steam version - Players and NPCs](/PlayersAndNpcs.md)

[Part 2: Old School RuneScape Steam version - Overhead text, world to screen, and ground items](/OverheadTextWorld2ScreenGroundItems.md)

Identifying the common function for adding stuff to the render list
-------------------------------------------------------------------

Suppose we have our render\_player and render\_npcs functions from part 1, how do we find and verify which function inside those functions is responsible for adding stuff to the render list? Because if we're lucky it may be used by things other than players or npcs.

We can use our good friends x64dbg and Ghidras function call tree here to help.

![Ghidra function call tree](/Images/Part3/osrs-blog3-outgoingcalls.png) _Ghidra function call tree_

We can go through each outgoing function and patch out the function calls if they seem to be doing something more than just reading data. FUN\_000566c0, FUN\_0012b530 and FUN\_000ace00 seem to be doing nothing more than reading object data in various ways and returning it, so I doubt those are responsible for any rendering. We also know which function is height\_adjustment from the part 2 post so it's not that. That leaves us with the candidates FUN\_000f8410 and FUN\_000f8290.

Debugging disclaimer from part 2:

```
The game doesn’t mind debuggers too much, but be wary of the fact that the game installs some mouse hooks and will cause issues while the game is paused. You may opt to deal with it by doing something about the hooks, or you just be confident about navigating menus with a keyboard in case the debugger pauses.
```

Let's attach x64dbg to osclient.exe, then let's hit ctrl+g to navigate to `osclient:base+f8410` and we'll arrive at the beginning of FUN\_000f8410.

Double click the instruction `mov ...` to patch the instruction and in the Assemble window type in the instruction `ret` and hit OK.

![Assemble window of x64dbg](/Images/Part3/osrs-blog3-assemble.png) _Assemble window of x64dbg_

And nothing happens. That's disappointing, but let's try the same steps again but for FUN\_000f8290.

![NPCs and players have disappeared](/Images/Part3/osrs-blog3-disappeared.png) _NPCs and players have disappeared after the second patch_

Players and npcs have disappeared, but game objects are still there after switching worlds. Switching worlds is recommended between patching stuff like this because it will cause a full reload of the game world.

Press CTRL+P to open the Patches window in x64dbg, and under the Patches list hit restore selected to remove the patches we made to restore normal game operation.

I'll relabel FUN\_000f8290 to renderlist\_add\_character. Now let's repeat these steps with the functions that renderlist\_add\_character call, which in this case would be FUN\_000f84f0.

![The only function called](/Images/Part3/osrs-blog3-renderlist_outgoing.png) _The only function called_

After patching the first instruction to be ret using the same steps as before and switching worlds, a lot was missing.

![A lot was missing now](/Images/Part3/osrs-blog3-nothing.png) _Only walls, ground decoration and doors are left_

Most interactable objects are gone, doors, ground decorations and walls are still there. But this is an important function, let's rename FUN\_000f84f0 to renderlist\_add\_object.

You may want to restore the patches now so you will actually find data in the next section.

Figuring out the render list
----------------------------

NOTE: In the following parts I'm going to be skipping some less important stuff thats not strictly required for iterating the render list due to length reasons. You can use x64dbg or ReClass to find out more about the parameters or variables, refer to part 2 for a quick look in to how to log parameters with x64dbg.

Let's look at the beginning of the renderlist\_add\_object function to see whats happening there.

```
// The 0x5548, 0x552c, 0x5530 were all interpreted as labels for some reason, I reformatted them here
param_5 = param_3 + param_5;
if (param_3 < param_5) {
    iVar13 = param_3;
    do {
        iVar11 = param_4;
        if (param_4 < param_4 + param_6) {
            do {
                if ((iVar13 < 0) || (iVar11 < 0)) {
                    return 0;
                }
                if (*(int *)(0x552c + param_1) <= iVar13) {
                    return 0;
                }
                if (*(int *)(0x5530 + param_1) <= iVar11) {
                    return 0;
                }
                lVar7 = *(longlong *)
                    (*(longlong *)(0x5548 + param_1) + 8 +
                     (longlong)
                     ((*(int *)(0x552c + param_1) * param_2 + iVar13) *
                      *(int *)(0x5530 + param_1) + iVar11) * 0x10); 
                if ((lVar7 != 0) && (4 < *(int *)(lVar7 + 0x98))) {
                    return 0;
                }
                iVar11 = iVar11 + 1;
            } while (iVar11 < param_4 + param_6);
        }
        iVar13 = iVar13 + 1;
    } while (iVar13 < param_5);
}
```

Let's go back a few function calls to figure out what some of the parameters are for now. In renderlist\_add\_character we're only really working with parameters from the earlier call, but in render\_player we see the parameters being the following.

```
uVar13 = DAT_018860a0;
uVar8 = FUN_000ace00(local_50);
renderlist_add_character
(uVar13,floor_num,*(undefined4 *)(lVar12 + 0x10),*(undefined4 *)(lVar12 + 0x14),
*(undefined4 *)(lVar12 + 0x3d8),0x3c,uVar8,*(undefined4 *)(lVar12 + 0x18),
local_60[0],
in_stack_ffffffffffffff70 & 0xffffff00 | (uint)*(byte *)(lVar12 + 0x1c),
&local_res20);
```

At a quick glance for renderlist\_add\_character some of the parameters are the following: param\_1 is the pointer from DAT\_018860a0, param\_2 is the floor\_num, and if you have read part 1 you may realize that param\_3 is the x position and param\_4 is the y position, for now I don't know what the others are but I just want a rough idea.

After renaming the parameters in renderlist\_add\_character let's look at the parameters for renderlist\_add\_object.

```
 else {
    iVar4 = pos_x - param_6;
    iVar5 = pos_y - param_6;
    iVar6 = pos_x + param_6;
    param_6 = pos_y + param_6;
    iVar2 = param_6;
    iVar3 = iVar6;
    if (param_10 != '\0') {
      iVar2 = param_6 + 0x80;
      if (0x2fe < param_8 - 0x281U) {
        iVar2 = param_6;
      }
      iVar3 = iVar6 + 0x80;
      if (0x2fe < param_8 - 0x481U) {
        iVar3 = iVar6;
      }
      if (0x500 < param_8 - 0x180U) {
        iVar5 = iVar5 + -0x80;
      }
      if (param_8 - 0x81U < 0x2ff) {
        iVar4 = iVar4 + -0x80;
      }
    }
    iVar4 = (int)((iVar4 >> 0x1f & 0x7fU) + iVar4) >> 7;
    iVar6 = (int)((iVar5 >> 0x1f & 0x7fU) + iVar5) >> 7;
    uVar1 = renderlist_add_object
                      (param_1,floor_num,iVar4,iVar6,
                       (((int)(iVar3 + (iVar3 >> 0x1f & 0x7fU)) >> 7) - iVar4) + 1,
                       (((int)((iVar2 >> 0x1f & 0x7fU) + iVar2) >> 7) - iVar6) + 1,pos_x,pos_y,
                       param_5,param_7,param_8,1,param_9,0,param_11);
  }
  ```

param\_1 and param\_2 are once again the pointer and floor\_num, based on the code the variables used for param\_3 and param\_4 are pos\_x and \_y adjusted in various ways. But most importantly the `>> 7` on the line above the function call shows that they are in the form of tile coordinates. As tile coordinates are `coordinate >> 7` or alternatively `coordinate / 128` . Let's rename these parameters in renderlist\_add\_object and look at the code again.

```
// The 0x5548, 0x552c, 0x5530 were all interpreted as labels for some reason, I reformatted them here
param_5 = tile_x + param_5;
if (tile_x < param_5) {
    _tile_x = tile_x;
    do {
        _tile_y = tile_y;
        if (tile_y < tile_y + param_6) {
            do {
                if ((_tile_x < 0) || (_tile_y < 0)) {
                    return 0;
                }
                if (*(int *)(0x552c + param_1) <= _tile_x) {
                    return 0;
                }
                if (*(int *)(0x5530 + param_1) <= _tile_y) {
                    return 0;
                }
                lVar7 = *(longlong *)
                    (*(longlong *)(0x5548 + param_1) + 8 +
                     (longlong)
                     ((*(int *)(0x552c + param_1) * floor_num + _tile_x) *
                      *(int *)(0x5530 + param_1) + _tile_y) * 0x10);
                if ((lVar7 != 0) && (4 < *(int *)(lVar7 + 0x98))) {
                    return 0;
                }
                _tile_y = _tile_y + 1;
            } while (_tile_y < tile_y + param_6);
        }
        _tile_x = _tile_x + 1;
    } while (_tile_x < param_5);
}
```

The code may look like C-style variable cast hell, but what jumps out are the parts like this: `param_1 + 0xXXXX` so let's look at ReClass now to see what's in those locations.

![param_1 + 5518](/Images/Part3/osrs-blog3-reclass.png) _Data from param\_1 + 5518 in ReClass_

So then, `param_1 + 0x552C` is 104, which is likely the currently loaded areas X dimension and `param_1 + 0x5530` is 104 and Y dimension respectively, similar how the ground item array was sized `104*104*floor_count` in the last part. I'll also make a guess about `param_1 + 0x5528` being the max floor count but for that I have no idea how valid my guess is, but it seems about right.

`param_1 + 0x5548` seems to contain an array of pointers though. I've seen enough OSRS pointer arrays to know what all this means, we've got another `104 * 104 * floor_count` array of pointer\[2\]'s on our hands.

It's time for some ReClass work, set the type the pointer is pointing at to be an Array by clicking the blue arrows next to the variable name, then set the Array to be an array of Class instance by clicking the blue arrows, add 8 bytes to the new class and convert the nodes to be pointers. Then set the array size to \[10816\] by changing the number in the square brackets. You should hopefully get something like this:

![object array](/Images/Part3/osrs-blog3-reclass2.png) _Object array in the renderlist_

A common thing in the OSRS engine is that the second pointer is usually the one where the actual object starts. But another question is if this array is supposedly accessed by tile\_array\[x\_tile \* 104 + y\_tile\], how do we know which tiles to check in the array?

Here are some possibilities:

*   loop through all x and y coordinates (basic nested for loop) and draw coordinates for each tile with the help of the world\_to\_screen function from part 2
*   reverse the mouseover structure to get the details of whatever object is under your cursor (sorry, using this even though I haven't written about it)

Here's some quick copypasta for that:

```
class mouseover_entry
{
public:
	int32_t x; //0x0000
	int32_t y; //0x0004
	char pad_0008[16]; //0x0008
	char* N00005815; //0x0018
	char* N0000089F; //0x0020
	char* N000008A0; //0x0028
	char pad_0030[8]; //0x0030
	char* N000008A2; //0x0038
	char* N000008A3; //0x0040
	char* N000008A4; //0x0048
	char pad_0050[16]; //0x0050
	int64_t objectid; //0x0060
	char pad_0068[24]; //0x0068
}; //Size: 0x0080
class mouseover_data
{
public:
	char pad_0000[688]; //0x0000
	int32_t actioncount; //0x02B0
	char pad_02B4[4]; //0x02B4
	mouseover_entry mouseover_entries[500]; //0x02B8
	char pad_FCB8[12]; //0xFCB8
	int32_t click_coord_x; //0xFCC4
	int32_t click_coord_y; //0xFCC8
	char pad_FCCC[124]; //0xFCCC
}; //Size: 0xFD48

...

uintptr_t mouseptr = 0;
memoryman.read(modulebase + 0x51e508, mouseptr);
mouseover_data mousedata;
memoryman.read(mouseptr, mousedata);
mouseover_entry entry = mousedata.mouseover_entries[1];
if (entry.objectid != -1)
{
    int x = entry.x * 128;
    int y = entry.y * 128;

    WORLD2SCREEN(x + 64, y + 64, 0, floor_num, &x, &y);
    int objectid = entry.objectid >> (0x14 & 0x3f);
    DrawString(std::to_string(entry.x) + "," + std::to_string(entry.y) + "," + std::to_string(entry.x*104+entry.y), 14, x, y, 1, 1, 1, 1, true);
    DrawString(std::to_string(objectid), 14, x, y + 18, 1, 1, 1, 1, true);
}
```

That should give us something a little like this:

![Mouseover](/Images/Part3/osrs-blog3-mouseover.png) _Mouseover tile overlay_

41,32 being the x,y coordinate and 4296 being 41 \* 104 + 32. So we can now plug in 4296 in the round brackets of the array in ReClass and look at the contents of ptr2.

![Tile structure overview](/Images/Part3/osrs-blog3-tile-ptr.png) _Tile structure overview_

I highlighted a few parts that jumped out at me first. +0x24 and +0x28 contain the tile coordinates (addition: 0x2c is floor), and + 0x98 was checked in the function we were looking at earlier and its 1 in this case. I changed the array index a bit and noticed that on tiles with no objects on them + 0x98 is 0 and a pair of pointers disappears from +0xC8.

![No object](/Images/Part3/osrs-blog3-tile-ptr-no-obj.png) _What the array looks like with no objects_

If we remember the second pointer rule, we need to investigate the pointer at +0xD0. I spent a while looking at different tiles, objects and stuff and managed to deduce this.

![object structure](/Images/Part3/reclass-mystery-pointer.png) _Object structure if the array isn't empty_

pos\_z, pos\_x and pos\_y are the starting coordinate for the object. Say we have a 2x2 Tree object, every one of the tiles that the tree is on points to the same starting point. The Z is height\_adjusted so if you want to world\_to\_screen it you can make a version that skips height\_adjustment, or you can just recalculate the z. object\_ptr points to the original object I believe. So an npc, player, other object metadata etc.

For the object\_id although it seems to be a bit off, it's the same thing I did with the mouseover code above. The actual object id for the Tree object I'm looking at is 1276 which is 4FC in hexadecimal. I can't remember exactly which function I saw this in but the game pretty much does this somewhere around the mouseover code iirc: `objectid >> (0x14 & 0x3f)`

Let's try iterating this stuff then to see what all is contained in this array.

```
struct ptr_pair
{
	uintptr_t ptr1;
	uintptr_t ptr2;
};
class renderlist_object
{
public:
	char pad_0000[4]; //0x0000
	int z; // different order than my vec3 struct, how annoying
	int x;
	int y;
	uintptr_t unk;
	class N00001346* opt_objectptr; //0x0018
	char pad_0020[4]; //0x0020
	uint32_t x_min; //0x0024
	uint32_t x_max; //0x0028
	uint32_t y_min; //0x002C
	uint32_t y_max; //0x0030
	char pad_0034[12]; //0x0034
	size_t object_id; //0x0040
	char pad_0048[320]; //0x0048
}; //Size: 0x0188

...

uintptr_t renderlist = modulebase + 0x18860a0;
memoryman.read(renderlist, renderlist);
memoryman.read(renderlist + 0x5548, renderlist);
std::vector<ptr_pair> ground(104 * 104 * 4);
memoryman.read(renderlist, ground[0], 104 * 104 * 4 * sizeof(ptr_pair));
for (auto& pair : ground)
{
    if (pair.ptr2)
    {
        int objectcount;
        vec3 pos;
        memoryman.read(pair.ptr2 + 0x24, pos);
        memoryman.read(pair.ptr2 + 0x98, objectcount);

        if (objectcount)
        {
            // It's an array of objectcount*ptr[2] starting at 0xc8 if there are more than 1 object
            std::vector<ptr_pair> objects(objectcount);
            memoryman.read(pair.ptr2 + 0xc8, objects[0], objectcount * sizeof(ptr_pair));
            for (auto& objectptr : objects)
            {
                if (objectptr.ptr2)
                {
                    renderlist_object render_object;
                    memoryman.read(objectptr.ptr2, render_object);

                    uint32_t object_id = render_object.object_id >> (0x14 & 0x3f);
                    vec2 screenpos;
					// 128 is the size of a single tile so to get real coord for tile coord just *128
                    // + 64 obviously for half a tile to get the center.
                    if (WORLD2SCREEN(pos.x*128+64, pos.y*128+64, 0, pos.z, &screenpos.x, &screenpos.y))
                    {
                        DrawString(std::to_string(object_id), 14, screenpos.x, screenpos.y, 1, 1, 1, 1, true);
                    }
                }
            }
        }
    }
}
```

And the results are fine too.

![Overlaying all the objects for each tile](/Images/Part3/objects.png) _Overlaying all the objects for each tile_

It's not fast though, especially externally, so thats a puzzle for the reader. Sorry!

But you promised everything!
----------------------------

Yup. I did, at least where to find them.. Writing up how I discovered them or how they are structured would take another 500 words so I'll leave it up to you at home to do for yourself! +0x88 contains a pointer to an array of ground item pointers (or close enough.), null if no items on the tile.

![Item array](/Images/Part3/osrs-blog3-items.png) _Item stack array_

Walls, doors and bank booths etc. can be found at +0x58 with a structure similar but not exactly the same as what we found in +0xd0, though it's a single pointer and not an array, once again null if no walls etc. on the tile.

There's still a lot for me to still explore in these structures, but those are the most important parts to get started.

Conclusion
----------

It was once again a struggle to keep it short. I might have to make a proper ReClass.NET tutorial so that I can refer to that when writing.

I figured out I actually have to iterate this stuff (as of writing this on 30.3.) yesterday so I've not done a deep dive on it yet. My previous method for getting static objects was too inconsistent externally due to map chunks getting overwritten sometimes, oh well, I had reversed ground items, dynamic objects and static objects separately instead of just iterating the render list from the beginning when it was sitting there under my nose _facepalm_.

I think my next blogpost will be on how do you find item and object names without cache parsing, though once again you don't need to bother to do that as you can use cache data (see https://www.osrsbox.com for good resources). After that I'll likely do one on scraping UI text and interfaces that contain items (e.g. inventory).

Feedback, complaints, whatever are easiest sent to [@alert\_insecure](https://twitter.com/alert_insecure) on twitter or alternatively email atte@reversing.games if you're old school like that.
