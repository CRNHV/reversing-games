# Old School RuneScape Steam version - Players and NPCs | Reversing Games
  
With the release of the [Steam version of Old School RuneScape](https://store.steampowered.com/app/1343370/Old_School_RuneScape/), the game client was ported from its old Java version to a native C++ version so I figured what's a better time to start looking in to reverse engineering the game.

The use of third party clients such as [RuneLite](https://runelite.net/) is very popular in Old School RuneScape, some even say it's necessary to use them to enjoy the game though I disagree with this personally. Third party clients are designed to bring a lot of Quality of Life improvements to the game by making UI additions that often present the game data better for example displaying the names of items on the ground to highlight good drops or overlaying the names of friends and clan chat members. These are simple examples but should give you an idea of what the clients look to do. Here's where I try to come in, since obviously none of the Java based third party clients would work for the new C++ version on Steam I figured maybe there would be some demand for some kind of an alternative for the Steam version.

Due to the nature of my goals I will not be reverse engineering or posting any analysis of the bot detection systems or mouse heuristics recording if they are there. I would also like to point out that I have no previous experience with the game and its client code, so some of the things I do may be done weird or might seem obvious to a more veteran RuneScape hackers.

In this first part of my blog series on reverse engineering this game, I will be going through finding the local player, the player list and the NPC list. I will be going through the process thoroughly step-by-step as I want to keep this post very accessible to even those who have no reverse engineering or game hacking background and I will mostly show the decompiler output rather than trying to explain the assembly.

Tools used: [ReClass.NET](https://github.com/ReClassNET/ReClass.NET) and [Ghidra](https://ghidra-sre.org/)

Contents of this blog are based on the 26.2.2021 build of osclient.exe

Where to start?
---------------

When starting to reverse engineer a new game and wanting to find player objects, the most common place to start is often memory scanning the game for player related data and using memory breakpoints to find out where the data is written or accessed using tools like ReClass or Cheat Engine. But I had a shortcut as I knew that in OSRS there is a chat command called ::renderself which you can use to toggle the rendering of your own character, which I figured would be a good place to start in this case.

I started off opening the game binary in Ghidra, running analysis with default settings and **rebasing the program to 0**, so that I can use addresses I see as relative ones to the game's exe modules base address.

Finding the local player
------------------------

I started off by searching for the string renderself and unsurprisingly there was a match.

![](/Images/Part1/1-renderself.png)

I went to the code that references the string and it seems to be very simple code.

UPDATE: Jagex moved from using chat command parsing to a console system, so the toggle function is now registered to a renderself command. The registered function is referenced and found a bit above the function that has “renderself” as parameter in the following way:

```
    local_40 = (code *)&RENDERSELF_TOGGLE;
    local_10 = &local_48;
    local_48 = &PTR_LAB_003df870;
    FUN_0008c820((longlong *)param_1,(undefined8 *)"renderself",(undefined8 *)&DAT_003df760, (undefined8 *)"Toggles drawing of the player character",(longlong)&local_48);
```

```
  char cVar2 = FUN_000ac2d0(local_e8,"renderself");
  if (cVar2 != '\0') {
    DAT_0043dd0a = DAT_0043dd0a == '\0';
  }
```
So there's a string comparison going on, comparing what is likely to be your chat input to renderself, and if the comparison matches we flip a boolean value, which we can name should\_renderself. We can use Ghidra to rename these variables and functions so that we get something like this.

```
  char match = strcmp(input,"renderself");
  if (match != '\0') {
    should_renderself = should_renderself == '\0'; // Would usually be seen as should_renderself = !should_renderself;
  }
```

Now we can double click should\_renderself in Ghidra to bring us to a view like this where we can see the places the variable is used.

![](/Images/Part1/2-should_renderself.png)

If we double click the XREF\[4\] we can get a nice list of all the XRefs to the variable, go through them to see how the variable is used.

The final XRef brings us to a function that decompiles nicely to the following.

```
void FUN_000ab1d0(void)

{
  if (should_renderself != '\0') {
    FUN_0009d280(&DAT_01705670,0);
    return;
  }
  return;
}
```

So obviously if should\_renderself is true we call FUN\_0009d280 with &DAT\_01705670 as a parameter. Let's see what DAT\_01705670 contains with ReClass by inputting `<osclient.exe>+1705670` as the address, remember this later as its how you get to these relative addresses in ReClass. If you're having issues with this, make sure you rebased the program to 0 in Ghidra.

![](/Images/Part1/3-reclass-localplayer.png)

We have two pointers at the address, we could go deeper in to this but in short, I believe it's some sort of a smart pointer where the second pointer points to the actual object we want to take a look at while the first one contains some data we really have no interest in.

Let's change the pointer type from void to point at a class instance by clicking the icon with the blue arrows going in a circle and selecting â€œClass Instance. Now expand the class and let's look at what the class contains. There's not much going on, but let's try doing something in the game to see if anything happens, and it does. It appears that object+0x10 contains a 2D integer vector of the local players X and Y coordinates and object + 0x18 contains an integer that seems to indicate the rotation of the local player.

![](/Images/Part1/4-reclass-coords.png)

That means success! We found the local player object.

We can now return to the decompilation of the function from before and make some neat assumptions about what the function is. Because FUN\_0009d280 is explicitly passed the local player I will assume it's a generic render\_player function rather than a function intended solely for the local player. So let's name it render\_player and let's name FUN\_000ab1d0 as render\_self.

```
void render_self(void)

{
  if (should_renderself != '\0') {
    render_player(&localplayer_ptr,0);
    return;
  }
  return;
}

```

Finding the player list
-----------------------

Now how do we do this? the local player was easy enough due to it being right next to the should\_renderself check. Well the key is in the function from earlier. We need to go to the render\_player function and go through its XRefs to see where and how it's called elsewhere.

Once in the render\_player function, you can see a â€œFunction Call Trees window at the bottom of Ghidra. We can use that to see where render\_player is called.

![](/Images/Part1/5-function-call-tree.png)

If we look at the first incoming reference FUN\_0009d200 we come to a function that seems to be iterating something. This is what we like to see!

Let's take a look at what DAT\_502b18 contains in ReClass. It's an integer that seems to be the number of something. If we move around in a populated area we can notice that it does seem to be the number of players nearby. This is very promising, let's look at DAT\_0050ab20 then, we can see it's an array from the decompilation and based on what ReClass shows us it's an array of 32 bit integers.

![](/Images/Part1/6-player-indices.png)

To be more pleasing to view we can change the type of the first line of the class to be an Array with the Change Type option, then we can set it to be an array of Int32's by once again clicking the blue arrows and selecting Int32. You can change the number in the arrays corner brackets to change the amount of Int32's the array is supposed to contain, though we don't know as of now the maximum size of this array so I just put an arbitrary number there to check out the values.

![](/Images/Part1/7-reclass-index-array.png)

So obviously no player pointers there, let's keep going. Seems like the number we get from the array is checked against DAT\_0043dda8 and DAT\_0043ddc8 so let's check those out. The first one seems to contain the value -1 and the second one seems to contain an integer that is also present in the array from earlier. If I were to make an educated guess it would be checking if it's the index for the local player so we don't get rendered twice, but we can confirm that later.

Now we get to the juicy part. The first parameter of render\_player which we know is supposed to be the pointer to a player object seems to be `&DAT_016fd670 + (longlong)iVar1 * 2` so let's check that out then.

![](/Images/Part1/8-playerlist.png)

It seems that we have a few pointers that coincidentally happen to be located on array\_base + 4 \* 0x10, 4 also happens to be the first index from the earlier array. So let's define the first line of the class as an Array again, but this time an array of a class instance which contains 2 pointers. In the earlier index list we could see indices up to the thousands so for testing purposes I'll just set the array size to something like 8000 to get something like the following.

![](/Images/Part1/9-playerarray.png)

If we change the pointer type to that of the localplayer we had earlier we can confirm that this is indeed a player object. I was also able to confirm that the integer at DAT\_0043ddc8 was the local player index.

![](/Images/Part1/10-playerobject.png)

We now know what everything is, we can get better names in the Ghidra decompilation.

How neat is that!

Bonus: Finding the NPC list with no particular method
-----------------------------------------------------

Sadly it seems that NPCs are not rendered with the same render\_player function. So we needed another method to find it, or I could just happen to go to where render\_self and render\_players were called through the function call tree and look at the adjacent functions to see if there is anything interesting.

![](/Images/Part1/11-renders.png)

And yup. FUN\_000a9ed0 began with something very familiar:

I won't go in to too much detail as this is very similar to what we went through in the earlier section. I'll just check what is contained at those locations in this case.

DAT\_0051d5f8 contained the NPC count like it did with the player count, DAT\_01708898 contained a pointer to the array of NPC indices, and finally DAT\_017088a8 contains a similar array to before with smart pointers. Though in this case the address is offset by 8, so to read it the same as you would the playerlist you need to read from 017088a0.

So we simply get


Conclusion
----------

This was just the first part of my planned series on Old School RuneScape. The next part will cover a bit about what the player and NPC objects contain, world to screen coordinate translation and ground item objects. If you want to get ahead of the blogpost I can give a few hints: overhead text rendering for world to screen and things found in the world to screen function can help with ground items a bit :)

Here's a bit of a preview of the stuff you should be able to do after the next posts contents.

![](/Images/Part1/12-preview.png)

I hope I didn't make it too unbearably step-by-step, but I want to help people understand how reversing a game can be done in practice without using too much technical language or pre-existing knowledge.

I'd really like to know if you think I shouldn't be this detailed in future posts or if you want me to keep it going the same way.

Feedback, complaints, whatever are easiest sent to [@alert\_insecure](https://twitter.com/alert_insecure) on twitter or alternatively email siteatte@reversing.games

Reposting the content of this blogpost is allowed as long as you link back to this blogpost as the original source and remember to credit me. Disclaimer: Linking this on something like UC is not possible so do not post this there.
