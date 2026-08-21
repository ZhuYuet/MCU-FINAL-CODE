# 8051 Calculator V9 — Project 12 Developer Scrolling Page

Project 12 is a separate copy of the stable Project 10 firmware. It adds a
fifth instruction page containing a continuous left-scrolling developer line.

## Documentation files

- `PROJECT12_CALCULATOR_FUNCTIONS_AND_BLOCK_DIAGRAMS.md`: complete calculator
  feature summary plus Process, Display and combined Mermaid block diagrams.
- `CALCULATE_V9_CODE_EXPLANATION.md`: detailed explanation of the Process
  controller source.
- `DISPLAY_V9_CODE_EXPLANATION.md`: detailed explanation of the Display
  controller source.

## Operator instructions

In Advance mode, press `D` to open the instruction screen:

```text
<- SCROLLING ->
4=LEFT 6=RIGHT

<- ARITHMETIC ->
A=+ B=- C=* D=/

<- 8bit-LOGIC ->
A=& B=| C=^ D=!

<- ADVANCE    ->
A=^2 B=^ C=^0.5

<- Developer  ->
DANIEL CHEE ... GOH SHAO SEAN ... TAN E-KEN ... THEN MUN PIN ...
```

- `4`: previous page
- `6`: next page
- `*`, `D`, or DEL: return to the empty Advance-mode screen
- Other keys are ignored while instructions are open

On the Developer page, all four names are stored in one line and scroll left
one LCD cell per animation frame. After Then Mun Pin disappears, the line loops
back to Daniel Chee.

In Advance mode, the square-root operator is displayed on the equation line as
the single token `^0.5`. One backspace press removes the complete token.

The Process controller clears the current ALU expression before opening or
closing the instruction pages, preventing display and calculation state from
becoming unsynchronized.

## Arithmetic intermediate chaining

Before the final `=` is pressed, selecting another operator evaluates the
pending Arithmetic operation and replaces line 1 with its real intermediate
value. For example:

```text
1+1+1  ->  2+1
```

The Process controller sends the sign and 16-bit intermediate magnitude to the
Display controller before the newly selected operator. `ANS` is still used when
an operator continues a calculation after a final `=` result.

## Leading Arithmetic operators

A leading `+`, `*`, or `/` is kept on line 1 for input feedback but is ignored
by the ALU. A leading `-` remains the negative sign. For example:

```text
*5+1=  ->  6
```

The LCD therefore keeps `*5+1` visible instead of changing it to `0+1`.

## Cursor correction

LCD hardware blinking (`0FH`) is not used. Before each title redraw, the cursor
is hidden. After line 2 is drawn, the LCD address is restored to the saved
line-1 input position. A software phase alternates between cursor-off (`0CH`)
and a steady underline cursor (`0EH`) at the same rate as the title frames.

Mode 2 backspace redraws are hidden until the complete logical line has been
rebuilt. This prevents stale Arithmetic characters from flashing briefly.

## Firmware and memory

| Controller | Source | Program this HEX |
|---|---|---|
| Process, keypad, and ALU | `CALCULATE V9.asm` | `CALCULATE V9.hex` |
| LCD and presentation | `DISPLAY V9.asm` | `DISPLAY V9.hex` |

| Target | Code size | Flash usage | Remaining | Highest code address |
|---|---:|---:|---:|---:|
| Process | 2,094 bytes | 25.6% | 6,098 bytes | `082DH` |
| Display | 2,668 bytes | 32.6% | 5,524 bytes | `0A6BH` |

Both targets assemble with Keil A51 and ASEM-51 V1.3 with zero errors and zero
warnings. Program both Project 12 HEX files together.
