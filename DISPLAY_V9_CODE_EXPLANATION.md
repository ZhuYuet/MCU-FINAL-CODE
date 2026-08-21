# `DISPLAY V9.asm` Code Explanation

## 1. Role of this controller

`DISPLAY V9.asm` runs on the Display AT89S52. It does not scan the keypad and it
does not perform calculator operations. Its responsibilities are:

- receiving digits, semantic events and result payloads over UART;
- acknowledging every received byte with `06H`;
- tracking the current visual mode;
- formatting equation characters and multi-character operator tokens;
- maintaining the complete Logic expression and its 16-cell window;
- formatting signed decimal, binary and quotient/remainder results;
- displaying error messages;
- controlling the 16x2 LCD through Ports 1 and 2;
- animating mode titles, software cursor blinking and the Developer page.

The Display controller trusts the Process controller's calculation result. It
only decides how that result should appear.

## 2. Program organization

The source contains these main regions:

1. Protocol constants, RAM allocations and visual-state bits
2. UART receive loop and event decoder
3. Operator and backspace formatting
4. LCD power, menu and mode screens
5. Mode-title and cursor animation
6. Instruction and Developer pages
7. Logic expression buffer and manual window
8. Result and error formatting
9. LCD initialization and low-level writes
10. Timing delays and text tables

Execution begins at `0000H` and jumps to `MAIN`.

## 3. Hardware interfaces

| Interface | Code use |
|---|---|
| Port 1 | LCD eight-bit data bus `D0-D7` |
| `P2.0` | LCD register select `RS` |
| `P2.1` | LCD read/write `RW`; held low for writes |
| `P2.2` | LCD enable pulse `EN` |
| `P3.0/RXD` | Receives Process-controller bytes |
| `P3.1/TXD` | Sends `06H` acknowledgement |

`LCD_CMD` clears `RS`; `LCD_DATA` sets `RS`. Both enter `LCD_WRITE`, which
places the byte on Port 1, forces `RW = 0`, pulses `EN` and waits for the LCD.

## 4. RAM map

| Address/name | Purpose |
|---|---|
| `30H RX_BYTE` | Most recently received byte or temporary character |
| `31H DISPLAY_MODE` | `0` menu, `1` Arithmetic, `2` Logic, `3` Advance |
| `32H:33H RESULT_H:RESULT_L` | Received 16-bit result |
| `34H:35H TMP_H:TMP_L` | Decimal conversion working value |
| `36H:37H QUO_H:QUO_L` | Divide-by-10 quotient |
| `38H REM_L` | Divide-by-10 remainder/digit |
| `39H DIGIT_COUNT` | Number of decimal digits |
| `3AH BIT_TEMP` | General character, bit or token-length temporary |
| `3BH:3CH RESULT_REM_H:L` | Received calculator remainder |
| `3FH TOTAL_WIDTH` | Width of a right-aligned remainder result |
| `40H-51H EXPR_BUF` | 18-byte complete Logic expression |
| `52H EXPR_LEN` | Current Logic expression length |
| `53H EXPR_VIEW` | First Logic-buffer character shown on line 1 |
| `54H LAST_CHAR_LEN` | Width of last token: 1, 2 or 4 characters |
| `55H MODE_SCROLL_OFFSET` | Current left-scrolling mode-title offset |
| `56H INPUT_CURSOR_ADDR` | Saved LCD line-1 cursor address |
| `57H INSTRUCTION_PAGE` | Page 0 through 4 |
| `58H DEV_SCROLL_OFFSET` | Developer-name scrolling offset |

## 5. Visual-state flags

| Flag | Meaning |
|---|---|
| `20H.0` | Received decimal result is negative |
| `20H.1` | Last equation character is a replaceable operator |
| `20H.3` | Equation cursor should be visible |
| `20H.4` | An animation is active |
| `20H.5` | Line 1 begins with complete `ANS` token |
| `20H.6` | Ignore normal backspace after `=` |
| `21H.0` | Software cursor-blink phase |
| `21H.1` | Instruction pages are active |

## 6. Startup sequence

`MAIN` performs the following:

1. Sets `SP = 5FH`, above the display variables.
2. Clears Port 1 and the used Port 2 outputs.
3. Initializes visual state and UART.
4. Waits for LCD power stabilization.
5. runs the standard 8-bit LCD initialization sequence;
6. calls `LCD_SOFT_POWER_OFF`.

The LCD remains off until the Process controller sends `EV_STARTUP`. At that
point `SHOW_STARTUP` clears the screen, displays `READY` near the center, waits,
then opens the mode menu.

