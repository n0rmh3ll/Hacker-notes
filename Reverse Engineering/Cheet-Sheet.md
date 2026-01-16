
## 🛠️ Reverse Engineering Cheat Sheet

### 1. The Discovery Phase (Outside the program)

Use these to find static clues without running the code fully.

- **`strings <file>`**: Finds all human-readable text. Look for "Correct!", "Wrong!", or hidden keys.
- **`ltrace ./<file>`**: Shows library calls (like `memcmp`, `read`, `printf`).
    - _Key Insight:_ Watch the `memcmp(addr1, addr2, len)` call. It’s the moment the program compares your guess to the answer.
- **`objdump -d <file> | less`**: Shows the assembly code. Look for `xor`, `rev`, or `sort` loops.
### 2. The GDB "Surgery" (Inside the program)

Use these when the program is running to look at memory.

|**Command**|**Action**|**Purpose**|
|---|---|---|
|**`gdb ./file`**|Start GDB|Prepares the environment.|
|**`break memcmp`**|Set Breakpoint|Stops the program right before it checks your input.|
|**`run`**|Execute|Starts the program.|
|**`x/25xb $rdi`**|Examine RDI|Shows **YOUR input** after it was mangled.|
|**`x/25xb $rsi`**|Examine RSI|Shows the **TARGET** (the answer key) in memory.|
|**`disas main`**|Disassemble|Shows the assembly instructions for the main function.|

### 3. The XOR Logic (The Math)

XOR is "symmetrical." If you know two parts, you can always find the third.

- **Formula:** $Input \oplus Key = Result$
- **To find the Key:** $Input \oplus Result = Key$
- **To find the Password:** $Target \oplus Key = Password$

Why use "AAAA" (0x41)?

If you send 25 As and x/25xb $rdi shows 0x4b 0x6e 0xde...:

1. **Byte 1 Key:** $0x41 \oplus 0x4b = 0x0a$
2. **Byte 2 Key:** $0x41 \oplus 0x6e = 0x2f$
3. Byte 3 Key: $0x41 \oplus 0xde = 0x9f$
    Conclusion: The key is 3 bytes long: 0x0a 0x2f 0x9f.

### 4. Handling Manglers (The Order)

When multiple manglers are used, you must undo them like a stack (Last-In, First-Out).

1. **Swap:** Swap the indexes back.
2. **Reverse:** Flip the string back.
3. **Sort:** Usually can be ignored if you just need the _set_ of bytes (since sorting happens at the end)
4. **XOR:** Apply the keys found in Step 3.