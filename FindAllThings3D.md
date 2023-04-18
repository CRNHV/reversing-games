# Old School RuneScape Steam version - Finding all things 3D | Reversing Games
  
Welcome to the third part of my OSRS reversing series, today weâ€™re finding all the things, kinda.

Weâ€™ll be looking at actually iterating what Iâ€™ve called the render list, which seems to be the best approach for finding most things with the notable exception of players, NPCs, and most likely projectiles. Theyâ€™re all present in the list of course, but at least externally (i.e. reading process memory rather than injecting code and hooking stuff) the data is very inconsistent due to the objects being cleared and added again every frame. Ground items and other objects on the other hand are only added once and are quite static until removal.

This means that externally this is a great source for world objects that arenâ€™t updated on every frame and internally it might be your one stop shop for literally everything you need if you hook the right part of the game.

I recommend looking at the first entry in this series to see how you can use ReClass to inspect data and how to navigate Ghidra a bit if itâ€™s unfamiliar.

Tools used: [ReClass.NET](https://web.archive.org/web/20210613131601/https://github.com/ReClassNET/ReClass.NET), [x64dbg](https://web.archive.org/web/20210613131601/https://x64dbg.com/), [Ghidra](https://web.archive.org/web/20210613131601/https://ghidra-sre.org/) and a modified [coltonons D2DOverlay for prototyping](https://web.archive.org/web/20210613131601/https://github.com/coltonon/D2DOverlay)

Contents of this blogpost are based on the 26.2.2021 build of osclient.exe

Previous parts:

[Part 1: Old School RuneScape Steam version - Players and NPCs](https://web.archive.org/web/20210613131601/https://reversing.games/jekyll/update/2021/03/05/osrs-player-npc-list.html)

[Part 2: Old School RuneScape Steam version - Overhead text, world to screen, and ground items](https://web.archive.org/web/20210613131601/https://reversing.games/jekyll/update/2021/03/23/osrs-overheadtext-world2screen-grounditems.html)

Identifying the common function for adding stuff to the render list
-------------------------------------------------------------------

Suppose we have our render\_player and render\_npcs functions from part 1, how do we find and verify which function inside those functions is responsible for adding stuff to the render list? Because if weâ€™re lucky it may be used by things other than players or npcs.

We can use our good friends x64dbg and Ghidras function call tree here to help.

![Ghidra function call tree](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-outgoingcalls.png) _Ghidra function call tree_

We can go through each outgoing function and patch out the function calls if they seem to be doing something more than just reading data. FUN\_000566c0, FUN\_0012b530 and FUN\_000ace00 seem to be doing nothing more than reading object data in various ways and returning it, so I doubt those are responsible for any rendering. We also know which function is height\_adjustment from the part 2 post so itâ€™s not that. That leaves us with the candidates FUN\_000f8410 and FUN\_000f8290.

Debugging disclaimer from part 2:

``

Letâ€™s attach x64dbg to osclient.exe, then letâ€™s hit ctrl+g to navigate to `osclient:base+f8410` and weâ€™ll arrive at the beginning of FUN\_000f8410.

Double click the instruction `mov ...` to patch the instruction and in the Assemble window type in the instruction `ret` and hit OK.

![Assemble window of x64dbg](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-assemble.png) _Assemble window of x64dbg_

And nothing happens. Thatâ€™s disappointing, but letâ€™s try the same steps again but for FUN\_000f8290.

![NPCs and players have disappeared](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-disappeared.png) _NPCs and players have disappeared after the second patch_

Players and npcs have disappeared, but game objects are still there after switching worlds. Switching worlds is recommended between patching stuff like this because it will cause a full reload of the game world.

Press CTRL+P to open the Patches window in x64dbg, and under the Patches list hit restore selected to remove the patches we made to restore normal game operation.

Iâ€™ll relabel FUN\_000f8290 to renderlist\_add\_character. Now letâ€™s repeat these steps with the functions that renderlist\_add\_character call, which in this case would be FUN\_000f84f0.

![The only function called](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-renderlist_outgoing.png) _The only function called_

After patching the first instruction to be ret using the same steps as before and switching worlds, a lot was missing.

![A lot was missing now](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-nothing.png) _Only walls, ground decoration and doors are left_

Most interactable objects are gone, doors, ground decorations and walls are still there. But this is an important function, letâ€™s rename FUN\_000f84f0 to renderlist\_add\_object.

You may want to restore the patches now so you will actually find data in the next section.

Figuring out the render list
----------------------------

NOTE: In the following parts Iâ€™m going to be skipping some less important stuff thats not strictly required for iterating the render list due to length reasons. You can use x64dbg or ReClass to find out more about the parameters or variables, refer to part 2 for a quick look in to how to log parameters with x64dbg.

Letâ€™s look at the beginning of the renderlist\_add\_object function to see whats happening there.

``

Letâ€™s go back a few function calls to figure out what some of the parameters are for now. In renderlist\_add\_character weâ€™re only really working with parameters from the earlier call, but in render\_player we see the parameters being the following.

``

At a quick glance for renderlist\_add\_character some of the parameters are the following: param\_1 is the pointer from DAT\_018860a0, param\_2 is the floor\_num, and if you have read part 1 you may realize that param\_3 is the x position and param\_4 is the y position, for now I donâ€™t know what the others are but I just want a rough idea.

After renaming the parameters in renderlist\_add\_character letâ€™s look at the parameters for renderlist\_add\_object.

``

param\_1 and param\_2 are once again the pointer and floor\_num, based on the code the variables used for param\_3 and param\_4 are pos\_x and \_y adjusted in various ways. But most importantly the `>> 7` on the line above the function call shows that they are in the form of tile coordinates. As tile coordinates are `coordinate >> 7` or alternatively `coordinate / 128` . Letâ€™s rename these parameters in renderlist\_add\_object and look at the code again.

``

The code may look like C-style variable cast hell, but what jumps out are the parts like this: `param_1 + 0xXXXX` so letâ€™s look at ReClass now to see whatâ€™s in those locations.

![param_1 + 5518](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-reclass.png) _Data from param\_1 + 5518 in ReClass_

So then, `param_1 + 0x552C` is 104, which is likely the currently loaded areas X dimension and `param_1 + 0x5530` is 104 and Y dimension respectively, similar how the ground item array was sized `104*104*floor_count` in the last part. Iâ€™ll also make a guess about `param_1 + 0x5528` being the max floor count but for that I have no idea how valid my guess is, but it seems about right.

`param_1 + 0x5548` seems to contain an array of pointers though. Iâ€™ve seen enough OSRS pointer arrays to know what all this means, weâ€™ve got another `104 * 104 * floor_count` array of pointer\[2\]â€™s on our hands.

Itâ€™s time for some ReClass work, set the type the pointer is pointing at to be an Array by clicking the blue arrows next to the variable name, then set the Array to be an array of Class instance by clicking the blue arrows, add 8 bytes to the new class and convert the nodes to be pointers. Then set the array size to \[10816\] by changing the number in the square brackets. You should hopefully get something like this:

![object array](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-reclass2.png) _Object array in the renderlist_

A common thing in the OSRS engine is that the second pointer is usually the one where the actual object starts. But another question is if this array is supposedly accessed by tile\_array\[x\_tile \* 104 + y\_tile\], how do we know which tiles to check in the array?

Here are some possibilities:

*   loop through all x and y coordinates (basic nested for loop) and draw coordinates for each tile with the help of the world\_to\_screen function from part 2
*   reverse the mouseover structure to get the details of whatever object is under your cursor (sorry, using this even though I havenâ€™t written about it)

Hereâ€™s some quick copypasta for that:

``

That should give us something a little like this:

![Mouseover](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-mouseover.png) _Mouseover tile overlay_

41,32 being the x,y coordinate and 4296 being 41 \* 104 + 32. So we can now plug in 4296 in the round brackets of the array in ReClass and look at the contents of ptr2.

![Tile structure overview](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-tile-ptr.png) _Tile structure overview_

I highlighted a few parts that jumped out at me first. +0x24 and +0x28 contain the tile coordinates (addition: 0x2c is floor), and + 0x98 was checked in the function we were looking at earlier and its 1 in this case. I changed the array index a bit and noticed that on tiles with no objects on them + 0x98 is 0 and a pair of pointers disappears from +0xC8.

![No object](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-tile-ptr-no-obj.png) _What the array looks like with no objects_

If we remember the second pointer rule, we need to investigate the pointer at +0xD0. I spent a while looking at different tiles, objects and stuff and managed to deduce this.

![object structure](https://web.archive.org/assets/Pics/OSRS_pictures/reclass-mystery-pointer.png) _Object structure if the array isnâ€™t empty_

pos\_z, pos\_x and pos\_y are the starting coordinate for the object. Say we have a 2x2 Tree object, every one of the tiles that the tree is on points to the same starting point. The Z is height\_adjusted so if you want to world\_to\_screen it you can make a version that skips height\_adjustment, or you can just recalculate the z. object\_ptr points to the original object I believe. So an npc, player, other object metadata etc.

For the object\_id although it seems to be a bit off, itâ€™s the same thing I did with the mouseover code above. The actual object id for the Tree object Iâ€™m looking at is 1276 which is 4FC in hexadecimal. I canâ€™t remember exactly which function I saw this in but the game pretty much does this somewhere around the mouseover code iirc: `objectid >> (0x14 & 0x3f)`

Letâ€™s try iterating this stuff then to see what all is contained in this array.

``

And the results are fine too.

![Overlaying all the objects for each tile](https://web.archive.org/assets/Pics/OSRS_pictures/objects.png) _Overlaying all the objects for each tile_

Itâ€™s not fast though, especially externally, so thats a puzzle for the reader. Sorry!

But you promised everything!
----------------------------

Yup. I did, at least where to find them.. Writing up how I discovered them or how they are structured would take another 500 words so Iâ€™ll leave it up to you at home to do for yourself! +0x88 contains a pointer to an array of ground item pointers (or close enough.), null if no items on the tile.

![Item array](https://web.archive.org/assets/Pics/OSRS_pictures/osrs-blog3-items.png) _Item stack array_

Walls, doors and bank booths etc. can be found at +0x58 with a structure similar but not exactly the same as what we found in +0xd0, though itâ€™s a single pointer and not an array, once again null if no walls etc. on the tile.

Thereâ€™s still a lot for me to still explore in these structures, but those are the most important parts to get started.

Conclusion
----------

It was once again a struggle to keep it short. I might have to make a proper ReClass.NET tutorial so that I can refer to that when writing.

I figured out I actually have to iterate this stuff (as of writing this on 30.3.) yesterday so Iâ€™ve not done a deep dive on it yet. My previous method for getting static objects was too inconsistent externally due to map chunks getting overwritten sometimes, oh well, I had reversed ground items, dynamic objects and static objects separately instead of just iterating the render list from the beginning when it was sitting there under my nose _facepalm_.

I think my next blogpost will be on how do you find item and object names without cache parsing, though once again you donâ€™t need to bother to do that as you can use cache data (see https://www.osrsbox.com for good resources). After that Iâ€™ll likely do one on scraping UI text and interfaces that contain items (e.g. inventory).

Feedback, complaints, whatever are easiest sent to [@alert\_insecure](https://web.archive.org/web/20210613131601/https://twitter.com/alert_insecure) on twitter or alternatively email atte@reversing.games if youâ€™re old school like that.
