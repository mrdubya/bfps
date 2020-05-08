Brainfuck Interpreter, in PostScript!
=====================================

PostScript is not the easiest language to work with, so implementing a
[Brainfuck](http://en.wikipedia.org/wiki/Brainfuck) interpreter in PostScript
seems the right thing to do.
This is the result of a weekend(ish) project to implement Brainfuck in
PostScript, including input, in a clean and maintainable way.

How to Run the Interpreter
==========================
The interpreter has been developed using the
[Ghostscript](http://www.ghostscript.com) command line interface, using stdin
and stdout for reading and writing output.
It does not write the result of the program to a raster so it should not be sent
to a printer.
The implementation uses standard PostScript operators and environment but not
all PostScript interpreters provide the full environment so your mileage may
vary.

The main file `bf.ps` expects to find a file called `prog.bf` in the same
directory as it.
To run the example hello world Brainfuck program do the following:

```console
$ cp examples/helloworld.bf prog.bf
$ gs -q -dBATCH -dNODISPLAY bf.ps
Hello World!
$ 
```

For convenience there is a simple shell script to invoke Ghostscript for you, so
you only need to run `bf`.
So the previous example becomes:

```console
$ cp examples/helloworld.bf prog.bf
$ bf
Hello World!
$ 
```

Interpreter Details
===================

The Implementation
------------------

The interpreter loop is very simple, read command byte from stdin, get
PostScript procedure for command from indexed array, execute the procedure.
An array for all possible command characters is initialised with no-op
procedures and then update it with procedures for the Brainfuck commands.

The most complex behaviour is with the jump commands.
Jumping forward is relatively simple, just count up for any seen `[` commands
and down for any `]` commands.
If a `]` command is seen and the count is zero you have found the matching jump
command.
Reading backwards is not straightforward in PostScript so I went with the simple
approach of keeping a stack of offsets.
At a `[` command and not jumping forward the current offset is pushed onto the
stack.
At the corresponding `]` command and jumping backwards use the offset on top of
the stack else remove it altogether.

The only option for interactive input is to use the `%lineedit` device.
This is line based so I use only the first character and throw the rest of the
line away.
If the program needs more input you will have to input on a new line.

The interpreter is invoked by calling `bf-interp` with a interpreter state
dictionary on the operand stack.
A new interpreter state dictionary is created with `bf-init` which takes a file
object on the operand stack for the source of the Brainfuck program.
A convenience procedure, `bf`, is provided that takes a program file object,
creates a new interpreter state and starts interpretation.

It is possible to redirect where the Brainfuck output goes and where input comes
from.
After creating the initial machine state the output can be sent to a file by
calling `bf-set-stdout` while a file can be used for feeding input with
`bf-set-stdin`.
Each procedure takes the machine state dictionary and a file object (opened
for read or write as required) and returns an update machine state dictionary.
This allows a sequence of calls to modify the machine state as required.
The following example sends output to a file in the same directory, taking input
from another file:

```Postscript
bf-init
(bf-output) (w) file bf-set-stdout
(bf-input) (r) file bf-set-stdin
bf-interp
```

Note that PostScript is itself an interpreted language so performance of the
interpreter will not be great.
I have tried to design the code to be performant yet still readable.
There are some things that could be made quicker but it is most likely not worth
it.
Also a new version of the return jump stack array is created on every push, and
while PostScript is normally garbage collected a sufficiently active or long
running program may encounter a PostScript `VMERROR` and stop.

Implementation Constraints
--------------------------

The dialect of Brainfuck is very close to the original.
The current constraints are as follows:

- The number of cells is 65535 (instead of 30000).
- It is an error moving outside the bounds of the cell array.
- Cells are byte valued (i.e. unsigned 8bit integers).
- It is an error to make a cell hold a value outside the byte value range.
- There is a limit of 65535 nested active return jumps.
- There is no check for unbalanced jump commands (an unbalanced taken `[` jump
  is the same as ending the program).

Any errors reported will be PostScript errors so it may not be obvious what the
actual problem is.

Advanced Usage
==============

If your printer's PostScript implementation supports the executive then you may
be able to run the interpreter on the printer.
There are a couple of approaches to be able to do this.

The first approach would be to Change the last line of `bf.ps` to `currentfile
bf` and then cut and paste the file contents into the executive.
Then you can cut and paste Brainfuck programs into the executive and see the
results there.

After that you would need to write some PostScript that would write output on
the page, coping with line wrapping and page throws.
In this case change the last line of `bf.ps` to `currentfile bf` and append the
Brainfuck program to the file and the just send the file to the printer.

Notes
=====

PostScript is a registered trademark of Adobe Systems Incorporated.
