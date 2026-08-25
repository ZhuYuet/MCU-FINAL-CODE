# LCD Event Code Test List

This list tests the Project 12 display-controller event decoder. Send bytes in
the order shown. When testing through UART, wait for the normal ACK after every
byte. In the standalone display test, the same sequences can be placed after
`TEST_VECTOR:` as `DB` statements.

## Event code reference

| Hex | Symbol | LCD function |
|---|---|---|
| `00-09` | Digit data | Display ASCII digit `0-9` |
| `C0` | `EV_OP_ADD` | Display `+` |
| `C1` | `EV_OP_SUB` | Display `-` |
| `C2` | `EV_OP_MUL` | Display `*` |
| `C3` | `EV_OP_DIV` | Display `/` |
| `C4` | `EV_OP_AND` | Display `&` |
| `C5` | `EV_OP_OR` | Display `|` |
| `C6` | `EV_OP_XOR` | Display `^` |
| `C7` | `EV_OP_NOT` | Display `!` |
| `C8` | `EV_OP_SQUARE` | Display `^2` |
| `C9` | `EV_OP_POWER` | Display `^` |
| `CA` | `EV_OP_SQRT` | Display `^0.5` |
| `CB` | `EV_INSTRUCTIONS` | Open instruction pages |
| `D0` | `EV_RESULT_DEC` | Decimal result; payload: sign, high, low |
| `D1` | `EV_RESULT_BIN` | 8-bit binary result; payload: value |
| `D2` | `EV_ERROR` | Error display; payload: error code |
| `D3` | `EV_RESULT_REM` | Division result; payload: sign, quotient high/low, remainder high/low |
| `E0` | `EV_STARTUP` | LCD on, READY, then mode menu |
| `E1` | `EV_MODE_AR` | Enter Arithmetic mode |
| `E2` | `EV_MODE_LOG` | Enter 8-bit Logic mode |
| `E3` | `EV_MODE_ADV` | Enter Advance mode |
| `E4` | `EV_CLEAR` | Clear and redraw the active mode |
| `E5` | `EV_MENU` | Display mode-selection menu |
| `E6` | `EV_ANS` | Display complete `ANS` token |
| `E7` | `EV_EQUALS` | Display `=` and hide cursor |
| `E8` | `EV_SCROLL_LEFT` | Scroll logic expression/help page left |
| `E9` | `EV_SCROLL_RIGHT` | Scroll logic expression/help page right |
| `EA` | `EV_BACKSPACE` | Delete one normal character/token cell |
| `EB` | `EV_BACKSPACE_ANS` | Delete the complete `ANS` token |
| `EC` | `EV_POWER_OFF` | Turn LCD display off |
| `ED` | `EV_BS_SHOW_OP` | Backspace exposes an operator; allow replacement |
| `EE` | `EV_BS_REMOVE_OP` | Backspace removes the exposed operator |
| `EF` | `EV_CHAIN_VALUE` | Arithmetic intermediate value; payload: sign, high, low |

## Result and error payloads

`sign = 00H` means positive and `sign = 01H` means negative.

```asm
; Decimal +123
DB 0D0H, 00H, 00H, 7BH

; Decimal -42
DB 0D0H, 01H, 00H, 2AH

; Binary 10100101
DB 0D1H, 0A5H

; Division +5/2 result: 2R1
DB 0D3H, 00H, 00H, 02H, 00H, 01H

; Arithmetic chain value +12
DB 0EFH, 00H, 00H, 0CH
```

Error payload codes:

| Sequence | Expected line 2 |
|---|---|
| `D2 01` | `ERROR:OVERFLOW` |
| `D2 03` | `ERROR:DIV ZERO` |
| `D2 04` | `ERROR:INPUT SIZE` |
| `D2 08` | `ERROR:MAX 8 BIT` |
| `D2 FF` | `ERROR` |

## Ordered functional tests

Run each test separately or insert an `E4` clear event between tests.

