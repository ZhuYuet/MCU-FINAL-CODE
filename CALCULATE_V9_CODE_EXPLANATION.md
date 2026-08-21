# `CALCULATE V9.asm` Code Explanation

## 1. Role of this controller

`CALCULATE V9.asm` runs on the Process AT89S52. It owns all behavior that must
remain mathematically correct regardless of how the LCD is formatted:

- scanning and debouncing the 4x4 keypad;
- reading the separate DEL/backspace button;
- selecting Arithmetic, Logic or Advance mode;
- validating and accumulating operands;
- tracking operators, `ANS`, sign, clear and power state;
- executing all ALU operations;
- detecting overflow, division-by-zero and input-size errors;
- sending semantic bytes and result payloads to the Display controller;
- waiting for one acknowledgement after every transmitted byte.

It never writes to the LCD directly.

## 2. Program organization

The source is organized into these functional regions:

1. Protocol constants and RAM allocations
2. Main key-processing loop
3. Operator and instruction dispatch
4. Backspace, clear and power control
5. Calculator state management
6. UART transmission and ACK handling
7. Keypad scanning and debounce
8. Decimal and binary accumulation
9. ALU dispatcher and operation routines
10. Error detection

Execution starts at address `0000H`, which jumps to `MAIN`.

## 3. Hardware interfaces

| Interface | Code use |
|---|---|
| Port 1 lower nibble | Drives one keypad row low at a time |
| Port 1 upper nibble | Reads keypad columns |
| `P2.0` | Active-low DEL/backspace/power button |
| `P3.0/RXD` | Receives the Display controller's `06H` ACK |
| `P3.1/TXD` | Sends digits, events and payload bytes |

`UART_INIT` configures Timer 1 in mode 2 and the serial port in UART mode 1.
With the normal 11.0592 MHz crystal and `TH1 = TL1 = FDH`, the link operates at
9600 baud.

## 4. Register and RAM state

### Working registers

| Storage | Meaning |
|---|---|
| `R2:R3` | Operand 1, accumulated result or `ANS` magnitude |
| `R4:R5` | Operand 2 |
| `R6` | Internal operator number: 1 through 11 |
| `R7` | Entry state: `0` operand 1, `1` operand 2, `2` unary operator locked |

### Main RAM variables

| Address/name | Purpose |
|---|---|
| `30H DIGIT_TMP` | Stable copy of the decoded key |
| `31H:32H TMP_H:TMP_L` | General 16-bit working value |
| `33H:34H QUO_H:QUO_L` | Division quotient |
| `35H:36H REM_H:REM_L` | Division remainder |
| `37H-3AH MC0-MC3` | Shifted multiplicand |
| `3BH-3EH PROD0-PROD3` | 32-bit multiplication product |
| `3FH:40H MUL0:MUL1` | Multiplier |
| `41H ERROR_CODE` | Current error identifier |
| `42H DIV_OVER` | Division remainder-overflow helper |
| `44H MODE` | `0` menu, `1` Arithmetic, `2` Logic, `3` Advance |
| `45H OP1_BITS` | Number of accepted bits in Logic operand 1 |
| `46H OP2_BITS` | Number of accepted bits in Logic operand 2 |
| `47H TX_BYTE` | Saved byte while waiting for ACK |
| `48H:49H` | Temporary base/input storage used by power and square root |

### Bit flags

| Flag | Meaning |
|---|---|
| `20H.0` | Error is active |
| `20H.1` | Clear was pressed; a following `#` returns to the menu |
| `20H.2` | Accumulated Arithmetic magnitude is negative |
| `20H.3` | A final result exists; a new operator should use `ANS` |
| `20H.4` | Current operand contains an accepted digit |
| `20H.5` | Operand 1 is the complete `ANS` token |
| `20H.6` | Backspace is locked after `=` |
| `20H.7` | Software power-off state |
| `21H.0` | Instruction pages are active |
| `21H.1` | Arithmetic entry began with unary minus |
| `21H.2` | A leading `+`, `*` or `/` is display-only |

## 5. Startup and main loop

`MAIN` performs the following initialization:

1. Moves the stack above the application's RAM variables with `SP = 5FH`.
2. Releases Ports 1 and 2 by writing `FFH`.
3. Initializes the UART.
4. Clears mode and calculator state.
5. Sets software power-off flag `20H.7`.

`CALC_LOOP` then repeatedly applies this priority:

1. If powered off, wait only for DEL.
2. If `P2.0` is low, handle DEL/backspace.
3. Otherwise call `KEYPAD_SCAN`.
4. If a key is found, wait for its release and dispatch it.
5. If no key is found (`FFH`), repeat.

This is a polling design; no interrupt service routine changes calculator state.

## 6. Keypad scanning

`KEYPAD_SCAN` first checks whether any column is active. If so, it drives each
row low in turn:

| Row output | Returned keys |
|---|---|
| `P1 = FEH` | `1`, `2`, `3`, `A` |
| `P1 = FDH` | `4`, `5`, `6`, `B` |
| `P1 = FBH` | `7`, `8`, `9`, `C` |
| `P1 = F7H` | `*`, `0`, `#`, `D` |

The decoded values are:

- digits: `00H-09H`;
- `A-D`: `0AH-0DH`;
- `*`: `0EH`;
- `#`: `0FH`;
- no key: `FFH`.

`DEBOUNCE_DELAY` confirms a press and `WAIT_KEY_RELEASE` prevents a held key
from being entered repeatedly.

## 7. Key dispatcher

### Menu state

When `MODE = 0`, only `1`, `2` and `3` are accepted by `SELECT_MODE`:

- `1` sends `EV_MODE_AR`;
- `2` sends `EV_MODE_LOG`;
- `3` sends `EV_MODE_ADV`.

The selected mode is stored before `RESET_STATE` is called, so the ALU state is
cleared without losing the mode number.

### Active mode

The dispatcher checks keys in this order:

1. Clear/menu sequence
2. Logic manual-scroll keys
3. Digits
4. Equals
5. Mode-specific operators

In Logic mode, `4` and `6` are reserved for left/right display-window control.
Digits `2-9` are ignored before they reach the accumulator.

## 8. Operand accumulation

### Decimal accumulation

`ACCUMULATE_DIGIT` implements:

```text
new_value = old_value * 10 + digit
```

`MUL10_TMP_CHECK` performs the multiplication in two bytes and reports carry if
the value would exceed `FFFFH`. The attempted digit is sent to the LCD first,
then error code `04H` is sent.

### Binary accumulation

`ACCUMULATE_BINARY` shifts the current byte left and ORs in the new bit:

```text
new_value = (old_value << 1) | new_bit
```

`OP1_BITS` and `OP2_BITS` enforce the eight-bit limit. A ninth bit produces
error code `08H`.

## 9. Operator decoding

The Process controller converts keypad letters into internal operators and
protocol events.

| Internal `R6` | Event | Operation |
|---:|---:|---|
| 1 | `C0H` | Add |
| 2 | `C1H` | Subtract |
| 3 | `C2H` | Multiply |
| 4 | `C3H` | Divide |
| 5 | `C4H` | AND |
| 6 | `C5H` | OR |
| 7 | `C6H` | XOR |
| 8 | `C7H` | NOT |
| 9 | `C8H` | Square |
| 10 | `C9H` | Power |
| 11 | `CAH` | Integer square root |

`R7` changes to `1` for a binary operator and to `2` for a unary operator.

## 10. Leading Arithmetic operators

`MODE_OPERATOR_IN_RANGE` gives the beginning of an Arithmetic expression
special treatment:

- leading `-` is stored as subtraction from zero and flagged as unary minus;
- leading `+`, `*` and `/` are sent to the LCD but are not stored in `R6/R7`;
- replacing unary minus with another leading operator clears the hidden ALU
  state before displaying the replacement.

This separation is why `*5+1=` can remain visible while the ALU calculates
`5+1=6`, instead of calculating `0*5+1=1`.

## 11. Intermediate chaining and `ANS`

`PREPARE_OPERATOR_ENTRY` runs before a new operator is decoded.

If operand 2 is complete, it calls `EXECUTE_MATH`. Arithmetic mode then sends
`EV_CHAIN_VALUE` with the real signed intermediate value and keeps that value
in `R2:R3` as the next operand 1. Thus:

```text
1+1+1  ->  2+1
```

After final `=`, flag `20H.3` causes `PREPARE_ANS_EXPRESSION` to send `EV_ANS`.
The previous result remains in `R2:R3`; operand 2 and operator state are reset.

## 12. Equals and result packets

`HASH_EXECUTE` ignores an incomplete binary expression. Otherwise it:

1. sends `EV_EQUALS`;
2. locks backspace with flags `20H.3` and `20H.6`;
3. clears the old error code;
4. calls `EXECUTE_MATH`;
5. sends an error packet or the appropriate result packet.