## 7. Main receive and animation loop

`DISPLAY_LOOP` gives UART data priority over animation:

1. If `RI` is set, receive and decode the byte immediately.
2. If no animation is active, continue waiting.
3. If animation is active, wait one frame interval.
4. Check `RI` again so a newly arrived command is not overwritten by a frame.
5. Draw the next mode-title or Developer frame.

This arrangement keeps page navigation and calculator input responsive while
still using a polling loop.

## 8. UART receive and ACK

`RECEIVE_BYTE` waits for `RI`, saves `SBUF`, clears `RI`, sends `06H` through
`SBUF`, waits until the ACK transmission completes, then returns the original
byte in the accumulator.

The same routine receives both event bytes and payload bytes. Therefore a
three-byte decimal result uses three separate receive-and-ACK operations after
its event byte.

## 9. Event decoder

Bytes `00H-09H` are digits. They are converted to ASCII by adding `30H`.
All higher values are compared with the protocol constants.

### Operator events

| Event | Display output |
|---:|---|
| `C0H` | `+` |
| `C1H` | `-` |
| `C2H` | `*` |
| `C3H` | `/` |
| `C4H` | `&` |
| `C5H` | `|` |
| `C6H` | `^` |
| `C7H` | `!` |
| `C8H` | `^2` |
| `C9H` | `^` for power |
| `CAH` | `^0.5` |

### Screen and edit events

| Event | Action |
|---:|---|
| `CBH` | Open instruction pages |
| `E0H` | Show startup sequence |
| `E1H-E3H` | Select Arithmetic, Logic or Advance display mode |
| `E4H` | Clear and redraw active mode |
| `E5H` | Show mode menu |
| `E6H` | Show `ANS` token |
| `E7H` | Append `=` and hide cursor |
| `E8H/E9H` | Logic window or instruction page left/right |
| `EAH` | Normal backspace |
| `EBH` | Delete complete `ANS` token |
| `ECH` | Software power off |
| `EDH` | Backspace exposed an operator |
| `EEH` | Backspace removed an operator |
| `EFH` | Receive and display Arithmetic chain value |

## 10. Operator formatting

`DISPLAY_OPERATOR` handles operator width and replacement.

Ordinary operators occupy one LCD cell. If flag `20H.1` says the previous
character was also a binary operator, the new operator replaces it instead of
being appended.

Multi-character Advance tokens are written explicitly:

- square: `^2`, width 2;
- square root: `^0.5`, width 4.

Their width is saved in `LAST_CHAR_LEN`, allowing one DEL press to erase the
whole token.

## 11. Screen states and LCD layout

### Mode menu

```text
MODES SELECTION
1:AR 2:LOG 3:ADV
```

`SHOW_MODE_MENU` stops animations, resets the Logic buffer and hides the
cursor.

### Active modes

LCD line 1 is the equation/input line. LCD line 2 contains the animated mode
title until a result or error replaces it.

The three title strings are stored twice. `MODE_SCROLL_STEP` selects a
16-character window and increments `MODE_SCROLL_OFFSET`, producing a one-cell
left rotation each frame.

## 12. Cursor correction and software blinking

LCD hardware blink command `0FH` is not used. It previously allowed the LCD's
internal cursor timing to drift relative to title animation.

For each title frame, `MODE_SCROLL_STEP`:

1. sends `0CH` to hide the cursor;
2. redraws all 16 characters of line 2;
3. restores `INPUT_CURSOR_ADDR` on line 1;
4. alternates between `0CH` and `0EH` using `21H.0`.

The underline therefore appears only at the saved input position and changes
at the same frequency as the title frame.

## 13. Logic expression buffer and manual window

Logic mode does not write input directly to DDRAM. `LOGIC_APPEND_CHAR` stores
each character in `EXPR_BUF`, whose maximum length is 18:

```text
8 operand bits + 1 operator + 8 operand bits + 1 equals sign
```

`LOGIC_REDRAW` copies at most 16 characters from `EXPR_VIEW` to LCD line 1 and
fills the unused cells with spaces.

- When length is 16 or less, `EXPR_VIEW = 0`.
- When new input exceeds 16 cells, the view follows the right end.
- `LOGIC_SCROLL_LEFT` sets the view to the first window.
- `LOGIC_SCROLL_RIGHT` sets the view to the final window.

Keys `4` and `6` cause the Process controller to send the two scroll events.
The functions do nothing if the expression does not exceed 16 cells.

## 14. Backspace handling

### Logic mode

