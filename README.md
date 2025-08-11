These are just IPS patch files to apply to English ROMs of Red and Yellow.
Many patch utilities exist out there, but you can use LunarIPS as one such.
Or if you do hex editing directly, you would overwrite the bytes "57 5E CB 3A CB 1B" at 0x03EE36 in Red (and Blue?) or at 0x03EFC2 with "16 00 1E 00 00 00".
How does this work? Terribly. It's a hack. I didn't want to accidentally mess with any offsets so I needed a way to replace n bytes with n bytes. It would be a lot better to probably just use a jump routine to skip over all the badge boost logic, but I didn't figure that out. This method works, but it is wasteful.

Reference https://github.com/pret/pokered/blob/master/engine/battle/core.asm and line 6609 .applyBoostToStat for the sequence.

We start by loading in the stat that's going to be boosted.

57 5E are the assembly instructions for loading the value in register a into register d, and load the value pointed to via register hl into register e, respectively. I think. I'm a newb at assembly.
CB 3A and CB 1B are directions for dividing a value across 2 registers -- (e.g. numbers as large as 65k as opposed to 255 in a 1-byte register) by 2. The OG file repeats CB 3A and CB 1B twice more for a total of 3 divisions by 2, effectively generating 1/8 the original value.
(CB 3A means bit shift right what's in register d, so something like 0001 1011 -> 0000 1101 and that last digit, which is a 1 is placed in the Carry flag; CB 1B means rotate right what's in register e which I don't really get why it's named this, but it's different from bit shift right because BSR will always put a 0 on the left-most bit, but rotate right will use what ever is in the carry flag -- the digit we "dropped" from the BSR.
Essentially, these two steps effectively do this: 0001 1011 1010 0001 -> 0000 1101 1101 0000.)

Now, 57 and 5E are shortcuts where a single byte instruction is able to tell the gameboy to move data between the pre defined registers.

But I wanted to just tell it to make the values 0, so that when we do any further optional divisions, it's going to be 0, and the rest of the math sequence is going to take the base stat + 0 badge boost stat to effectively nullify the badge boost.

The byte sequence to load a custom value less than 256 into register d is to use 16 nn and into register e is to use 1E nn where nn is the value you want put in. So I could have used 16 00 1E 00 in place of 57 5E. But that expands the file size. So I don't do that. We follow the logic and realize we can immediately shortcut something here.

If we overwrite 57 5E CB 3A with 16 00 1E 00, we'd be in business with having loaded 00 into registers d and e. But the next steps executed are the rotate right CB 1B. I don't know what the carry flag is going to be from any previous operations. If it was a 0, we'd be fine. But if it's a 1, then we could end up with the binary 0000 0000 1000 0000 = 0x0080 or in decimal a value of 128, and after the 2 divisions by 2 to come yet, could end up accidentally adding +32 to a stat.

So instead I just write a couple no-operation bytes (00) to overwrite the first CB 1B. Now we're sure that the next actual operation is CB 3A which will set the carry flag to 0 as it shifts 0000 0000 -> 0000 0000 (0).

You could for elegance's sake overwrite with a few more 00, but to my naive knowledge, it wouldn't matter. Instruction 00 takes 1 machine cycle; instructions CB 3A and CB 1B take 2 machine cycles each; so using two 00s to overwrite one CB ?? instruction doesn't net you any performance increase.

And that's one point in favor of not jumping past the badge boost routine. If you want to match the normal machine cycle performance of a vanilla game, you'll still want to execute all these steps.

However, this implementation costs one additional machine cycle. The change from 57 to 16 00 goes from 1 cycle to 2 cycles. (The 5E to 1E 00 is 2 cycles to 2 cycles.) So overall performance is degraded with this hack, albeit, over the course of a run, on the magnitude of milliseconds. With the GB processing at 1 mega hertz or 1 million operations per second, we'd need 1000 iterations of this badge boost application to hit 1 millisecond difference between vanilla and this hack. That could be achievable when you consider: Any stat changes may have called the badge boost functions, there are up to 4 badges that boost, so definitely possible across 250 or fewer battles / level ups to hit that. It might, might even be 62.5 battles because if the 1MHz number is the T-cycles and 4 T-Cycles = 1 Machine Cycle, then it will creep up to a measurable difference 4x as fast.