`SEND_RESULT` chooses the payload by mode and operator:

| Result type | Packet |
|---|---|
| Arithmetic/Advance decimal | `D0H`, sign, high byte, low byte |
| Logic binary | `D1H`, result byte |
| Division | `D3H`, sign, quotient high/low, remainder high/low |

## 13. ALU implementation

`EXECUTE_MATH` validates `R6`, converts the 1-based operator number to a
three-byte jump-table offset and jumps through `ALU_JUMP_TABLE`.

### Addition and subtraction

`MATH_ADD` and `MATH_SUB` operate on 16-bit magnitudes. They compare magnitudes
when a negative accumulated value is involved and update `20H.2` separately.
Carry out of the high byte produces Arithmetic overflow.

### Multiplication

`MATH_MUL` uses a 16-cycle shift-and-add algorithm. The temporary product is
32 bits. If either upper product byte is nonzero, the 16-bit result would not
fit and error `01H` is raised.

### Division

`MATH_DIV` first rejects a zero divisor. It then performs 16 iterations of
restoring binary division. The quotient replaces `R2:R3`; the remainder stays
in `REM_H:REM_L` for the `R` result packet.

### Logic

`LOGIC_AND`, `LOGIC_OR`, `LOGIC_XOR` and `LOGIC_NOT` operate on the low operand
bytes and clear the unused high result byte.

### Square

`MATH_SQUARE` copies operand 1 to operand 2 and reuses the multiplication
routine, including its overflow detection.

### Power

`MATH_POWER` uses repeated multiplication. The original base is stored at
`48H:49H`; the exponent in `R4:R5` is decremented until complete. An exponent
of zero directly returns one.

### Integer square root

`MATH_SQRT` increments an 8-bit candidate in `R0`, squares it with `MUL AB` and
compares it with the 16-bit input. It returns an exact root when found or the
previous candidate after overshoot, producing the floor of the square root.

## 14. Error handling

| Code | Routine | Meaning |
|---:|---|---|
| `01H` | `ARITH_OVERFLOW` | 16-bit result overflow |
| `03H` | `DIVIDE_BY_ZERO` | Zero divisor |
| `04H` | `INPUT_TOO_LARGE` | Decimal operand exceeds `FFFFH` |
| `08H` | `BINARY_TOO_LONG` | More than eight Logic bits |

`SET_ERROR` sets `20H.0`. `SEND_ERROR` transmits `EV_ERROR` followed by the
error code. The error remains part of Process state until clear/reset.

## 15. Backspace and clear state synchronization

`HANDLE_BACKSPACE` edits the numeric state before sending a matching display
event:

- decimal deletion divides the current operand by 10;
- binary deletion shifts the current operand right and decrements its bit count;
- deleting the last operand-2 digit exposes the operator;
- deleting again removes the operator and returns to operand 1;
- unary `^2` or `^0.5` removes the full operator state;
- `ANS` deletion clears its complete token and result state;
- a display-only leading operator has its own flag and removal event.

After `=`, DEL calls `CLEAR_CURRENT` instead of modifying the completed result.

The three display-edit events distinguish what the LCD must do:

| Event | Display meaning |
|---|---|
| `EAH EV_BACKSPACE` | Remove a normal character |
| `EDH EV_BS_SHOW_OP` | Remove the final operand digit and expose an operator |
| `EEH EV_BS_REMOVE_OP` | Remove the operator itself |

## 16. Power and instruction states

In the mode menu, DEL toggles software power. Power-off sends `EV_POWER_OFF`;
power-on resets state and sends `EV_STARTUP`.

Advance key `D` resets the current expression, sets instruction flag `21H.0`
and sends `EV_INSTRUCTIONS`. While active:

- `4` sends previous-page event `E8H`;
- `6` sends next-page event `E9H`;
- `*`, `D` or DEL clears state and sends `EV_CLEAR`;
- all other keys return without changing the ALU.

## 17. UART send-and-ACK protocol

`SEND_BYTE` saves the outgoing byte in `TX_BYTE`, writes it to `SBUF`, waits for
`TI`, then waits until `RI` contains `06H`. Only then does it restore the sent
byte to the accumulator and return.

Multi-byte packets call `SEND_BYTE` once per byte. This prevents a fast Process
controller from overwriting a byte before the Display controller has consumed
and acknowledged it.

## 18. Build result

The Project 12 Process firmware occupies 2,094 bytes, or 25.6% of the AT89S52
8 KB program memory. The highest used code address is `082DH`.
