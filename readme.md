# Analysis of the Sega X-MEN Reset Logic (Genesis / Mega Drive)

> A hardware-level look at the infamous “Reset the computer!” puzzle from *X-MEN* on Sega Genesis / Mega Drive.

---

## What is this repo about?

This repository is a short write–up about how one of the strangest puzzles in the 16‑bit era actually works under the hood.

In the 1993 *X-MEN* game for the Sega Genesis / Mega Drive, there is a stage where the game keeps yelling **“RESET THE COMPUTER!”** at you. The problem: there is no in‑game reset button, option, or menu.

The real solution is to **tap the physical RESET button on the console**.

This document explains, at a high level but with real technical details, how that trick is implemented in terms of:

- the Genesis / Mega Drive reset circuitry
- how the 68000 CPU boots
- how the game uses RAM and a “magic value” to detect that you pressed RESET instead of power‑cycling the console

The goal is to stay at around **C1‑level English**: not overly academic, but still precise and technical enough for people who enjoy this kind of hardware/software archaeology.

---

## The infamous “RESET THE COMPUTER!” moment

Near the end of the game, after the Mojo stage, you reach a computer terminal. Your party keeps saying some version of:

> **“Reset the computer!”**

Naturally, you look for some object to interact with, a menu entry, a hidden switch… and you find nothing. The manual does not help either. For many kids back then, this looked like a softlock or a bug, and a lot of people simply gave up.

What the game actually wants you to do is **tap the physical RESET button on the console**:

- If you **tap** RESET briefly, you will return to the boot sequence and then jump straight to the final stage.
- If you **hold** RESET for too long or power‑cycle the console, the game just starts from the very beginning again.

So how does the game even know the difference between those situations?

---

## What the RESET button really does

On the Mega Drive / Genesis, the RESET button does **not** cut the power. Instead, it sends a reset signal to the CPU and other chips, putting them into a “freshly started” state while **keeping the power rails alive**.

This means:

- the **68000 CPU** and several peripherals are reset
- some internal registers are re‑initialized
- but as long as the power is stable, **main RAM keeps its contents**

In other words, after a RESET:

- the system behaves as if you just powered it on
- but the **RAM may still contain whatever data was there before**

The game can use this to distinguish between:

- “I was running, then the RESET line was asserted, but power stayed on”
- versus “I was never running before, this is a clean cold boot”

The trick is to leave a small marker in RAM before asking the player to reset the game.

---

## The magic value in RAM

If we look at the game’s startup code (disassembled 68000 assembly), we can see a check that happens early in the boot sequence:

```asm
cmpi.l  #$57425554, $00FF0800.l  ; compare RAM[0xFF0800] with 0x57425554
bne.b   @notmagic               ; if different, do a normal boot
jmp     $027FE0                 ; if equal, jump to special “late game” code
```

The value `0x57425554` is a 32‑bit constant. You can think of it as a **magic number** the game uses to mark a special state.

Somewhere in the code near the “RESET THE COMPUTER!” scene, the game writes that magic value into RAM at address **`0xFF0800`**. After that, it waits for the player to press RESET.

When you **tap** RESET:

1. The reset signal re‑initializes the CPU and some glue logic.
2. **Power stays on**, so the RAM contents at `0xFF0800` are not wiped.
3. The startup code runs again from the internal ROM / boot vector.
4. The boot code compares `RAM[0xFF0800]` with the magic number.
5. If the value matches, the code knows:
   > “This is not a totally fresh boot. The game asked the player to reset, and RAM still contains the marker.”

So instead of starting the game from the title screen, it **jumps directly into the late‑game sequence**, effectively treating the physical RESET button as an in‑game puzzle action.

---

## Why does holding RESET sometimes “erase” your progress?

Players often reported that if they **held** RESET for too long, the trick did not work. Instead, the game behaved like a full cold boot and the marker was gone.

Strictly speaking, the game cannot measure “how long” you held the button. When the RESET line is active, the CPU is not running at all, so there is no timing logic inside the game code itself.

However, a few plausible effects can explain this behaviour:

- **Power instability**: if the power rail sags or bounces while you hold RESET, RAM may drop below a safe voltage and lose its contents.
- **Glitchy reset circuits**: depending on the console revision or wear, a long or noisy button press might make the reset circuitry misbehave, causing extra resets or partial initialization.
- **Code paths that clear RAM**: on some boot paths, the firmware or game might run routines that clear parts of RAM. If this happens after the magic value was written, the marker at `0xFF0800` will be overwritten.

From the player’s point of view, this all collapses into one simple rule of thumb:

> **Tap** RESET quickly to solve the puzzle.  
> If you hold it or power‑cycle, you risk losing the magic value and your progress.

---

## Bonus: why the Nomad version is basically unwinnable

The handheld **Genesis Nomad** does not have a physical RESET button.

That means:

- you can still reach the “RESET THE COMPUTER!” scene
- but you have no way to generate the kind of RESET signal that the game is expecting
- so you cannot put the system into the special “marker is still in RAM, code runs from the beginning” state

In practice, this makes that particular late‑game puzzle **impossible** on real Nomad hardware, unless you use some external trick (like rewinding in an emulator, patching the ROM, or hardware mods).

It is one of those charming, slightly cursed design choices from the era where developers assumed a very specific physical console in front of the player.

---

## TL;DR

- The *X-MEN* Genesis game has a late‑game puzzle that literally asks you to **press the console’s RESET button**.
- The Mega Drive / Genesis RESET line **re‑initializes** the system but **does not cut power**, so RAM can survive a reset.
- Before asking for a reset, the game writes a **magic value** into RAM at `0xFF0800`.
- On boot, the startup code checks that address:
  - if it sees the magic value, it jumps directly into the final section of the game
  - if not, it performs a normal startup
- A quick tap on RESET keeps RAM intact and lets the game detect the special state.  
  Holding RESET too long or power‑cycling can destroy the magic value, making the game behave like a full cold boot.
- On hardware like the **Nomad**, where you do not have a RESET button, the puzzle becomes effectively unsolvable without external help.

This repo is just a small note to document this fun piece of console history and to show how a clever mix of hardware behaviour and software design can turn a physical reset switch into part of the game’s narrative and puzzle design.
