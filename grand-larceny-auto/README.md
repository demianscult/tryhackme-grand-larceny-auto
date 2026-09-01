# Grand Larceny Auto I — TryHackMe Write-up

- **Platform:** TryHackMe
- **Difficulty:** Medium
- **Category:** Reverse Engineering / Game Hacking
- **Engine:** Godot / C#

## 1. Overview

Grand Larceny Auto is a small 3D crime game where the objective is to open a supposedly inaccessible vault.

Since the challenge description hinted at decompilation and debugging, I decided to inspect the provided files instead of relying only on normal gameplay.

## 2. Inspecting the Game Files

Inside the Windows data directory, I found several .NET runtime libraries, `GodotSharp.dll`, and the main game assembly:

* `GrandLarcenyAuto.dll`
  
<img width="1571" height="118" alt="image" src="https://github.com/user-attachments/assets/90b4e434-4faa-4189-ab5c-3c2d75d3e0af" />



I then disassembled the assembly using `monodis`:

```bash
monodis --output=gla1_code.il GrandLarcenyAuto.dll
```

This generated a CIL representation of the game's managed code.

## 3. Finding the Vault Logic

Next, I searched the disassembled code for the method responsible for opening the vault:

```bash
grep -n -C 15 "TryOpen" gla1_code.il
```

The method `SafehouseVault::TryOpen()` retrieves the player's wanted level and compares it against the value `6`:

```text
callvirt instance int32 class GrandLarcenyAuto.PlayerState::get_WantedStars()
ldc.i4.6
blt IL_00cc
```

The `SafehouseVault` class also defines:

```text
UnlockStars = 6
```

The game even contains the following failure message:

```text
The vault stays shut. (You need SIX stars.)
```

This confirmed that the vault requires **six wanted stars** to open.

## 4. The Logic Flaw

After that, I searched for the wanted system implementation:

```bash
grep -n -C 10 "MaxStars" gla1_code.il
```

Inside `WantedSystem`, I found the following field:

```text
MaxStars = 5
```

<img width="1917" height="636" alt="image" src="https://github.com/user-attachments/assets/4acca9ac-f7c3-4a49-9d38-25a435d21054" />


This reveals the core logic flaw:

* The vault requires **6 stars**
* The wanted system caps the player at **5 stars**

That makes the vault condition impossible to satisfy through normal gameplay.

## 5. Recovering the Flag

After identifying the six-star requirement, I continued inspecting `SafehouseVault::TryOpen()`.

The method passes the player's wanted level to:

```text
CryptoUtil::DeriveKey(int32)
```

The `SafehouseVault` class also contains a static byte array named `SealedBlob`.

This showed that the value used by the vault was also involved in recovering the protected data.

From the previous analysis, I already knew that the expected value was `6`. I then used this value during the flag-recovery step.

<img width="1328" height="70" alt="image" src="https://github.com/user-attachments/assets/352dec0e-67f1-45b0-bf8f-0b77081b5a3c" />

The recovered flag was:

```text
THM{h0tf1x3d_my_0wn_w4nt3d_l3v3l}
```

## 6. What I Learned

This challenge was a great introduction to reverse engineering managed game code.

Instead of solving the room only through gameplay, inspecting the .NET assembly exposed the internal business logic of the application. The key finding was a contradiction between two client-side rules: the vault requires six wanted stars, while the wanted system only allows five.

It also reinforced an important security lesson: sensitive logic should not rely entirely on client-side enforcement, since distributed applications can be inspected and analyzed.
