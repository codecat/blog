+++
title = "Reverse engineering and 'exploiting' a game trainer"
date = 2018-05-20
path = "2018/05/reverse-engineering-and-exploiting-a-game-trainer"
+++

In the weeks after [Heroes of Hammerwatch](https://store.steampowered.com/app/677120/Heroes_of_Hammerwatch/) came out, people kept asking for cheats for the game. You would expect in any other single player game there would be one or two cheats available that players can use, but in Heroes of Hammerwatch we've decided to keep the cheats locked up only for developers. There's a few reasons for that, but the main reason is that the player's stats are persistent; players can bring their own character into multiplayer sessions.

<!-- more -->

![](/2018/05/hwr.jpg)

So, to avoid cheat abuse, there are no official cheats in the game. Further, to avoid people using basic Cheat Engine skills, we've made a few precautions to make it a bit harder to scan and change variables with programs like Cheat Engine.

Naturally, this brings attention of the game to game trainer developers. If you've never heard the term "trainer" before, a game trainer is an external application that allows you to cheat in a game using automated memory editing, often using shortcuts on the keyboard. Since cheating in Heroes of Hammerwatch is not as straight forward as typing "god 1" in the console, slowly but surely trainers started to pop up.

Eventually, people started griefing public lobbies with cheated characters using these trainers, making several players unhappy. This made me think – is there anything we can do to stop griefing? Of course, if you've even slightly been in the loop in terms of security, you already know the answer to that is a definitive NO. Nothing is ever perfectly secure. People will always find ways around your protections as long as the code in question is running on someone else's machine.

But, just for shits and giggles, I decided to take a look at how some of these trainers work and how they're bypassing our protections. There was one trainer in particular (that just happened to be freely available for download) that I decided to look at. So, let's start with our trusty [x64dbg](https://x64dbg.com/) and open up the trainer.

I started by analyzing the entire application so it will find and label all functions, followed by doing a quick search for string references.

![](/2018/05/Strings.png)

Right off the bat we notice a few things;

* This trainer is not in any way protected against debugging, this makes our life a lot easier.
* The message "Could not find code" suggests that it's doing pattern searching for some code, and therefore probably some kind of code patching/hooking.

So, let's take a look at the pattern searching code:

![](/2018/05/FindPattern01.png)

This looks straight forward enough – it's doing a pattern search with the pattern defined on the stack, where 0x3F (the ascii character ‘?') is a wildcard character.

Let's switch x32dbg over to the actual game binary, and see what this pattern is matching:

![](/2018/05/HWR01.png)

Ah, this happens to be one of our protected variable functions. The pattern that is being matched are the lines selected. The importance of this will be made clear soon.

After the trainer has found this code in the game, it will also find another pattern for a much bigger codeblock, which I won't be going into in this blogpost, but it does about the same as described below. Let's take a look what the code does after finding both pointers:

![](/2018/05/Readxor-1.png)

It takes the first pointer it found in the game (013B4694) and adds 4 to it. If you take a look at the previous screenshot, you will see this will put eax on the address of the xor value directly. This is what the variables in the game are xor'd by. It then reads the value from the game process' memory and stores it. (The xor key changes with every game update, which is why the trainer has to grab it.)

After that, it allocates 2 sections in memory, makes it executable, and writes code to it to handle the enabled/disabled/toggled state of the cheats. For the Code01 section, it's a godmode cheat. This is where execution will eventually jump to in order to skip the actual health value set.

If you've been paying attention to the assembly code in the screenshot above, you'll have noticed that it takes the pointers from the pattern search again, subtracts it with a number, and then stores it again. In the case of our previous assembly from the game code, it subtracts 6 bytes, which lands here, 2 instructions before the matched div instruction:

![](/2018/05/SubDiv.png)

Further on in the trainer's code, this "new" address is used to patch a jmp instruction to its own code that was written to memory earlier:

![](/2018/05/PatchedOK.png)

There's a flaw here. Normally, if the game were to get updated and the pattern unable to be found, the trainer would bail out and ask you to "Please let [the developer] know" that it's not working. I thought it would be a fun challenge to break this trainer without having that message show.

Since the new address after subtraction is never validated if it's the right address, we can do some interesting things here. One option is to simply add a single nop instruction in the original game code, like this:

![](/2018/05/Nop1.png)

Now, when the trainer writes its jmp instruction to the -6 bytes offset, it will instead land 1 byte into the first mov instruction, and start writing from there. This means that the resulting assembly is all garbage instructions, which of course will cause undefined behavior in the game (most likely a crash):

![](/2018/05/Nop2.png)

Now, I know, of course, this is not going to stop the trainer developer from updating their trainer, but at least it's an amusing flaw that I found interesting to explore.

Which interesting anti-cheat measures have you come up with or come across?
