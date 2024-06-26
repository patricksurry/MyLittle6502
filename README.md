# Fake6502, revamped Plus fake65c02

Mike Chambers' Fake6502 with revamped bug fixes for decimal mode, along with a few other fixes.

Now with an all-new Fake65c02 for the CMOS chip.

The header files in this repository are in the public domain.

The tests files are not! They are GPL'd instruction set exercisers.

fake65c02.h incorporates CMOS support code from the codebase for the [Commander X16 Emulator](https://github.com/commanderx16/x16-emulator/tree/master/src/cpu) which is not a public domain repository, however the changes are relatively minor and fake6502.c is still marked as fully public domain.

Since their Fake6502 code is still marked as public domain, I figured that it would be ok to incorporate those changes here.

I have a pending github issue on the commander x16 repository asking if they intended this or if I have to relicense under the BSD 2 clause.

See https://github.com/SamCoVT/TaliForth2/tree/master-64tass/c65 for an example that implements the fake65c02.h interface.

# CHANGELOG

- (PDS) Correct 65c02 NOP behavior, passing Klaus Dormann's tests
- Wrote decimal mode according to http://www.6502.org/tutorials/decimal_mode.html#A
  to be exactly correct.
- Fixed interrupt masking.
- Fixed decimal mode adc and sbc
- Fixed exec6502 possibly executing many billions more instructions than desired.
- Fixed documentation
- Fixed overflow calculation (I believe) for decimal mode. The V flag is undocumented, and its value is pretty much useless,
  but I believe it is correct. I have yet to run the test code from http://www.6502.org/tutorials/decimal_mode.html#A
  but it is a TODO.

The emulator uses global state and there is no "instancing" it.

To use the emulator, the expected usage is that you include it in _ONE_ c file.

these are the functions you must (and typically would only) implement:

```c
uint8 read6502(ushort addr) {
	/*Return something that would make sense given address "addr"*/
}

void write6502(ushort addr, uint8 val) {
    /*Do something here using the 8 bit value "val"*/
}
```

you can additionally define a "hook" to be executed after every instruction.

I used [this](https://github.com/omarandlorraine/fake6502) instruction exerciser along with
[this](https://github.com/mist64/kernalemu) C64 kernal emulator to verify that my fixes were correct and
did not break anything.

I used these references for opcodes and what they do:

[6502.org's opcodes list](http://6502.org/tutorials/6502opcodes.html)

[obelisk.me.uk's 6502 reference](http://www.obelisk.me.uk/6502/reference.html)

[This one I found on google sites](https://sites.google.com/site/6502asembly/6502-instruction-set)

Commodore basic v2 boots and BASIC works just as expected. For loops, prints, math, all work as expected.

All other 6502 based systems I tested also work, although kernalemu only implements a few of them completely.

For instance, many commands in the C128 kernalemu implementation are completely dead due to being stubbed out with NYI()

(Not Yet Implemented) calls.

I have yet to verify that this emulator works with MOARNES, mike chamber's NES emulator.
However, given that the NES's 6502 variant does not use decimal mode, I would guess
that it would still work just fine.

If you find any errors feel free to post an issue or make a PR.

Moarnes can be found on github [here](https://github.com/darlanalves/moarnes)

or on sourceforge, [here](https://sourceforge.net/projects/moarnes/)

The latter is supposedly "more official"

Further documentation on how to use the emulator is in the header file.

```
FAQ
```

Q: How do I use this in multiple C files?

A: Define FAKE6502_NOT_STATIC. Then, create externs for all the registers and functions you want to access.

These are the registers as they are declared in the header file, with FAKE6502_NOT_STATIC. the only difference
when you don't define FAKE6502_NOT_STATIC is that these are all declared "static" so that hopefully the compiler
will optimize the code a bit better.

```
/*6502 CPU registers*/
ushort pc;
uint8 sp, a, x, y, status;
/*helper variables*/
uint32 instructions = 0;
uint32 clockticks6502 = 0;
signed long clockgoal6502 = 0; /*Made a signed number.*/
ushort oldpc, ea, reladdr, value, result;
uint8 opcode, oldstatus;
```

And for fake65c02, there is an additional variable "waiting" which indicates if the WAI instruction has been executed,
meaning that the processor will not execute anything until an interrupt occurs.

```
/*6502 CPU registers*/
ushort pc;
uint8 sp, a, x, y, status;
/*helper variables*/
uint32 instructions = 0;
uint32 clockticks6502 = 0;
uint32 clockgoal6502 = 0;
ushort oldpc, ea, reladdr, value, result;
uint8 opcode, oldstatus, waiting6502 = 0;
```

These are the functions as they are declared in fake6502.h:

```c
 void reset6502()
   /* Call this once before you begin execution*/

 uint32 exec6502(uint32 tickcount)
   /* Execute 6502 code up to, and possibly one instruction over, the next specified
     count of clock ticks. Returns the number of clock ticks actually executed. */

 uint32 step6502()
   /*Execute a single instrution. */

 void irq6502()
   /* Trigger a hardware IRQ in the 6502 core.    */

 void nmi6502()
   /* Trigger an NMI in the 6502 core. */

 void hookexternal(void *funcptr)
   /* Pass a pointer to a void function taking no
     parameters. This will cause Fake6502 to call
     that function once after each emulated
     instruction. */
```

Q: why did you define these weird types like "ushort" and "uint8"!!! Why not just use stdint

A: C89 compliance.

Q: Why are the registers global variables? Don't you know that's super slow and bad? Make a struct!

A: Well since you will typically be stepping this CPU alongside emulating other devices in a computer,
it's probably not possible for the opimizer to turn the emulated cpu's registers into real registers,
since you'll be jumping between functions often.

Also, function pointers are used for the dispatch. Obviously that rules out the possibility of using hardware
registers entirely.

Yes, I know, this makes it more difficult to have multiple virtual 6502's running at the same time. You'll figure out
how to work around that.

Blame Mike Chambers. He made that design decision. If you really want it as a class or struct, then make a pull request.

Q: I'm writing an NES emulator...

A: Another one?

Q: Can this be used to emulate the commander X16?

A: Yes. Some changes were adapted from the Commander X16's emulator, although that emulator has bugs in some of its decimal mode ops.

Those bugs have been fixed in this repository.

Q: I have found a bug in your implementation!

A: Make an issue about it!

Q: Did you implement decimal mode correctly?

A: I believe so. The overflow flag works based on the results of a binary (not BCD) addition or subtraction,

which is what I have implemented. It's what 6502.org says.

The carry flag is _not_ set back to one (the non-carry state, it is an inverse borrow) by SBC if a carry does not occur.

This is, I believe, the intended behavior. However, it means that having the carry flag cleared will cause more than just

the subsequent SBC to be treated as if a carry occured. Specifically, if the accumulator starts out at 0 and the

carry flag starts out at 5 (inverse borrow) then...

```asm
SED   ; enable decimal mode
SBC 4 ; the carry flag is set so an additional 1 is subtracted.
SBC 1 ; This causes A to be 99. The carry flag is now

```
