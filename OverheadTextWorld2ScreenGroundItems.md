# Old School RuneScape Steam version - Overhead text, world to screen, and ground items | Reversing Games
  
Welcome to the second part of my OSRS reversing series. In the last part we found out how trivial finding players and NPCs is. In todays blogpost we have another challenge though, how do we get screen coordinates for the world coordinates we got in the earlier findings? Thats a fundamental goal for any game reversing project so let's get it early, it will help with testing stuff and verifying results. You may want to ensure you have a way to draw 2D graphics in any way.

The world to screen findings also helped with finding the ground item objects, so we'll go over those as well.

I recommend looking at the first entry in this series to see how you can use ReClass to look at data at specific addresses in memory, as I may not be as thorough here with displaying the values in ReClass as that might just get repetitive.

Tools used: [ReClass.NET](https://github.com/ReClassNET/ReClass.NET), [x64dbg](https://x64dbg.com/) and [Ghidra](https://ghidra-sre.org/)

Other stuff used: a slightly modified version of [coltonons D2DOverlay for 2D graphics](https://github.com/coltonon/D2DOverlay), It's definitely not the best thing out there, but it was the most plug and play thing I found to get prototyping fastest as I personally had no previous projects that were fit for the task, you may use what you wish, either internal or external to verify results.

Contents of this blogpost are based on the 26.2.2021 build of osclient.exe

NOTE 23.3.2021:

I intended to have this post up earlier, but due to OVH's SBG2 datacenter fire my site was down and I really didn't want to spend the money on a new VPS to get the site up, so waiting it was.

Overhead text
-------------

After finding player and NPC iteration in FUN\_000a9400 I decided to take some time going through the functions called from there to see if they contained any similar object iterations, mainly in search of ground items and other things of interest. I ended up finding FUN\_00047080 which was doing some iteration using npc\_count+player\_count globals. But also something unknown to me, so I decided to look at the part I didn't recognize. Which looked a bit like this:

```
  local_2040 = 0;
  if (0 < DAT_0051d6c8) {
    local_2018 = 0;
    local_2038 = 0;
    do {
      local_2008 = (longlong **)((longlong)&DAT_0051da10 + local_2038);
      DAT_0043dd84 = *(int *)((longlong)&DAT_0051d940 + local_2038);
      DAT_0043dda0 = *(int *)local_2008;
      do {
        lVar16 = 0;
        bVar3 = false;
        if (local_2038 < 1) break;
        do {
```


I usually happen to be idling around the Grand Exchange while reversing which was convenient, as when I plugged DAT\_0051d6c8 in to ReClass it turned out to be the count of active overhead texts. I found this to be very promising, as where there is rendering text that is related to player positions, there should be a world to screen function. Let's look at DAT\_0051da10 and DAT\_0051d940 in ReClass then. At first I assumed it would be and index array again, but then I noticed the values aren't very static when I moved the camera, and looked like an array of coordinates instead. It's a safe guess, but this is why we have an overlay to verify things with. I put together the code below and checked it out.

```	

int text_count;
memoryman.read(modulebase + 0x51d6c8, text_count);
for (int i = 0; i < text_count; i++)
{
    vec2 coordtest;
    memoryman.read(modulebase + 0x51da10 + i * 0x4, coordtest.y);
    memoryman.read(modulebase + 0x51d940 + i * 0x4, coordtest.x);

    DrawBox(coordtest.x, coordtest.y, 40, 40, 1, 1, 1, 1, 1, true);
}
```

![](/Images/Part2/blog2-osrs-text_coords.png)

Yup, they were screen position arrays. DAT\_0051da10 being the array for the vertical positions and DAT\_0051d940 for the horizontal positions.

This means I will relabel the ghidra globals like this.

```
if (0 < overhead_text_count) {
    local_2018 = 0;
    index = 0;
    do {
        local_2008 = (longlong **)((longlong)&textpos_array_y + index);
        some_x_pos = *(int *)((longlong)&textpos_array_x + index);
        some_y_pos = *(int *)local_2008;
```

Although we don't know what some\_x\_pos and some\_y\_pos are yet, I decided to name them like that just so that I can spot them easier in the future. Investigating the textpos\_arrays further is where we will likely find world to screen functionality. Though lets look a bit further to find the actual contents of overhead text in case we need or want to use them for the future.

A bit lower in the loop I noticed the following and checked out DAT\_01708240

```
puVar12 = &DAT_01708240 + (longlong)local_2040 * 4;
```

![](/Images/Part2/blog2-pic3-reclass.png)

Yup, it's the array of overhead texts, I think it's time to investigate the coordinates though and not go any further.

World to screen
---------------

Let's start off by looking at the text position arrays references, to be specific the write references in this case as we want to know how the coordinates are determined.

![](/Images/Part2/blog2-pic4-references.png)

The reference at FUN\_00987c0:00098ce0 doesn't seem too interesting.

```
puVar10 = (undefined8 *)&textpos_array_y;
do {
    *puVar10 = 0;
    puVar10[1] = 0;
    puVar10[2] = 0;
    puVar1 = puVar10 + 8;
    puVar10[3] = 0;
    puVar10[4] = 0;
    puVar10[5] = 0;
    puVar10[6] = 0;
    puVar10[7] = 0;
    lVar13 = lVar13 + -1;
    puVar10 = puVar1;
} while (lVar13 != 0);
````

It's mostly writing zeroes so I doubt its what we are looking for. Let's take a look at what we find at FUN\_00047d90:0004868e next then.

```
(&textpos_array_y)[lVar16] = local_c0;
(&DAT_0051d870)[lVar16] = uVar4;
(&textpos_array_x)[lVar16] = iVar24;
```

This snippet seems _very_ interesting. Let's take a look at how local\_c0 and iVar24 are determined.

At the top of the function we find the following.

```
iVar24 = *param_3;
local_c0 = param_3[1];
```

So param\_3 is an integer array with the text position, let's look at where the function is called then to possibly see where the 3rd parameter comes from.

```
piVar4 = (int *)FUN_0004a170(local_70,param_3,param_4,plVar3);
text_pos = *piVar4;
iStack140 = piVar4[1];
iStack136 = piVar4[2];
iStack132 = piVar4[3];
local_80 = *(undefined8 *)(piVar4 + 4);
local_78 = piVar4[6];
FUN_00054a40(plVar3,local_50);
FUN_00049d70(param_1,local_50,&text_pos,2,param_3,param_4,param_5,param_6,param_7);
uVar5 = param_1;
FUN_00049d70(param_1,local_50,&text_pos,0,param_3,param_4,param_5,param_6,param_7);
FUN_00048710(uVar5,(longlong)plVar3,(longlong)&text_pos,param_3,param_4,param_5,param_6);
FUN_00047d90(uVar5,plVar3,&text_pos,'\x01',param_2,param_3,param_4,param_5,param_6);
FUN_00049d70(param_1,local_50,&text_pos,1,param_3,param_4,param_5,param_6,param_7);
  ```

I took the liberty to change the 3rd param of FUN\_00047d90 to be called text\_pos. And as we can clearly see we have to look at FUN\_0004a170.

```
int * FUN_0004a170(int *param_1,int param_2,int param_3,longlong param_4)

{
  int iVar1;
  int iVar2;
  
  *(undefined *)(param_1 + 6) = 0;
  FUN_000520b0(param_4);
  iVar2 = some_x_pos;
  iVar1 = *(int *)(param_4 + 0x148);
  param_1[1] = some_y_pos + param_3;
  *param_1 = iVar2 + param_2;
  FUN_000520b0(param_4,iVar1 / 2);
  param_1[2] = some_x_pos + param_2;
  param_1[3] = some_y_pos + param_3;
  FUN_000520b0(param_4,0xfffffff1);
  param_1[4] = some_x_pos + param_2;
  param_3 = some_y_pos + param_3;
  *(undefined *)(param_1 + 6) = 1;
  param_1[5] = param_3;
  return param_1;
}
```

param\_1 array is getting filled up after FUN\_000520b0 calls. We must, go, deeper!

```
void FUN_000520b0(longlong param_1,undefined4 param_2)

{
  FUN_000520c0(*(undefined4 *)(param_1 + 0x10),*(undefined4 *)(param_1 + 0x14),param_2);
  return;
}
```

FUN\_000520c0 might be it, assuming this is player or NPC related, keen eyed readers might remember that +0x10 and +0x14 of the player object are X and Y coordinates. Deeper we go!

```
void FUN_000520c0(int param_1,int param_2,int param_3)

{
  int iVar1;
  int iVar2;
  int iVar3;
  int iVar4;
  longlong lVar5;
  longlong lVar6;
  longlong lVar7;
  int iVar8;
  int iVar9;
  int iVar10;
  
  if ((((0x7f < param_1) && (0x7f < param_2)) && (param_1 < 0x3301)) && (param_2 < 0x3301)) {
    iVar4 = FUN_00052250();
    param_1 = param_1 - DAT_0051d0c8;
    param_2 = param_2 - DAT_0051d0cc;
    iVar9 = (iVar4 - param_3) - DAT_0051d0c0;
    lVar5 = FUN_000e8fb0();
    lVar7 = (longlong)DAT_0051d0a8;
    iVar4 = *(int *)(lVar5 + lVar7 * 4);
    lVar6 = FUN_000e8fc0();
    iVar1 = *(int *)(lVar6 + lVar7 * 4);
    iVar2 = *(int *)(lVar5 + (longlong)DAT_0051d0ac * 4);
    iVar3 = *(int *)(lVar6 + (longlong)DAT_0051d0ac * 4);
    iVar8 = iVar3 * param_2 - iVar2 * param_1 >> 0x10;
    iVar10 = iVar4 * iVar9 + iVar1 * iVar8 >> 0x10;
    if (0x31 < iVar10) {
      FUN_001041b0(*(longlong *)(DAT_0051e528 + 0x78) + 0x10,
                   DAT_0051d08c / 2 +
                   DAT_0051dbec +
                   (DAT_0051d084 * (iVar3 * param_1 + iVar2 * param_2 >> 0x10)) / iVar10,
                   DAT_0051dc70 + (DAT_0051d084 * (iVar1 * iVar9 - iVar4 * iVar8 >> 0x10)) / iVar10
                   + DAT_0051d088 / 2,&some_x_pos,&some_y_pos);
      return;
    }
  }
  some_x_pos = 0xffffffff;
  some_y_pos = 0xffffffff;
  return;
}
```


Well fancy seeing you here, some\_x\_pos and some\_y\_pos.

Looks like this might be it, let's confirm our theory about the parameters and also look at the global variables we see here to look for clues on the behavior.

For the first time in this series we're whipping out a debugger! The game doesn't mind debuggers too much, but be wary of the fact that they install some mouse hooks and _will_ cause issues while the game is paused. You may opt to deal with it by doing something about the hooks, or you just be confident about navigating menus with a keyboard in case the debugger pauses.

We can navigate to FUN\_000520c0 by pressing ctrl + g in x64dbg after attaching to osclient.exe and using osclient:base+520c0 as the expression to follow.

![](/Images/Part2/blog2-pic5-debugger.png)

We want to breakpoint the first instruction of the function and log the parameter registers. Ghidras disassembler view can easily tell us which register contains which parameter.

![](/Images/Part2/blog2-pic6-registers.png)

So we want to know the integers from ECX, EDX and R8D. How do we do that? By using a conditional breakpoint with logging. Right click the instruction following an expression took you to in x64dbg and from the â€œBreakpointâ€� submenu click â€œSet Conditional Breakpointâ€�. Then in the Edit Breakpoint window use the following options.

![](/Images/Part2/blog2-pic7-conditional.png)

Those options mean that the debugger will not pause when hitting the breakpoint and prints the log text to the â€œLogâ€� tab with the specified formatting. [Click here for more information on x64dbg log formatting.](https://help.x64dbg.com/en/latest/introduction/Formatting.html)

After saving the breakpoint and typing in chat, we get debugger log output like this.

```
...
param_1 = 7232 param_2 = 6720 param_3 = 97
param_1 = 7232 param_2 = 6720 param_3 = 4294967281
param_1 = 7232 param_2 = 6720 param_3 = 209
param_1 = 7232 param_2 = 6720 param_3 = 97
param_1 = 7232 param_2 = 6720 param_3 = 4294967281
```

Let's check the localplayer we found in the first part if those are the coordinates.

![](/Images/Part2/blog2-pic9-localplayer.png)

Yup. param\_3 is a mystery though, lets go back a bit and see if we can figure that one out.

Looking back at FUN\_0004a170

```	

*(undefined *)(param_1 + 6) = 0;
FUN_000520b0(param_4);
iVar2 = some_x_pos;
iVar1 = *(int *)(param_4 + 0x148);
param_1[1] = some_y_pos + param_3;
*param_1 = iVar2 + param_2;
FUN_000520b0(param_4,iVar1 / 2);
```

There's some wonkyness with the decompilation of the parameters here, for the first call of FUN\_000520b0 we have only 1 parameter when later calls have 2. This is a good reminder that trusting the decompiler doesn't always work. Let's look at the disassembly from the first call.

```
0004a18b 41 8b 91      MOV       EDX,dword ptr [R9 + 0x148]
48 01 00 
00
0004a192 4c 8b f1      MOV       R14,RCX
0004a195 83 c2 0f      ADD       EDX,0xf
0004a198 49 8b c9      MOV       RCX,R9
0004a19b 49 8b d9      MOV       RBX,R9
0004a19e 41 8b f0      MOV       ESI,R8D
0004a1a1 e8 0a 7f      CALL      FUN_000520b0                               undefined FUN_000520b0()
00 00
```


`R9+0x148` seems interesting, as the decompiler shows `param_4 + 0x148` being used as parameter in the second call.

EDX also has 0xf added to it, so the second parameter here may be `*(int *)(param_4 + 0x148) + 0xf` based on the disassembly. And we know param\_4 here is the localplayer object. Let's look at what ReClass again and see whats at +0x148

![](/Images/Part2/blog2-pic10-height.png)

`193 + 15 = 209` which looks around what we'd expect for one of the calls in the debugger log. it seems to vary slightly but guess what, so does the height of the player in the game as the idle animation cycles. so the third parameter of our suspected W2S function is height, but it does also seem like it's relative from ground, as it didn't increase substantially when going up a hill etc. when I tested it. We can now define the parameter names in Ghidra to be something a bit nicer.

```
void FUN_000520c0(int pos_x,int pos_y,int pos_z)
```

Next let's look at these globals in ReClass.

```
iVar10 = pos_x - DAT_0051d0c8;
iVar9 = pos_y - DAT_0051d0cc;
iVar11 = (iVar4 - pos_z) - DAT_0051d0c0;
```

![](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)

Looks like it's camera positioning, although it seems that zooming has no effect. Probably handled separately by the game independent of camera position. So let's rename the globals and local variables as following.

```
iVar4 = FUN_00052250();
camera_delta_x = pos_x - camera_pos_x;
camera_delta_y = pos_y - camera_pos_y;
camera_delta_z = (iVar4 - pos_z) - camera_pos_z;
```

iVar4 is some adjusting coordinate to pos\_z, but FUN\_00052250 isn't the smallest, so I'm focusing on identifying the global variables first.

On to the next part then.

```
lVar5 = FUN_000e8fb0();
lVar7 = (longlong)DAT_0051d0a8;
iVar4 = *(int *)(lVar5 + lVar7 * 4);
lVar6 = FUN_000e8fc0();
iVar1 = *(int *)(lVar6 + lVar7 * 4);
iVar2 = *(int *)(lVar5 + (longlong)DAT_0051d0ac * 4);
iVar3 = *(int *)(lVar6 + (longlong)DAT_0051d0ac * 4);
iVar8 = iVar3 * param_2 - iVar2 * param_1 >> 0x10;
iVar10 = iVar4 * iVar9 + iVar1 * iVar8 >> 0x10;
```

Let's look at whats at DAT\_0051d0a8 and the following DAT\_0051d0ac.

![](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)

Camera rotation it seems, first being pitch and second being yaw, the yaws range seems to be 0-2047. Let's relabel some things.

```
lVar5 = FUN_000e8fb0();
_camera_rot_pitch = (longlong)camera_rot_pitch;
iVar4 = *(int *)(lVar5 + _camera_rot_pitch * 4);
lVar6 = FUN_000e8fc0();
iVar1 = *(int *)(lVar6 + _camera_rot_pitch * 4);
iVar2 = *(int *)(lVar5 + (longlong)camera_rot_yaw * 4);
iVar3 = *(int *)(lVar6 + (longlong)camera_rot_yaw * 4);
iVar8 = iVar3 * param_2 - iVar2 * param_1 >> 0x10;
iVar10 = iVar4 * iVar9 + iVar1 * iVar8 >> 0x10;
```

The rotation is used as an array index apparently. lVar5 is an array at DAT\_016bd540 which is returned by FUN\_000e8fb0, and lVar6 is an array at DAT\_01678d40 returned by FUN\_000e8fc0. Let's retype lVar5 and lVar6 to int\* and rename them to array\_1 and array\_2. I also renamed some others of the local variables.

```
array_1 = (int *)get_array1_ptr();
_camera_rot_pitch = (longlong)camera_rot_pitch;
pitch_arr1 = array_1[_camera_rot_pitch];
array_2 = (int *)get_array2_ptr();
yaw_unk = array_2[camera_rot_yaw] * camera_delta_y - array_1[camera_rot_yaw] * camera_delta_x >> 0x10;
pitch_unk = pitch_arr1 * camera_delta_z + array_2[_camera_rot_pitch] * yaw_unk >> 0x10;
```

It got a lot more compact, but also annoyingly ugly due to unnecessary temp var being used. I'm not sure if I could fix it in ghidra but here it is cleaned up for the blog.

```
array_1 = (int *)get_array1_ptr();
array_2 = (int *)get_array2_ptr();
yaw_unk = array_2[camera_rot_yaw] * camera_delta_y - array_1[camera_rot_yaw] * camera_delta_x >> 0x10;
pitch_unk = array_1[camera_rot_pitch] * camera_delta_z + array_2[camera_rot_pitch] * yaw_unk >> 0x10;
```

How neat is that. Now let's check the last part of this function.

```
if (0x31 < pitch_unk) {
    FUN_001041b0(*(longlong *)(DAT_0051e528 + 0x78) + 0x10,
                 DAT_0051d08c / 2 +
                 DAT_0051dbec +
                 (DAT_0051d084 *
                  (array_2[camera_rot_yaw] * camera_delta_x +
                   array_1[camera_rot_yaw] * camera_delta_y >> 0x10)) / pitch_unk,
                 DAT_0051dc70 +
                 (DAT_0051d084 *
                  (array_2[_camera_rot_pitch] * camera_delta_z - pitch_arr1 * unk >> 0x10)) /
                 pitch_unk + DAT_0051d088 / 2,&some_x_pos,&some_y_pos);
    return;
}
```

Oof that's ugly. Let's reformat it a bit so it's nicer on the eyes.

```
if (0x31 < pitch_unk) {
    int _param2 = DAT_0051d08c / 2 + DAT_0051dbec + (DAT_0051d084 * (array_2[camera_rot_yaw] * camera_delta_x + array_1[camera_rot_yaw] * camera_delta_y >> 0x10)) / pitch_unk;
    int _param_3 = DAT_0051dc70 + (DAT_0051d084 * (array_2[_camera_rot_pitch] * camera_delta_z - pitch_arr1 * unk >> 0x10)) /
        pitch_unk + DAT_0051d088 / 2;

    FUN_001041b0(*(longlong *)(DAT_0051e528 + 0x78) + 0x10,_param2,_param3,&some_x_pos,&some_y_pos);
    return;
}
```

More globals to look at with ReClass. yay

Looking at DAT\_0051d084 and the following few integers it seems like DAT\_0051d084 is some sort of zoom scale, DAT\_0051d088 is the vertical resolution and DAT\_0051d08c is the horizontal resolution. For DAT\_0051dbec and DAT\_0051dc70 I really couldn't figure out how to get other than 0's in reclass, ignore if you dare but I'd imagine they are some kind of screen position offsets set by content elsewhere.

```
if (0x31 < pitch_unk) {
    int _param2 = resolution_x / 2 + x_offset + (zoom_scale * (array_2[camera_rot_yaw] * camera_delta_x + array_1[camera_rot_yaw] * camera_delta_y >> 0x10)) / pitch_unk;
    int _param3 = y_offset + (zoom_scale * (array_2[_camera_rot_pitch] * camera_delta_z - pitch_arr1 * unk >> 0x10)) /
    pitch_unk + resolution_x / 2;

    FUN_001041b0(*(longlong *)(DAT_0051e528 + 0x78) + 0x10,_param2,_param3,&some_x_pos,&some_y_pos);
return;
}
```

I'll let you in on a little secret now that we've made it this far. The last function call is apparently useless for our purposes, \_param2 and \_param3 contain x and y screen position for the world coordinate. Figured this out by implementing what I had figured out so far in to my test project and testing things out. Warning, ugly proof of concept code incoming, I'd not recommend using this as is lol.

```
bool world_to_screen(vec3 pos, vec2* out)
{
    int camera_z_adjust;
    std::vector<int> array_1(2048);
    memoryman.read(modulebase + 0x016bd540, array_1[0], 2048 * sizeof(int));
    std::vector<int> array_2(2048);
    memoryman.read(modulebase + 0x01678d40, array_2[0], 2048 * sizeof(int));
    vec3 camerapos;
    memoryman.read(modulebase + 0x0051d0c0, camerapos.z);
    memoryman.read(modulebase + 0x0051d0c8, camerapos.x);
    memoryman.read(modulebase + 0x0051d0cc, camerapos.y);
    vec2 camerarot;
    memoryman.read(modulebase + 0x0051d0a8, camerarot);

    int scene_data[3];
    memoryman.read(modulebase + 0x0051d084, scene_data[0], 3 * sizeof(int));

    int unk;
    int camera_delta_y;
    int camera_delta_x;
    int camera_delta_z;
    int pitch_unk;
    int pitch_arr1;
    
    if ((((0x7f < pos.x) && (0x7f < pos.y)) && (pos.x < 0x3301)) && (pos.y < 0x3301)) {
        camera_z_adjust = 0;//height_adjustment();
        camera_delta_x = pos.x - camerapos.x;
        camera_delta_y = pos.y - camerapos.y;
        camera_delta_z = (camera_z_adjust - pos.z) - camerapos.z;
        unk = array_2[camerarot.y] * camera_delta_y - array_1[camerarot.y] * camera_delta_x >> 0x10;
        pitch_unk = array_1[camerarot.x] * camera_delta_z + array_2[camerarot.x] * unk >> 0x10;
        if (0x31 < pitch_unk) {
            int _param2 = scene_data[2] / 2 + 0 + (scene_data[0] * (array_2[camerarot.y] * camera_delta_x + array_1[camerarot.y] * camera_delta_y >> 0x10)) / pitch_unk;
            int _param3 = 0 + (scene_data[0] * (array_2[camerarot.x] * camera_delta_z - array_1[camerarot.x] * unk >> 0x10)) / pitch_unk + scene_data[1] / 2;
            out->x = _param2;
            out->y = _param3;
            return true;
        }
    }
    out->x = 0xffffffff;
    out->y = 0xffffffff;
    return false;
}
...
auto loc = osrs_man->playerlist_man->localplayer->get_location(true);
vec2 screenpos;
if (world_to_screen(vec3(loc.x, loc.y, 0), &screenpos))
{
    DrawString("test", 14, screenpos.x, screenpos.y, 1, 1, 1, 1);
}
```

![](/Images/Part2/blog2-pic13-height-off.png)

The height is indeed off. So it's time to finally implement whatever dark magics are in FUN\_00052250 or as I've renamed it height\_adjustment.

For some reason in the world to screen functions disassembly it had determined that it had no parameters for me, but by editing the function signature and setting the parameters to (uint param\_1,uint param\_2,int param\_3) I got it to show the parameters correctly.

```
camera_z_adjust = height_adjustment(pos_x,pos_y,DAT_0051e4f4);
```

Took some testing in game to figure out but DAT\_0051e4f4 is the current floor number and as such the function is as following.

```
int height_adjustment(uint pos_x,uint pos_y,int floor)
{
  longlong lVar1;
  uint uVar2;
  int iVar3;
  longlong lVar4;
  int iVar5;
  longlong lVar6;
  
  iVar5 = (int)pos_x >> 7;
  iVar3 = (int)pos_y >> 7;
  if ((((-1 < iVar5) && (-1 < iVar3)) && (iVar5 < 0x68)) && (iVar3 < 0x68)) {
    if ((floor < 3) &&
       ((*(byte *)((longlong)iVar3 + ((longlong)iVar5 + 0x68) * 0x68 + DAT_01c01970) & 2) != 0)) {
      floor = floor + 1;
    }
    uVar2 = pos_x & 0x7f;
    lVar4 = (longlong)iVar3;
    lVar6 = ((longlong)floor * 0x69 + (longlong)iVar5) * 0x69;
    lVar1 = ((longlong)iVar5 + 1 + (longlong)floor * 0x69) * 0x69;
    return (int)(((int)(*(int *)(DAT_01c01958 + 4 + (lVar4 + lVar1) * 4) * uVar2 +
                       (0x80 - uVar2) * *(int *)(DAT_01c01958 + 4 + (lVar4 + lVar6) * 4)) >> 7) *
                 (pos_y & 0x7f) +
                (0x80 - (pos_y & 0x7f)) *
                ((int)(*(int *)(DAT_01c01958 + (lVar1 + lVar4) * 4) * uVar2 +
                      (0x80 - uVar2) * *(int *)(DAT_01c01958 + (lVar6 + lVar4) * 4)) >> 7)) >> 7;
  }
  return 0;
}
```

There are two globals here.

DAT\_01c01970 seems to be a pointer to an array of bytes, so let's retype it to a `byte *` and DAT\_01c01958 seems to be a pointer to an integer array, so lets retype it to be an `int *`.

Let's try and see if we can implement this then.

```
int height_adjustment(uint32_t pos_x, uint32_t pos_y, int floor)
{
    long long lVar1;
    uint32_t uVar2;
    int iVar3;
    long long lVar4;
    int iVar5;
    long long lVar6;

    iVar5 = (int)pos_x >> 7;
    iVar3 = (int)pos_y >> 7;

    std::vector<byte> array_1(0x68 * 0x68 * 5);
    uintptr_t array_add;
    memoryman.read(modulebase + 0x01c01970, array_add);
    memoryman.read(array_add, array_1[0], 0x68 * 0x68 * 5);

    std::vector<int> array_2(0x68 * 0x68 * 5);
    uintptr_t array2_add;
    memoryman.read(modulebase + 0x01c01958, array2_add);
    memoryman.read(array2_add, array_2[0], 0x68 * 0x68 * 5 * sizeof(int));

    if ((((-1 < iVar5) && (-1 < iVar3)) && (iVar5 < 0x68)) && (iVar3 < 0x68)) {
        if ((floor < 3) && ((array_1[(long long)iVar3 + ((long long)iVar5 + 0x68) * 0x68] & 2) != 0))
        {
            floor = floor + 1;
        }
        uVar2 = pos_x & 0x7f;
        lVar4 = (long long)iVar3;
        lVar6 = ((long long)floor * 0x69 + (long long)iVar5) * 0x69;
        lVar1 = ((long long)iVar5 + 1 + (long long)floor * 0x69) * 0x69;
        return (int)(((int)(array_2[lVar4 + lVar1 + 1] * uVar2 +
            (0x80 - uVar2) * array_2[lVar4 + lVar6 + 1]) >> 7) * (pos_y & 0x7f) +
            (0x80 - (pos_y & 0x7f)) *
            ((int)(array_2[lVar1 + lVar4] * uVar2 +
                (0x80 - uVar2) * array_2[lVar6 + lVar4]) >> 7)) >> 7;
    }
    return 0;
}
bool world_to_screen(vec3 pos, vec2* out)
{
    int camera_z_adjust;
    std::vector<int> array_1(2048);
    memoryman.read(modulebase + 0x016bd540, array_1[0], 2048 * sizeof(int));
    std::vector<int> array_2(2048);
    memoryman.read(modulebase + 0x01678d40, array_2[0], 2048 * sizeof(int));
    vec3 camerapos;
    memoryman.read(modulebase + 0x0051d0c0, camerapos.z);
    memoryman.read(modulebase + 0x0051d0c8, camerapos.x);
    memoryman.read(modulebase + 0x0051d0cc, camerapos.y);
    vec2 camerarot;
    memoryman.read(modulebase + 0x0051d0a8, camerarot);

    int scene_data[3];
    memoryman.read(modulebase + 0x0051d084, scene_data[0], 3 * sizeof(int));

    int unk;
    int camera_delta_y;
    int camera_delta_x;
    int camera_delta_z;
    int pitch_unk;
    int pitch_arr1;
    
    if ((((0x7f < pos.x) && (0x7f < pos.y)) && (pos.x < 0x3301)) && (pos.y < 0x3301)) {
        camera_z_adjust = height_adjustment(pos.x,pos.y,floor_num);
        camera_delta_x = pos.x - camerapos.x;
        camera_delta_y = pos.y - camerapos.y;
        camera_delta_z = (camera_z_adjust - pos.z) - camerapos.z;
        unk = array_2[camerarot.y] * camera_delta_y - array_1[camerarot.y] * camera_delta_x >> 0x10;
        pitch_unk = array_1[camerarot.x] * camera_delta_z + array_2[camerarot.x] * unk >> 0x10;
        if (0x31 < pitch_unk) {
            int _param2 = scene_data[2] / 2 + 0 + (scene_data[0] * (array_2[camerarot.y] * camera_delta_x + array_1[camerarot.y] * camera_delta_y >> 0x10)) / pitch_unk;
            int _param3 = 0 + (scene_data[0] * (array_2[camerarot.x] * camera_delta_z - array_1[camerarot.x] * unk >> 0x10)) / pitch_unk + scene_data[1] / 2;
            out->x = _param2;
            out->y = _param3;
            return true;
        }
    }
    out->x = 0xffffffff;
    out->y = 0xffffffff;
    return false;
}
```

There it is! Highly proof of concept but working world to screen function. The height adjustment function adjusts for regular ground elevation and platform elevation, e.g. bridges or for example the Grand Exchange tiles.

![](/Images/Part2/blog2-pic14-working-w2s.png)

It honestly could have been a bigger pain in the ass than it was. It was relatively easy to find this needle from the haystack even if it is quite non traditional in terms of how world to screen functions usually are.

Bonus: Ground item stacks
-------------------------

Even though the world to screen part was huge. I'll quickly show how I found ground items as it's somewhat related to what we just found. After I found the height\_adjustment function I noticed it was called by for example the render\_player function from part 1. So I went exploring places it was called to maybe find ground items or interactable objects and such. I found FUN\_000aaa60 which when breakpointed was hit every time an item appeared on ground with the parameters being the x and y tiles of the item, so I naturally called it called\_on\_ground\_appearance. In a function calling it I spotted something interesting though.

```
FUN_000769e0(&DAT_017888a0 + (((longlong)floor_number * 0x68 + (longlong)(int)tile_x) * 0x68 + (longlong)(int)tile_y) * 3);
called_on_ground_appearance(tile_x,(ulonglong)tile_y);
```

the constant 0x68 is familiar from the height adjustment function and it's likely the highest tile number for each loaded map chunk. and DAT\_017888a0 seems to have an array of pointer\[3\]'s.

![](/Images/Part2/blog2-pic15-ptrarray.png)

They seem to be all pointing at themselves though. But I happen to have a theory, I've ran in to cyclic linked lists before in my traversing of OSRS code, so maybe these point to items if they are present.

Let's write a quick proof of concept loop to check this theory out.

```
std::vector<uintptr_t> gnd_array(0xFFFF * 3);
memoryman.read(modulebase + 0x017888a0, gnd_array[0], 0xFFFF * 3 * sizeof(uintptr_t));
for (int k = 0; k < 3; k++)
{
    for (int i = 0; i < 0x68; i++)
    {
        for (int j = 0; j < 0x68; j++)
        {
            uintptr_t index = ((k * 0x68 + i) * 0x68 + j) * 3;
            uintptr_t startptr = ((modulebase + 0x017888a0) + index * 8);
            if (startptr == gnd_array[index])
                continue;

            std::cout << std::hex << gnd_array[index] << std::endl;
        }
    }

}
```

![](/Images/Part2/blog2-pic16-evilgenius.png)

Turns out I was right, let's inspect further. Though turns out ground items like to disappear if I take my time so here it is.

![](/Images/Part2/blog2-pic17-itemlinkedlist.png)

So in a nutshell, DAT\_17888a0 is a 0x68 \* 0x68 \* max\_floors array of linked lists that define ground item stacks. Each map tile has its own linked list and if there are multiple items on the same tile you just continue reading the linked list until the next ptr points to the startptr as seen in my loop.

This list does not seem to contain the names or other data for the items, so you need to find another way to find that stuff. I'd look in to parsing the game data files to dump item, object and other ID's rather than doing it dynamically, but I can say that it's possible to somewhat get that all dynamically, but for now this post is getting long enough.

and bad proof of concept code is as following, for item position you can obviously use the i,j,k as the tile position. Just `tile * 128 + 64` to get the coordinate for the middle of the tile.

```
class ground_item
{
public:
	uintptr_t* unk_ptr;
	uintptr_t unk_int;
	uint32_t item_id;
	uint32_t item_count;
};

class grounditem_linked_list
{
public:
	uintptr_t* next;
	uintptr_t* last;
	uintptr_t* smart_ptr;
	uintptr_t* raw_ptr;
};

std::vector<uintptr_t> gnd_array(0xFFFF * 3);
memoryman.read(modulebase + 0x017888a0, gnd_array[0], 0xFFFF * 3 * sizeof(uintptr_t));
for (int k = 0; k < 3; k++)
{
    for (int i = 0; i < 0x68; i++)
    {
        for (int j = 0; j < 0x68; j++)
        {
            uintptr_t index = ((k * 0x68 + i) * 0x68 + j) * 3;
            uintptr_t startptr = ((modulebase + 0x017888a0) + index * 8);
            if (startptr == gnd_array[index])
                continue;

            uintptr_t nextptr = gnd_array[index];

            uintptr_t lastptr = startptr;

            while (nextptr != startptr && nextptr != lastptr)
            {
                grounditem_linked_list ground;
                memoryman.read(nextptr, ground);

                lastptr = nextptr;
                nextptr = (uintptr_t)ground.next;

                ground_item item;

                memoryman.read((uintptr_t)ground.rawptr, item);

            }
        }
    }

}
```

Conclusion
----------

That was a big one. Once again I tried to be thorough on almost all the steps so that you can follow along easily with almost no experience.

I also included some proof of concept code this time that you can concretely test all the stuff we found in this blog entry. Though like said in the post the code isn't great and it isn't supposed to be great. It was mostly just for confirming that it works, so if you want to use that stuff in â€œproductionâ€� you better refactor it to be cleaner and likely faster.

The next one wont be as long, as I may have over promised on whats to come a bit in the last part.

Here's what I'm looking to do posts on in the future and have found:

*   Static interactable objects (Trees, Rocks, Doors, whatever)
    
*   Dynamic objects (Chopped down Trees, mined rocks, opened Doors, Dwarf Cannons etc.)
    
*   Projectiles
    
*   Interface text parsing
    
*   Smaller misc. stuff
    

Feedback, complaints, whatever are easiest sent to [@alert\_insecure](https://twitter.com/alert_insecure) on twitter or alternatively email atte@reversing.games
