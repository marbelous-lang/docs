Marbelous Spec 
===
Overview
---

Marbelous is a language centered around the movement of marbles.

A Marbelous program consists of at least one rectangular **board**. Such boards are divided into two-character **cells**. 

The entire flow of the program is driven by the movement of **marbles**, or 8-bit values. All marbles naturally fall towards the bottom of the board at the same speed. The program runs in time steps called **ticks**.

If a marble falls off the bottom of a board, the corresponding character (ASCII) is outputted to STDOUT. If multiple marbles fall off the bottom of a single board in one tick, they are outputted from left to right. If a marble is moved off the side of the board, it may either be cycled to the other side of the board (cylindrical boards) or discarded; this shall be determined by the implementation of Marbelous.

Some cells may contain either **devices** or calls to other boards; these may shift marbles left or right unconditionally or under certain conditions, or may change the value of marbles that pass through them. Devices and calls to other boards do not move. (See Appendix 1 for a list of devices).

All addition is done modulo `256`. Negative numbers resulting from subtraction are incremented by `256` until between `0` and `255` inclusive.

Section 1 - Boards
---
Each Marbelous program has at least one board, which is executed when the program starts. This board is the main board, and has the name `MB`. 

A board is run in timesteps called **ticks**. During one tick, every marble not held in place (see Appendix 1, synchronisers) falls downward by one cell. If multiple marbles are forced into the same cell at the end of a tick, they are merged together (they become a single marble, with all individual values added together). 

    01 .. # hex literal (01)
    .. 02 # hex literal (02)
    .. // # shifts to the left
    
    # Results in:
    Tick  1      2      3
        01 ..  .. ..  .. ..
        .. 02  01 ..  .. ..
        .. //  .. 02  03 // # 01 and 02 have merged in tick 3
    
Every board, including the main board, has inputs and outputs. Inputs are denoted by `}n`, and outputs are denoted by `{n`, `{<`, or `{>`, where `n` is a base-36 digit. 

At the beginning of execution of each board, all input devices (`}n`) are replaced by a marble with the value of the `n`th input to the board. If there is more than one device with the same `n`, each are replaced with a marble of the respective value.

    # Passing 05, 03, and 02 as inputs 0, 1, and 2 respectively, to:
    }0 }2 }1
    }2 .. }1
    .. .. ..

    # Results in the following at the start of tick 1:
    05 02 03
    02 .. 03
    .. .. ..

Integer arguments passed to the program are used to fill input devices in the first execution of the main board. The first argument goes into `}0`, the second in `}1`, etc.

