# WRITE UP: [RE] CrackNotMe's CrackmesForBeginners (CFB) #2
**Difficulty:** 2.0  
**Quality:** 4.5  

## Challenge Overview
<img width="1022" height="277" alt="image" src="https://github.com/user-attachments/assets/2dbea421-3f70-4d10-a8b2-c7752356c7b0" />

You can find this challenge [here](https://crackmes.one/crackme/6a154cab17539b5175d1238a).

### Description
This is the fourth challenge from the Crackmes for Beginners (CFB) series.

Are you ready to dive into custom cryptanalysis? This level features a stateful 3-Rotor Cipher with Ciphertext Feedback, closely inspired by the famous Enigma machine.

The password is encrypted dynamically, and the shifts of the rotors mutate at each step based on the generated output. To solve it, you will need to analyze the mathematical transformations of each rotor, identify the structural weakness in the dynamic feedback loop, and write a Python decryption script to recover the 13-character activation password.

---

## Information Gathering & Static Analysis

After loading the file into Ghidra, I started at the `entry` point, which led me to the C Runtime (CRT) initialization function: `__scrt_common_main_seh()`. 

By scrolling down to the bottom of this CRT function, I found the call to the actual `main` function (which takes 3 parameters: `argc`, `argv`, `envp`) right before the `_cexit()` call. Ghidra named it `main`.

```c
// Inside __scrt_common_main_seh()
uVar6 = main(*puVar10,uVar1,uVar8);
```

### Analyzing the Main Logic

Jumping into `main`, the decompiled code is quite long, but we can quickly see the interesting string in code like:

```C

  FUN_140001220(&DAT_14003f600,"===================================================\n");
  FUN_140001220(&DAT_14003f600,"            Crackme #4                             \n");
  FUN_140001220(&DAT_14003f600,"           [+] by pwn.by [+]                      \n");
  FUN_140001220(&DAT_14003f600,"         --> pwned.space <--                      \n");
  FUN_140001220(&DAT_14003f600,"===================================================\n\n");
  FUN_140001220(&DAT_14003f600,"[*] Enter activation password (exactly 13 chars):\n");
  FUN_140001220(&DAT_14003f600,"[+] Password: ");

```

So that, we find all string in the code and we see the important string `"[*] Enter activation password (exactly 13 chars):"`,`"You have successfully solved CFB4!"` and `"[+] ACCESS GRANTED! Congratulations!"`

Those strings are inside the `if`

```C

      if (((byte)(((cVar15 + b26 + *(byte *)((longlong)pass + 0xc) ^ 0x3a) + 0x13 ^ 0x7f) -
                 (b26 ^ check1)) == '\x7f') &&
         (b26 == 0x98 &&
          (b17 == 0x35 &&
          (b16 == 0x3f &&
          (b15 == 0x52 &&
          (b14 == 0x54 &&
          (b13 == 0xfa &&
          (b12 == 0xb7 &&
          (b11 == 0x9e && (b10 == 0x6e && (b9 == 0x2b && (b30 == 0xb7 && b28 == 0xc6)))))))))))) {
        pcVar14 = "   You have successfully solved CFB4!              \n";
        pcVar12 = "   [+] ACCESS GRANTED! Congratulations!            \n";
      }

```

Scrolling up, we find the find the code which check the length of the pass and we find this:

```C

if (length == 0xd) {
.....
}

```

So that, the logic of this program is in this.

But in the `if`, the last character is not in it. However, the `cVar15` and `check1` is updated after changing the raw passwword and they aren't updated in the last one. In the `if` to check to solve this challenge, we can see the code like `(byte)(((cVar15 + b26 + *(byte *)((longlong)pass + 0xc)`, it has the logic like the update of `cVar15` and `check1` after after changing the raw passwword. That we find the hex of last character is 0xc.

I have wrote the python-code to solve this logic to solve this challenge:

```Python

def decrypt_password():
    targets = [0xc6, 0xb7, 0x2b, 0x6e, 0x9e, 0xb7, 0xfa, 0x54, 0x52, 0x3f, 0x35, 0x98]
    addends = [0x15, 0x15, 0x16, 0x18, 0x1b, 0x1f, 0x24, 0x2a, 0x31, 0x39, 0x42, 0x4c]
    
    password = ""
    cVar15 = 0
    check1 = 0
    
    for i in range(12):
        target_b = targets[i]
        add = addends[i]
        
        v = target_b ^ 0xa5
        v = (v - add) & 0xFF
        v = v ^ 0x5c
        
        if i == 0:
            v = (v + 0xd) & 0xFF
        else:
            v = (v + check1) & 0xFF
            
        v = v ^ 0x7f
        v = (v - 0x13) & 0xFF
        v = v ^ 0x3a
        
        if i == 0:
            p = (v - 5) & 0xFF
            check1_int = target_b ^ 0xa5
            check1 = (check1_int ^ 0xa8) & 0xFF
            cVar15 = (target_b + 5) & 0xFF
        else:
            p = (v - cVar15) & 0xFF
            cVar15 = (cVar15 + target_b) & 0xFF
            check1 = (check1 ^ target_b) & 0xFF
            
        password += chr(p)

    v13 = (0x7f + check1) & 0xFF
    v13 = v13 ^ 0x7f
    v13 = (v13 - 0x13) & 0xFF
    v13 = v13 ^ 0x3a
    p13 = (v13 - cVar15) & 0xFF
    
    password += chr(p13)
        
    return password

if __name__ == '__main__':
    password = decrypt_password()
    print(f"Mật khẩu 13 ký tự là: {password}")

```

The password to solve this challenge is  `rotors_spin_9`