`HANDLE_LCD_BACKSPACE` temporarily sends LCD command `08H`, decrements
`EXPR_LEN`, rebuilds the replaceable-operator flag from the new final buffer
character and redraws the complete logical line. `LCD_CURSOR_ON` then restores
the visible display. Hiding the LCD during reconstruction prevents a stale
Arithmetic equation from flashing for part of a frame.

### Arithmetic and Advance modes

The normal path calls `ERASE_ONE_LCD_CHAR`, which checks that the cursor is not
already at `80H`. It then blanks one cell and restores the DDRAM address.

`LCD_BS_TOKEN_LOOP` continues for the remaining width of `^2` or `^0.5`. If
the line begins with `ANS`, the loop stops at address `83H`, so it cannot erase
the `S` accidentally.

### Complete `ANS`

`HANDLE_LCD_BACKSPACE_ANS` clears all three characters together. In normal
modes it returns `INPUT_CURSOR_ADDR` to `80H`; in Logic mode it resets and
redraws the expression buffer.

## 15. Intermediate Arithmetic display

`RECEIVE_CHAIN_VALUE` reads sign, high byte and low byte after event `EFH`.
`SHOW_CHAIN_VALUE` clears only line 1, prints the signed intermediate value,
updates the saved input cursor address and leaves the animated mode title
running on line 2.

This is the display half of behavior such as:

```text
1+1+1  ->  2+1
```

## 16. Result formatting

### Signed decimal

`LCD_PRINT_U16_RIGHT` repeatedly calls `DIV_TMP_BY_10`, pushes decimal digits
on the stack, calculates the required padding and prints the optional minus
sign plus digits right-aligned on line 2.

### Logic binary

`LCD_PRINT_BIN8_RIGHT` writes eight leading spaces, then rotates the result
byte and prints exactly eight `0`/`1` characters in columns 9-16.

### Division remainder

`LCD_PRINT_REMAINDER` counts the quotient and remainder digits, includes the
width of `R` and an optional minus sign, pads line 2, then prints:

```text
quotientRremainder
```

Example: `1/2=0R1`.

### Errors

`DISPLAY_ERROR` clears line 2 and selects a code-memory string:

| Code | Text |
|---:|---|
| `01H` | `ERROR:OVERFLOW` |
| `03H` | `ERROR:DIV ZERO` |
| `04H` | `ERROR:INPUT SIZE` |
| `08H` | `ERROR:MAX 8 BIT` |
| Other | `ERROR` |

## 17. Instruction pages

Advance-mode `D` causes `SHOW_OPERATOR_INSTRUCTIONS` to stop the mode title,
hide the cursor and start on page zero.

| Page | Line 1 | Line 2 |
|---:|---|---|
| 0 | `<- SCROLLING ->` | `4=LEFT 6=RIGHT` |
| 1 | `<- ARITHMETIC ->` | `A=+ B=- C=* D=/` |
| 2 | `<- 8bit-LOGIC ->` | `A=& B=| C=^ D=!` |
| 3 | `<- ADVANCE    ->` | `A=^2 B=^ C=^0.5` |
| 4 | `<- Developer  ->` | Left-scrolling developer sequence |

`SHOW_PREVIOUS_INSTRUCTION` and `SHOW_NEXT_INSTRUCTION` wrap between pages 0
and 4.

## 18. Developer scrolling implementation

The four names are stored as one 61-character circular sequence:

```text
DANIEL CHEE    GOH SHAO SEAN    TAN E-KEN    THEN MUN PIN    
```

The first 15 characters are repeated after the sequence so a 16-character
window can cross the loop boundary without special per-character wrap logic.

`DEVELOPER_SCROLL_STEP` adds `DEV_SCROLL_OFFSET` to the table address, writes
16 consecutive characters to line 2 and increments the offset. At offset 61 it
returns to zero. `MODE_SCROLL_DELAY` provides the same one-cell frame timing as
the mode titles.

## 19. Low-level LCD support

`LCD_INIT` sends the standard sequence for an HD44780-compatible module:

1. repeated `30H` wake-up commands;
2. `38H`: eight-bit, two-line mode;
3. `0CH`: display on, cursor off;
4. `06H`: increment cursor after writes;
5. `01H`: clear display.

The design uses fixed timing delays and keeps `RW` low instead of reading the
LCD busy flag.

`LCD_PUTS` reads zero-terminated strings from code memory using `MOVC` and
writes them until `00H` is reached.

## 20. Build result

The Project 12 Display firmware occupies 2,668 bytes, or 32.6% of the AT89S52
8 KB program memory. The highest used code address is `0A6BH`.