Once at least one of every single type of output device (`{0`, `{D`, `{>`, etc.) contains a marble, the board is **terminated**. Once a board is terminated, all marbles in each type of output device are summed together, and considered as the board's output. 

    # Passing 1 as input 0.
    }0 .. 32
    {0 .. {0
    
    # Results in
    Tick  0         1
      01 .. 32  .. .. ..
      {0 .. {0  01 .. 32 # All outputs are filled, board terminates.
      
    Output 0: (01 + 32) = 33
    
A board is also terminated if no movement occurred during one tick.

    24 .. 
    .. ..
    
    # Results in
    Tick  0      1     2     3
        24 ..  .. .. .. .. .. ..
        .. ..  24 .. .. .. .. .. # Nothing has moved, board terminates.
    
    STDOUT: $ (value=0x24=36)

Termination of the first execution of the main board ends the Marbelous program. The return code will be the output in `{0` if present on the board, or `0` otherwise.

All boards save the first board must be named. If the first board is not named, it is assumed to be `MB`. A board is named by a line starting with `:`, and followed by the name of the board. This line should appear right before the first line of cells of the board. Board names may consist of non-whitespace printable ASCII characters. 

The length of a board's name must be less than or equal to `2 * max(1, N+1, M+1)` characters long, where `N` is the greatest input number used (`}5` would be 5, and `}A` would be 10), and `M` is the greatest output number used (`{<` and `{>` are not counted). 

The **actual name** of a board shall be the board's given name, repeated and truncated as needed to be exactly `2 * max(1, N+1, M+1)` characters long.

If there are multiple boards with the same name (here referring to the actual name) within a single file, the last board in a file with the given name shall be used.

Section 2 - Cells and Devices
---
Cells are the fundamental parts of a board. Cells are typically written with two non-whitespace characters.

    Examples of cells:
    ++  Ab  BA  ^0  ~~  @D  @@  >4  ..
Cells may be separated by spaces. This is optional, but recommended.
    
    4A .. .. 3D
    # both above/below are valid and equivalent
    4A....3D

These two characters must represent one of the following:

1. Hexadecimal Literal: Written as two hexadecimal digits, with both characters always capitalized. Represents a marble at the location of the cell on tick 1, with a value equal to the hexadecimal literal. Examples of hexadecimal literals:

        BA  30  29  54  7b # note that this last cell, 7b, is NOT a literal
    
2. Empty Cell: Usually written as two periods: `..`, this represents an empty cell. 

        .. .. ..
    If the spaces between cells are NOT used, two spaces may also be used to represent an empty cell.
    
        ......  .. # 5 empty cells
    Mixing of these two styles is allowed, but discouraged.

3. Device: Devices modify either the value or direction of marbles that pass through them. For a complete list of devices, see Appendix 1.

        /\ \/ -- +D ?7 # each of these are valid devices
        
4. Call to Another Board: See Section 3 for details on calling boards. In the examples of cells at the start of this section, the following are calls to other boards:

        Ab @@ # either a call to a board named Ab@@
              #  or calls to boards named Ab and @@
    If there are ambiguities in determining what boards to use (for example, `ab cd ef` when there are boards named `ab`, `abcd`, `cdef`, and `ef`), the longest matching name shall be used (`abcd` in this case). Then, out of the remaining characters, the longest matching name shall be used (`ef`), and so on. Thus `ab cd ef` represents a call to `abcd` and a call to `ef`, not a call to `ab` and a call to `cdef`.
    
    Any board may be called, including `MB`. Recursion is permitted.

If there are any ambiguities as to what a cell may represent, the earliest item on the above list that matches the cell should be used.

Section 3 - Calling Boards
---
A board may be called from another board. Such a call requires exactly `max(1, M+1, N+1)` adjacent cells, where `M` is the largest input number used in the board being called (`}D` would require `M>=14`, regardless of whether `0-C` were used), and `N` is the largest output number used (left/right outputs are not used for this count).

These cells are to be filled with the name of the board to be called, truncated or repeated as many times as needed to fill the space.

For example:
    
    :a # 3 input board, }0, }1 unused
    }2
    :bBc # 2 input, 1 output board.
    }0}1}0
    {0{0{0
    :bC # no input/output
    ..
    :bD # only left output
    32
    {<
    :MB
    aa aa aa # call to a
    bB cb .. # call to bBc
    bC .. .. # call to bC
    bD .. .. # call to bD

In the case of conflicts due to truncation or repetition, the latest defined board in the same file (or in the latest included file if not in defined in the current file) is used.

These board calls are not made until the inputs are filled up, and act similar to synchronisers (see Appendix 1, synchroniser) until the calls are made. The first cell occupied by the call represents the 0th input, and will be replaced by the 0th output once run. The second cell represents the 1st input/output, etc.

    24 .. 24
    .. .. ..
    .. 32 .. 
    Bo ar ..
    .. .. ..
    
    :Boar
    }1 }0
    {0 {0 # add two inputs together
    
    # results in
    Board/Tick   MB/1      MB/2      MB/3      MB/4   Boar/1    MB/4      MB/5
               29 .. 24  .. .. ..  .. .. ..  .. .. ..  32 29  .. .. ..  .. .. ..
               .. .. ..  29 .. 24  .. .. ..  .. .. ..  {0 {0  .. .. ..  .. .. ..
               .. 32 ..  .. .. ..  29 .. 24  .. .. .. Boar/2  .. .. ..  .. .. ..
               Bo ar ..  Bo 32 ..  Bo 32 ..  29 32 24  .. ..  5B ar 24  Bo ar ..
               .. .. ..  .. .. ..  .. .. ..  .. .. ..  32 29  .. .. ..  5B .. 24
    
    STDOUT: [$ (0x5B=[, then 0x24=36=$)
               
Execution of a board starts and completes in one tick of the caller board. Order of execution of multiple board calls started in the same tick is not specified (this is only an issue if both boards attempt to read from STDIN or output to STDOUT). 

Left/Right outputs act similarly, except in the tick after the board executes, the marble is placed one cell to the left/right of the call block, instead of one cell under.

Section 4 - Includes and Comments
---
A number sign (`#`) marks the start of either a comment or an include statement.

If `#` is the first character of a line, excluding whitespace, and is immediately followed by the word `include` (case-sensitive), then it is an include statement. Otherwise, it is a comment, ending at the newline.

An include statement takes the following form:

    #include file_name.mbl

Note that quotes are not used. Include statements will load all boards defined in the specified file. 

In the event of board name conflicts between a file and the included file, the board defined in the original file is used in the following cases:

1. In a file including the original file, that does not itself define a board with such a name.
2. In the original file itself.

The board defined in the included file is used in the following cases:

1. Inside the included file itself.

Boards defined inside a file included by an included file are *not* available for usage in the original file. That is to say, if file A includes file B, and file B includes file C, and file C defines a board named `Test`, then file A may not use file C's `Test` unless file A itself includes file C.

Each file should have its own `MB` board (see Section 1, boards). This `MB` board will be unavailable to any file including a file, and will only be run if the file itself is run. This allows for unit testing to be done in included files.


Appendix 1 - Comprehensive List of Devices
---
In this list, unless otherwise stated, `n` must be replaced by a base-36 digit (`0-Z`).

|  |Device Name|Description|
|--|-----------|----------|
||**Flow Control**||
|`//`|Left Deflector|Shifts the passing marble to the cell left of the deflector. Movement leftward occurs during the tick after a marble reaches the left deflector, and replaces the standard downward falling action.|
|`\\`|Right Deflector|Shifts the passing marble to the cell right of the deflector. Movement rightward occurs during the tick after a marble reaches the left deflector, and replaces the standard downward falling action.|
|`@n`|Portal|A marble entering a portal will exit through another portal in the same board with the same value for `n`. If more than one such portal exists, the portal of exit will be selected randomly. In the special case where no other portal with the same `n` can be located, the marble simply continues downward. Movement to the location of another portal occurs in the same tick in which the marble reaches the portal.|
|`&n`|Synchroniser|Holds passed marbles in place, and only releases them when all synchronisers in a board with the same `n` have been filled. If marbles are passed through the same synchroniser while it holds a marble in place, the marbles are merged with the marble in the synchroniser.|
||**Conditionals**||
|`=n`|Equals To|If the passed marble is equal to `n`, then this device acts as an empty cell. Otherwise, this device acts as a right deflector.|
|`>n`|Greater Than|If the passed marble is strictly greater than `n`, then this device acts as an empty cell. Otherwise, this device acts as a right deflector.|
|`<n`|Less Than|If the passed marble is strictly less than `n`, then this device acts as an empty cell. Otherwise, this device acts as a right deflector. Note that for `n=0`, marbles are unconditionally shifted to the right.|
||**Arithmetic**||
|`+n`|Adder|Adds `n` to passing marbles.|
|`-n`|Subtractor|Subtracts `n` from passing marbles.|
|`++`|Incrementor|Adds `1` to passing marbles. Equivalent to `+1`.|
|`--`|Decrementor|Subtracts `1` from passing marbles. Equivalent to `-1`.|
||**Bit Operations**||
|`^n`|Bit Checker|A bit checker returns the value of the `n`th bit of the passed marble as either 0 or 1, counting upwards from the least significant bit (considered as the 0th bit). `n` must be between `0` and `7` inclusive.|
|`<<`|Left Bit Shifter|Performs a logical left bit shift (shifts by `1`) on the values of passing marbles.|
|`>>`|Right Bit Shifter|Performs a logical right bit shift (shifts by `1`) on the values of passing marbles.|
|`~~`|Binary Not|Inverts the bits of the values of passing marbles.|
||**Input/Output**||
|`]]`|STDIN|Tries to read a single byte of input from STDIN. If unsuccessful, the passed marble is diverted to the right. Otherwise, the marble passes downward with its value changed to the byte read.|
|`}n`|Input|This is replaced during every call of the board by a marble with the value of the `n`th input passed to the board. Acts like an empty cell. More than one input device with the same `n` may be used per board. See Section 1 for more details.|
|`{n`|Output|Acts similarly to a synchroniser with all other output devices on the board, except the board terminates after all outputs are filled. More than one output device with the same `n` may be used per board. See Section 1 for more details.|
|`{<`|Left Output|Similar to output devices, except the marble will be outputted from the board on the left instead of downwards. See Section 1 for more details.|
|`{>`|Right Output|Similar to output devices, except the marble will be outputted from the board on the right instead of downwards. See Section 1 for more details.
||**Miscellaneous**||
|`\/`|Trash Bin|Any marble that passes through a trash bin is removed from the board.|
|`/\`|Cloner|A cloner emits copies of the passed marble to both the cell left of, and the cell right of itself. The original marble is then discarded.|
|`!!`|Terminator|The board will terminate at the end of the tick where a marble reaches a terminator. Any filled outputs at this point will still be used.|
|`?n`|Random|Ignores the value of the passed marble and sets its value to be a random integer between `0` and `n`, inclusive.|
|`??`|Random|Sets the value of passing marbles to be a random integer between 0 and the value of the passed marble, inclusive.|

Appendix 2 - Examples
---
to do