| Test | Raw byte sequence | Expected behavior |
|---|---|---|
| Startup/menu | `E0` | `READY`, then `MODES SELECTION / 1:AR 2:LOG 3:ADV` |
| Arithmetic add | `E1 01 C0 02 E7 D0 00 00 03` | `1+2=` and decimal result `3` |
| Arithmetic subtract | `E1 09 C1 04 E7 D0 00 00 05` | `9-4=` and result `5` |
| Arithmetic multiply | `E1 06 C2 07 E7 D0 00 00 2A` | `6*7=` and result `42` |
| Division remainder | `E1 05 C3 02 E7 D3 00 00 02 00 01` | `5/2=` and result `2R1` |
| Logic AND/scroll | `E2 01 00 01 00 01 00 01 00 C4 01 01 00 00 01 01 00 00 E7 E8 E9 D1 88` | Full logic expression scrolls; result `10001000` |
| Logic operators | `E2 01 C4 00 E4 01 C5 00 E4 01 C6 00 E4 C7 01` | Shows `&`, `|`, `^`, and `!` |
| Square | `E3 05 C8 E7 D0 00 00 19` | `5^2=` and result `25` |
| Power | `E3 02 C9 03 E7 D0 00 00 08` | `2^3=` and result `8` |
| Square root | `E3 09 CA E7 D0 00 00 03` | `9^0.5=` and result `3` |
| Normal backspace | `E1 01 02 03 EA` | `123` becomes `12` |
| Operator backspace | `E1 01 C0 02 ED EE` | Removes `2`, exposes `+`, then removes `+` |
| ANS deletion | `E1 E6 EB` | Displays `ANS`, then removes all three characters |
| Intermediate chain | `E1 EF 00 00 0C` | Line 1 changes to `12` |
| Instruction pages | `E3 CB E9 E9 E9 E9 E8` | Opens pages, moves right four times, then left once |
| Menu | `E5` | Returns to the two-line mode menu |
| Power cycle | `EC E0` | LCD turns off, then starts and returns to menu |

## Copyable standalone test vector

Replace the existing `TEST_VECTOR` block with this shorter regression set:

```asm
TEST_VECTOR:
    ; Startup and menu
    DB EV_STARTUP

    ; Arithmetic: 1+2=3
    DB EV_MODE_AR
    DB 01H, EV_OP_ADD, 02H, EV_EQUALS
    DB EV_RESULT_DEC, 00H, 00H, 03H

    ; Division: 5/2=2R1
    DB EV_CLEAR
    DB 05H, EV_OP_DIV, 02H, EV_EQUALS
    DB EV_RESULT_REM, 00H, 00H, 02H, 00H, 01H

    ; Logic expression and binary result
    DB EV_MODE_LOG
    DB 01H,00H,01H,00H,01H,00H,01H,00H
    DB EV_OP_AND
    DB 01H,01H,00H,00H,01H,01H,00H,00H
    DB EV_EQUALS, EV_SCROLL_LEFT, EV_SCROLL_RIGHT
    DB EV_RESULT_BIN, 088H

    ; Advance-mode tokens
    DB EV_MODE_ADV
    DB 05H, EV_OP_SQUARE, EV_EQUALS
    DB EV_RESULT_DEC, 00H, 00H, 019H
    DB EV_CLEAR
    DB 02H, EV_OP_POWER, 03H, EV_EQUALS
    DB EV_RESULT_DEC, 00H, 00H, 08H
    DB EV_CLEAR
    DB 09H, EV_OP_SQRT, EV_EQUALS
    DB EV_RESULT_DEC, 00H, 00H, 03H

    ; Backspace and ANS-token deletion
    DB EV_MODE_AR, 01H, EV_OP_ADD, 02H
    DB EV_BS_SHOW_OP, EV_BS_REMOVE_OP
    DB EV_CLEAR, EV_ANS, EV_BACKSPACE_ANS

    ; All error messages
    DB EV_ERROR, 01H
    DB EV_ERROR, 03H
    DB EV_ERROR, 04H
    DB EV_ERROR, 08H

    ; Intermediate value and instruction navigation
    DB EV_MODE_AR, EV_CHAIN_VALUE, 00H, 00H, 0CH
    DB EV_MODE_ADV, EV_INSTRUCTIONS
    DB EV_SCROLL_RIGHT, EV_SCROLL_RIGHT, EV_SCROLL_RIGHT, EV_SCROLL_RIGHT
    DB EV_SCROLL_LEFT

    ; Finish on the mode menu
    DB EV_MENU
TEST_VECTOR_END:
TEST_VECTOR_SIZE EQU TEST_VECTOR_END - TEST_VECTOR
```
