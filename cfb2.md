# WRITE UP: [RE] CrackNotMe's CrackmesForBeginners (CFB) #2
**Difficulty:** 2.0  
**Quality:** 4.5  

## Challenge Overview
![image](https://github.com/user-attachments/assets/e844675f-43a7-42db-bb8a-67887c90e0c8)

You can find this challenge [here](https://crackmes.one/crackme/6a15496417539b5175d12386).

### Description
This is the second challenge from the Crackmes for Beginners (CFB) series. This time, there are no dynamic string calculations or mathematical equations. Instead, you are dropped into a hidden 10x10 maze stored in memory. 
Your goal is to reverse-engineer the binary, extract the maze layout from the `.rdata` section, identify the movement mechanics, and find the correct WASD path from start to finish.

---

## Information Gathering & Static Analysis

After loading the file into Ghidra, I started at the `entry` point, which led me to the C Runtime (CRT) initialization function: `__scrt_common_main_seh()`. 

By scrolling down to the bottom of this CRT function, I found the call to the actual `main` function (which takes 3 parameters: `argc`, `argv`, `envp`) right before the `_cexit()` call. Ghidra named it `FUN_140006370`.

```c

uVar6 = FUN_140006370(*puVar10, uVar1, uVar8

```

### Analyzing the Main Logic
Jumping into `FUN_140006370`, the decompiled code is quite long, but we can quickly narrow down the core logic by looking for interesting strings like `"[-] Only W, A, S, D are allowed.\n"`, `"[-] Hit a wall at step"`, and `"[+] ACCESS GRANTED! Congratulations!"`.

I renamed some variables in Ghidra to make the code readable. The program processes our input character by character and updates two variables, which clearly represent the **X and Y coordinates** of the player:

```c
// Processing movement (WASD)
iVar7 = toupper((uint)*(byte *)(input_buffer + loop_index));
char move = (char)iVar7;

if (move == 'A') {
    x_pos = x_pos - 1;       
}
else if (move == 'D') {
    x_pos = x_pos + 1;      
}
else if (move == 'S') {
    y_pos = y_pos + 1;       
}
else if (move == 'W') {
    y_pos = y_pos - 1;   
}
else {
    print("[-] Only W, A, S, D are allowed.\n");
    // ... exit logic ...
}
```

### The Maze Structure
After updating the coordinates, the program checks our new position against a data structure in memory at `&DAT_14002b3c0` (I'll call it `maze_map`). 

```c
// Formula to calculate 1D array index from 2D coordinates: x + (y * 10)
// This confirms the maze is a 10x10 grid.

// Check for WALL (0x01)
if (maze_map[(int)(x_pos + y_pos * 10)] == '\x01') {
    print("\n[-] Hit a wall at step ");
    // ... fail logic ...
}

// Check for FINISH LINE (0x02)
if ((maze_map[(int)(x_pos + y_pos * 10)] == '\x02') && (loop_index == input_len - 1)) {
    is_finished = true;
}
```

From this snippet, we know:
*   The maze is a **10x10 grid** (due to `y_pos * 10`).
*   Path/Empty space = `0x00`.
*   Wall = `0x01`.
*   Destination = `0x02`.
*   The player starts at `(0, 0)` and must reach `(9, 9)`.

### Extracting the Maze
By double-clicking `maze_map` (`DAT_14002b3c0`), we can view the raw hex data in Ghidra's Listing view:

![image](https://github.com/user-attachments/assets/cd610d87-061d-4eb2-8116-e9b430b440f6)

Formatting this 100-byte block into a 10x10 matrix makes the maze visible:

```text
{0, 1, 1, 1, 1, 1, 1, 1, 1, 1}, 
{0, 0, 0, 1, 0, 0, 0, 0, 0, 1}, 
{1, 1, 0, 1, 0, 1, 1, 1, 0, 1}, 
{1, 0, 0, 0, 0, 1, 0, 0, 0, 1}, 
{1, 0, 1, 1, 1, 1, 0, 1, 1, 1}, 
{1, 0, 0, 0, 1, 0, 0, 0, 0, 1}, 
{1, 1, 1, 0, 1, 1, 1, 1, 0, 1}, 
{1, 0, 0, 0, 0, 0, 0, 1, 0, 1}, 
{1, 0, 1, 1, 1, 1, 0, 1, 0, 0},
{1, 1, 1, 1, 1, 1, 0, 0, 0, 2}  
```

## Solution
By simply tracing the `0` values from the top-left `(0,0)` to the bottom-right `(9,9)`, we can map out the correct sequence of inputs.

The correct path is: **`SDDSSASSDDSSDDDSSDDD`**

![image](https://github.com/user-attachments/assets/91805622-5cc9-4d86-89e7-2f536150c6e1)

**Challenge solved!**
