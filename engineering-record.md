---
title: "Fluid Fortune Engineering Record"
subtitle: "Problems Encountered. Solutions Discovered. Lessons Documented."
version: "v1.0 — April 2026"
author: "Eric Becker // FluidFortune.com"
---

> **This document is a confidential engineering reference.** It is available to the DEF CON 34 CFP Review Board on request (Submission ID: 1349) and to enterprise licensing inquiries. The six canonical engineering problems documented in Section 1 are also covered in the public Sovereignty White Paper at https://fluidfortune.com/sovereignty.html

---

**FLUID FORTUNE**

Engineering Record

*Problems Encountered. Solutions Discovered. Lessons Documented.*

Version 1.0 --- April 2026

Eric Becker // FluidFortune.com

**Introduction**

This document is a record of problems encountered, diagnosed, and solved
across the Fluid Fortune project stack. It is written for two audiences
simultaneously: the technical reader who wants to understand the
specific engineering decisions and why they were made, and the
non-technical reader who wants to understand what was actually
accomplished and why it is significant.

Where a technical concept requires deep domain knowledge to understand,
a plain English sidebar appears immediately following. These sidebars
are not simplifications --- they are translations. The technical
explanation is accurate. The sidebar makes it accessible.

The projects covered in this document, in the order they are addressed:

-   Pisces Moon OS --- custom general-purpose operating system for the
    LilyGO T-Deck Plus (ESP32-S3)

-   Pisces Moon Linux --- the companion tablet platform running Debian
    13 on a Fujitsu Q508

-   The Phantom --- local AI agent framework running on home hardware

-   Spadra Threat Intelligence System --- distributed network security
    monitoring

-   WozBot --- lightweight local AI proof of concept

-   Fluid Fortune Publishing Infrastructure --- Punky, Static, and the
    zero-server publishing stack

-   VaporwareOS --- embedded satirical platform and technical critique

Each section follows the same structure: what the project is, what
problems it encountered, what solutions were developed, and what those
solutions proved about the technology and the philosophy behind it.

> *The problems are not incidental to the story. They are the story.
> Every solution documented here is a discovery --- something that did
> not exist in public documentation before this project encountered the
> need and worked out the answer.*

**1. Pisces Moon OS**

**The First Known Documented Dual-Core Persistent Background Tasking OS for Field Intelligence on the ESP32-S3**

The LilyGO T-Deck Plus is a $50 handheld device with a color screen, a
physical QWERTY keyboard, a touchscreen, a trackball, and five wireless
radios: WiFi, Bluetooth Low Energy, LoRa long-range radio, and GPS. It
runs on the ESP32-S3 microcontroller at 240 megahertz across two
processor cores simultaneously.

Before Pisces Moon OS, every piece of software ever written for this
hardware did one thing. A wardriving tool. A mesh radio platform. A
game. When you turned on the device, it did its one function and nothing
else.

Pisces Moon OS is a true general-purpose operating system for this
hardware --- one where you launch different apps, switch between them,
and the device's identity is not defined by what is currently running.
As of v1.0.0, it ships 47 applications across 7 categories. It was the
first known documented dual-core persistent background tasking OS for field intelligence on this hardware class.

+-----------------------------------------------------------------------+
| **📘 Plain English: What makes this unusual**                         |
|                                                                       |
| Think of the difference between a microwave oven and a smartphone. A  |
| microwave has one job. A smartphone can run any app. Before Pisces    |
| Moon OS, every ESP32-S3 device was essentially a microwave. This      |
| project turned one into a smartphone --- on $50 of hardware, without |
| any of the infrastructure that smartphones rely on. That had never    |
| been done on this class of chip before.                               |
+-----------------------------------------------------------------------+

**1.1 The Hardware**

  -----------------------------------------------------------------------
  **Component**               **Specification**
  --------------------------- -------------------------------------------
  **Processor**               ESP32-S3 dual-core Xtensa LX7 @ 240MHz

  **Internal RAM**            320KB SRAM

  **External RAM**            8MB OPI PSRAM

  **Storage**                 16MB flash + MicroSD (up to 32GB)

  **Display**                 320×240 IPS LCD (ST7789)

  **Input**                   QWERTY keyboard, capacitive touchscreen,
                              trackball

  **Radios**                  WiFi 802.11 b/g/n, BLE 5.0, LoRa SX1262
                              (150-960MHz)

  **Location**                GPS receiver (L76K/UBlox, auto-baud
                              detection)

  **Audio**                   ES7210 I2S microphone + speaker

  **Power**                   AXP2101 PMU, LiPo battery

  **Cost**                    $50 retail
  -----------------------------------------------------------------------

**1.3 What Actually Happens When You Boot**

The boot sequence is the best single illustration of how all the
architectural decisions fit together. Here is exactly what happens, in
order, from the moment power is applied.

**Core 1 --- The Main OS (immediately on power-on)**

**GPIO power pins high.** The AXP2101 power management chip activates.
Peripheral power rails enable. Display, touch controller, and GPS power
path wake up.

**Backlight held OFF.** The display is powered but the backlight stays
dark.

+-----------------------------------------------------------------------+
| **📘 Plain English: Why the backlight waits**                         |
|                                                                       |
| If you turn the backlight on before the display's video memory is    |
| initialized, the screen shows white noise --- a flash of garbage      |
| pixels --- before the first real frame renders. Holding the backlight |
| off means the first thing the user ever sees is the correct image. A  |
| small detail that separates "real device" from "prototype."       |
+-----------------------------------------------------------------------+

**SPI bus initialized, display filled black twice, backlight ON.** Two
black fills clear any residual garbage pixels. Screen goes live.

**BIOS screen begins scrolling.** Green text on black. Each hardware
subsystem initializes and prints a status line. The aesthetic evokes
DOS-era boot screens deliberately --- communicating that this is a real
OS with a real hardware layer underneath it.

**Ghost Partition checks boot keys.** Before any other system reads GPIO
state, the Ghost Partition system reads the trackball and keyboard for a
combination that signals Ghost Partition mode. Must happen before
anything else touches those pins.

**PIN authentication screen appears.** Three possible PINs, three
possible outcomes: Tactical Mode (full OS, Ghost Partition mounted),
Student Mode (safe apps only, Ghost Partition invisible), Nuke (data
destruction in milliseconds). Normal Mode if no Ghost Partition
configured.

**WiFi auto-connect from /wifi.cfg.** Five second timeout. Critically:
the BIOS screen line renders completely BEFORE this call is made.

+-----------------------------------------------------------------------+
| **📘 Plain English: The WiFi cursor bug**                             |
|                                                                       |
| The WiFi SDK internally uses the same SPI hardware as the display.    |
| When WiFi initializes, it changes SPI state, which corrupts where the |
| display thinks it is currently drawing. If you start a line of text,  |
| call WiFi init, then finish the line, the text lands in the wrong     |
| place. Fix: always finish the complete display line first, then call  |
| WiFi. Simple once you know the cause. Invisible and maddening before  |
| you do.                                                               |
+-----------------------------------------------------------------------+

**SPI mutex created.** The FreeRTOS mutex enforcing the SPI Bus Treaty
is created here, before the wardrive task spawns. Both cores must have a
valid handle before either touches shared hardware.

**wardrive_core initialized. Ghost Engine spawned on Core 0.** The
wardrive task is pinned to Core 0 and created --- but it waits 15
seconds before doing anything.

+-----------------------------------------------------------------------+
| **📘 Plain English: Why Core 0 waits 15 seconds**                     |
|                                                                       |
| Core 1 is still finishing boot when Core 0 starts. If Core 0          |
| immediately touches the SD card and WiFi radio, it collides with Core |
| 1's initialization before the Treaty system is fully in place. The   |
| 15-second delay is a deliberate safety margin: by then, Core 1 has    |
| finished and the mutex is operational.                                |
+-----------------------------------------------------------------------+

**Gamepad driver initialized.** After wardrive_core, because the gamepad
uses the same NimBLE stack. wardrive.cpp must own NimBLE before the
gamepad attaches to it.

**Rainbow splash screen.** Brief animated signature of a successful
boot.

**run_launcher().** Category grid draws. Main event loop begins.
Everything from here is user-driven.

**Core 0 --- Ghost Engine (starts 15 seconds after Core 1)**

**GPS power pin high, 500ms stabilization wait.** GPS module requires
power stabilization before responding to serial commands.

**Serial initialized with 512-byte buffer.** Default Arduino serial
buffer is 128 bytes. GPS outputs continuous NMEA sentences. During the
2-4 second WiFi scan windows that block the CPU, 128 bytes overflows and
fix data is lost. 512 bytes survives scan windows without losing
sentences.

**Auto-baud GPS detection.** Tries 38400 baud first. If no valid NMEA
within timeout, switches to 9600. Two hardware variants of the T-Deck
Plus use different GPS modules at different speeds. Detection is
automatic.

**NimBLE initialized, scan object created.** Core 0 takes permanent
ownership of NimBLE for the session. No other component may
re-initialize it.

**Main wardrive loop begins.** Alternating 4-second WiFi scan windows
and 2-second BLE scan windows, GPS fed continuously, results written to
session-numbered CSV files via the SPI mutex. Runs indefinitely,
regardless of what Core 1 is doing.

+-----------------------------------------------------------------------+
| **📘 Plain English: What is actually happening while you use the      |
| device**                                                              |
|                                                                       |
| The T-Deck has two independent processors running simultaneously. One |
| runs whatever you are looking at on screen. The other one ---         |
| invisibly, silently, always --- is scanning for WiFi networks,        |
| logging Bluetooth devices, and recording GPS coordinates. Playing     |
| chess, reading a document, doing nothing. The Ghost Engine is always  |
| running. The device is always watching.                               |
+-----------------------------------------------------------------------+

From power-on to PIN screen: approximately 3-4 seconds. From PIN entry
to launcher: under one second. By the time the user has opened their
first app, the Ghost Engine has already logged every wireless device in
range.

**1.4 Problem: The Shared Bus Conflict**

The T-Deck Plus routes three major hardware components through a single
shared SPI bus --- the same physical wires:

-   The MicroSD card (storage)

-   The SX1262 LoRa radio (long-range communications)

-   The ST7789 display (screen)

In a single-function firmware, this conflict is invisible. If the
firmware only uses the SD card, the LoRa radio is idle and there is
nothing to collide. But a general-purpose OS with multiple simultaneous
operations --- wardrive background task writing to the SD card, LoRa
radio receiving messages, screen refreshing --- means all three are
competing for the same wires at the same time.

The result was a Guru Meditation error --- the ESP32's equivalent of a
fatal crash --- with no warning and no recovery. The device rebooted.
Whatever was on screen was lost. This happened repeatedly during early
development.

+-----------------------------------------------------------------------+
| **📘 Plain English: The shared bus problem**                          |
|                                                                       |
| Imagine four people trying to use the same telephone line             |
| simultaneously. When only one person is on the call, everything       |
| works. When two people try to talk at once, both conversations are    |
| destroyed. The SPI bus is that telephone line. The SD card, LoRa      |
| radio, and display are the people. Without rules about who can talk   |
| and when, everything crashes. Pisces Moon OS needed to invent those   |
| rules --- they did not exist for this hardware configuration anywhere |
| in public documentation.                                              |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **⚠ THE PROBLEM: SPI Bus Contention**                                 |
|                                                                       |
| Multiple hardware components sharing one SPI bus. Any two components  |
| attempting simultaneous access corrupts both transactions and crashes |
| the device. No documented solution existed for this specific hardware |
| configuration running a general-purpose OS.                           |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: The SPI Bus Treaty**                                |
|                                                                       |
| A formal architectural agreement governing every component of the OS. |
| Four rules: (1) Hit and run --- open a file, write, close, release    |
| immediately. Never hold the bus. (2) No extended holds --- no         |
| encryption during writes, no formatting during operation, nothing     |
| that monopolizes the bus. (3) Radio traffic management --- a shared   |
| flag (wifi_in_use) tells the wardrive task when WiFi is in use for an |
| API call, so it pauses its scan window. (4) Metadata-only destructive |
| functions --- the security Nuke sequence deletes index files          |
| (milliseconds) rather than formatting the card (seconds). A FreeRTOS  |
| mutex (spi_mutex) enforces the Treaty in code: every component        |
| touching shared hardware must acquire the mutex first and release it  |
| immediately after.                                                    |
+-----------------------------------------------------------------------+

The SPI Bus Treaty is documented, named, and publicly available. It is
the first such architectural standard for this hardware platform. Any
developer who encounters this problem in the future now has a documented
reference.

The Treaty was discovered empirically --- through crashes, diagnosis,
and engineering from first principles --- not found in existing
documentation. The historical precedent for this class of problem was
identified and is instructive: the IBM PC IRQ conflicts of the 1980s,
Unix mutex conventions from the 1970s, the Apollo Guidance Computer's
1202 alarm in 1969, and the Nintendo 64's RSP time budget from 1996 all
represent the same fundamental engineering problem: multiple subsystems
competing for a shared resource with no automatic arbitration.

**1.4 Problem: Memory Exhaustion Under Simultaneous Workloads**

The ESP32-S3 has 320 kilobytes of fast internal SRAM. For a
single-function firmware, this is generous. For a general-purpose OS
running a wardriving engine, an AI client, game logic, GPS logging, BLE
scanning, and display rendering simultaneously, it is not.

The device has 8 megabytes of additional PSRAM --- external RAM
connected over a high-speed bus. This sounds like the solution. The
problem: the default Arduino/ESP-IDF configuration routes all heap
allocations to internal SRAM. PSRAM exists on the hardware but is not
automatically used.

+-----------------------------------------------------------------------+
| **📘 Plain English: The RAM problem**                                 |
|                                                                       |
| Imagine you have a small desk (320KB SRAM) and a large filing cabinet |
| in the corner (8MB PSRAM). Every time you need to work on something,  |
| your instinct is to put it on the desk --- but the desk fills up fast |
| when you're working on seven things at once. The filing cabinet is   |
| sitting there empty. The fix: tell the computer to automatically put  |
| large items in the filing cabinet instead of cramming them onto the   |
| desk.                                                                 |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **⚠ THE PROBLEM: SRAM Exhaustion**                                    |
|                                                                       |
| All heap allocations defaulted to 320KB internal SRAM. Running        |
| multiple simultaneous applications, radio stacks, and background      |
| tasks exhausted available SRAM, causing allocation failures and       |
| crashes. The 8MB PSRAM chip was present but unused.                   |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: PSRAM Heap Redirect**                               |
|                                                                       |
| A single build flag --- CONFIG_SPIRAM_USE_MALLOC=1 --- instructs the  |
| memory management system to automatically route heap allocations      |
| above a threshold to PSRAM. Applications and libraries require no     |
| changes. Internal SRAM is now reserved for small, fast, time-critical |
| buffers. Everything else goes to PSRAM. This was not a documented     |
| solution for an ESP32-S3 OS context --- no previous firmware was      |
| complex enough to require it.                                         |
+-----------------------------------------------------------------------+

**1.5 Problem: Dual-Core Task Synchronization**

The ESP32-S3 has two independent processor cores. Most ESP32 projects
use only one and leave the other idle. Pisces Moon OS explicitly divides
them: Core 1 runs the user interface and all applications; Core 0 runs a
continuous background wardriving task called the Ghost Engine.

The Ghost Engine scans for WiFi networks, logs BLE beacons, reads GPS
coordinates, and writes structured records to the MicroSD card --- all
silently, all the time, regardless of what the user is doing on Core 1.
Playing a game, using the AI terminal, browsing files: the Ghost Engine
keeps collecting.

The challenge: both cores share hardware resources. Without explicit
coordination, Core 0 writing to the SD card at the same moment Core 1
reads from it produces corrupted data and crashes.

+-----------------------------------------------------------------------+
| **📘 Plain English: Two cores, one set of hardware**                  |
|                                                                       |
| Think of two workers sharing one set of tools. If both reach for the  |
| same tool at the same time, you get a conflict. The solution is a     |
| sign-out system: before using a shared tool, you check it out. When   |
| done, you check it back in. Anyone else who needs it waits. Pisces    |
| Moon OS implements this as a software "mutex" --- a lock that       |
| ensures only one core touches shared hardware at a time.              |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: The Dual-Core Architecture with SPI Mutex**         |
|                                                                       |
| The wardrive task is pinned to Core 0 via xTaskCreatePinnedToCore()   |
| with a deliberate 15-second startup delay, allowing Core 1 to         |
| complete boot before Core 0 begins touching shared hardware. The SPI  |
| mutex (part of the Bus Treaty) coordinates SD card access between     |
| cores. A parallel wifi_in_use flag coordinates WiFi radio access.     |
| Neither core needs to know what the other is doing --- coordination   |
| happens through shared state flags, not direct coupling.              |
+-----------------------------------------------------------------------+

The result: Pisces Moon OS actively wardrives --- scanning WiFi
networks, logging BLE beacons, writing GPS-tagged records --- while the
user plays a game, uses the AI terminal, or does anything else. True
parallel execution on separate silicon. This is believed to be a novel
implementation of continuous background wardriving on this hardware
class.

**1.6 Problem: Dense RF Environment Instability**

In a lab environment with one or two WiFi networks present, the
wardriving system worked correctly. In downtown Los Angeles with 40+
access points and hundreds of Bluetooth devices, the system crashed.

The root cause: BLE advertisement callbacks run in interrupt service
routine context, which has a fixed, small stack. In a dense RF
environment, callbacks fire faster than the system can process them.
Stack overflow. Crash.

+-----------------------------------------------------------------------+
| **📘 Plain English: The dense city problem**                          |
|                                                                       |
| Imagine you're taking notes every time someone walks past you. In a  |
| quiet hallway, this is easy. In a crowded train station, people are   |
| passing faster than you can write. You run out of notepad space and   |
| drop everything. The solution: instead of writing everything down     |
| immediately, hand notes to a queue and process them at a manageable   |
| pace.                                                                 |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: ISR-Safe BLE Queue**                                |
|                                                                       |
| BLE advertisement callbacks now do the minimum possible work: they    |
| push a small record into a lock-free circular queue and return        |
| immediately. A separate task on Core 0 drains the queue at a          |
| controlled pace. The ISR never overflows because it does almost       |
| nothing. The queue absorbs bursts. Validated in real-world testing:   |
| 40+ WiFi access points and 100+ BLE devices, no crash.                |
+-----------------------------------------------------------------------+

**1.7 Problem: GPS Module Hardware Variation**

The T-Deck Plus was manufactured with two different GPS modules across
production batches: one operating at 9600 baud, one at 38400 baud.
Software written for one variant fails silently on the other. There is
no marking on the hardware to distinguish them. The official
documentation does not mention the variation.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Auto-Baud GPS Detection**                           |
|                                                                       |
| The GPS initialization routine tries 9600 baud first. If no valid     |
| NMEA sentences are received within a timeout, it switches to 38400.   |
| The RX buffer was also increased from the default 128 bytes to 512    |
| bytes --- the default overflows during the 2-4 second WiFi scan       |
| windows that block the CPU, causing GPS fix acquisition to stall.     |
+-----------------------------------------------------------------------+

**1.8 Problem: I2S DMA SRAM Ceiling**

Adding audio recording (v1.0.0) revealed a new constraint class. The I2S
audio driver --- the system that manages microphone and speaker hardware
--- always allocates its DMA buffers from internal SRAM, regardless of
the PSRAM heap redirect. This is a driver-level behavior that cannot be
overridden by configuration.

With WiFi, BLE, and the wardrive task resident in memory, launching the
audio recorder with standard DMA settings (8 buffers × 1024 bytes =
16KB) failed with a Guru Meditation before a single pixel of UI was
drawn.

+-----------------------------------------------------------------------+
| **📘 Plain English: The audio memory problem**                        |
|                                                                       |
| The PSRAM heap redirect from Problem 1.3 was a great solution --- but |
| it only works for some kinds of memory. The audio system uses a       |
| different kind of memory allocation that bypasses the redirect        |
| entirely and always takes from the small desk (SRAM). With the desk   |
| already crowded, there was no room for audio. The fix: make the audio |
| system ask for less desk space.                                       |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: I2S DMA Budget Rule**                               |
|                                                                       |
| DMA buffers capped at 4 buffers × 512 bytes = 4KB (down from 16KB).   |
| Read buffers moved to ps_malloc() --- PSRAM-allocated on launch,      |
| freed on exit. Warmup buffers reduced from 8KB stack allocations to   |
| 512-byte stack arrays. This constraint is now documented as a hard    |
| rule for all future I2S users in the OS.                              |
+-----------------------------------------------------------------------+

**1.9 Problem: Static Global BSS Overflow**

Adding eight new CYBER applications in a single development session
caused the linker to report: .dram0.bss overflow by 70,120 bytes.

The root cause: large data structures declared as static global arrays.
BSS is the segment of internal SRAM where all static globals live ---
permanently, at boot, regardless of whether the app that owns them is
running. The PSRAM redirect only affects heap allocations. Static
globals always go to SRAM. Eight new apps each bringing large static
arrays loaded the entire set into SRAM simultaneously.

+-----------------------------------------------------------------------+
| **📘 Plain English: The static global problem**                       |
|                                                                       |
| Imagine every app leaving its furniture in the hallway even when the  |
| app is closed. The hallway fills up. The PSRAM redirect was supposed  |
| to send large items to the storage room --- but it only works for     |
| items you specifically request storage for. If you declare something  |
| as always-available-in-the-hallway, it stays in the hallway. The fix: |
| stop leaving furniture in the hallway. Bring it out when the app      |
| opens, put it away when it closes.                                    |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **⚠ THE PROBLEM: BSS Overflow --- 70,120 Bytes**                      |
|                                                                       |
| Static global arrays across 8 new apps loaded permanently into        |
| internal SRAM at boot: RF spectrum waterfall buffer (105KB), probe    |
| device arrays (27KB), packet analysis structures (9KB), and others.   |
| Total BSS pressure exceeded available SRAM by 70KB.                   |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: PSRAM Dynamic Allocation Pattern**                  |
|                                                                       |
| All large data structures converted from static globals to            |
| ps_malloc() allocated on app launch and freed on exit. The RF         |
| waterfall buffer (105KB) alone moved to PSRAM. Total moved from BSS   |
| to PSRAM: \~137KB, resolving the 70KB deficit with margin. This       |
| pattern is now documented as the required approach for any large      |
| buffer in the OS --- no large arrays in global scope.                 |
+-----------------------------------------------------------------------+

**1.10 Problem: NimBLE Singleton Conflict**

As more applications required Bluetooth access --- the GATT explorer,
the BLE keyboard injector, the gamepad driver --- each attempted to
initialize the NimBLE Bluetooth stack. NimBLE is a singleton: there is
exactly one instance, and it cannot be re-initialized. Multiple apps
competing for it caused crashes and non-functional BLE behavior.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: NimBLE Ownership Protocol**                         |
|                                                                       |
| wardrive.cpp owns NimBLE permanently for the lifetime of the session. |
| Any app requiring BLE access calls wardrive_ble_stop() before using   |
| the scan object --- which halts the active scan and clears callbacks  |
| cleanly --- and wardrive_ble_resume() when finished. The app in the   |
| middle has exclusive, clean access. No app may call                   |
| NimBLEDevice::init() or deinit() under any circumstances.             |
+-----------------------------------------------------------------------+

**1.11 Problem: USB CDC vs HID Exclusivity**

The USB Ducky application --- which makes the T-Deck appear as a USB
keyboard to inject keystrokes into a connected computer --- required USB
HID mode. But the standard build uses USB CDC mode for serial
communications and PlatformIO flashing. The ESP32-S3's USB peripheral
can be one or the other, not both simultaneously.

+-----------------------------------------------------------------------+
| **📘 Plain English: The USB identity problem**                        |
|                                                                       |
| A USB device tells the computer what it is when it plugs in. It can   |
| say "I'm a serial port" or "I'm a keyboard" --- but not both at |
| once. The T-Deck needs to be a serial port for development work. For  |
| keyboard injection testing, it needs to be a keyboard. These are      |
| physically the same socket but logically incompatible identities.     |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Dual Build System**                                 |
|                                                                       |
| Two separate build environments in platformio.ini. The standard build |
| (esp32s3) uses CDC serial mode --- full development capability,       |
| PlatformIO flashing, stable GPIO1. The HID build (esp32s3_hid) uses   |
| USB HID mode --- keyboard injection, no serial console. The USB Ducky |
| app detects which build is running and displays appropriate UI for    |
| each case. BLE Ducky provides keyboard injection in the standard      |
| build without any reflashing required.                                |
+-----------------------------------------------------------------------+

**1.12 Problem: Trackball Lockout Mismatch**

The trackball uses a 250ms debounce lockout --- the minimum time between
registered direction inputs. This is correct for UI navigation, where
fast repeated inputs would cause accidental menu traversal. But Pac-Man
runs at 30 frames per second, polling the trackball every 33ms. At 250ms
lockout, 7-8 frames pass between valid direction inputs. Turns near
junctions were consistently missed.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Split Trackball API**                               |
|                                                                       |
| update_trackball() retains the 250ms lockout for all UI navigation    |
| --- unchanged. A new update_trackball_game() provides an 80ms lockout |
| for game loops. All existing UI code is unaffected. All game loops    |
| call the game variant. Future games must use update_trackball_game(). |
+-----------------------------------------------------------------------+

**1.13 The Ghost Partition Security System**

The Ghost Partition is a security architecture unique to Pisces Moon OS.
It addresses a specific and honestly-documented threat model: physical
inspection of the device by someone with authority but without
sophisticated forensic tools.

A MicroSD card is formatted with two partitions. Partition 1 (public)
contains ordinary data --- reference guides, games, notes. Partition 2
(Ghost) contains sensitive operational data --- wardrive logs, AI
session histories, security tool output. The Ghost Partition is mounted
only in Tactical Mode, accessible only through a three-layer
authentication system.

  -----------------------------------------------------------------------
  **Aspect**                  **Detail**
  --------------------------- -------------------------------------------
  **Authentication Layer 1**  PIN code --- three separate PINs with
                              different outcomes

  **Primary PIN outcome**     Prompts for hardware second factor,
                              proceeds to Tactical Mode

  **Decoy PIN outcome**       Loads Student Mode --- only safe apps
                              visible, Ghost Partition never mounted, no
                              indication it exists

  **Nuke PIN outcome**        Rapid data destruction --- index files
                              deleted in milliseconds, data unreachable
                              through OS

  **Authentication Layer 2**  Hardware MFA --- a specific key must be
                              pressed after the correct PIN, with no
                              label or indication of which key

  **Wrong key after correct   Silently loads Student Mode --- no error,
  PIN**                       no indication of failure

  **Partition visibility**    Windows shows only Partition 1 (FAT32
                              convention). macOS/Linux: MBR type byte
                              changed from 0x0B (FAT32) to 0xDA (unknown)
                              --- partition displays as unreadable

  **Security posture**        Appropriate for the threat model:
                              inspection by non-forensic adversary. Not
                              designed for nation-state chip-level
                              analysis.
  -----------------------------------------------------------------------

+-----------------------------------------------------------------------+
| **📘 Plain English: What the Ghost Partition does**                   |
|                                                                       |
| Imagine a briefcase with a hidden compartment. The visible section    |
| has innocent contents --- maps, a calculator, some notes. The hidden  |
| section requires two separate keys to open. If you enter the wrong    |
| combination, the briefcase shows you the innocent section anyway,     |
| with no indication that a hidden section exists. If you need to       |
| destroy the hidden contents quickly, one code wipes the filing system |
| --- leaving the data physically present but permanently unreachable   |
| through normal means. This is the Ghost Partition.                    |
+-----------------------------------------------------------------------+

The security architecture is documented with explicit, honest limitation
statements. It provides meaningful protection against casual inspection.
It does not provide protection against dedicated forensic analysis of
the flash memory at the chip level. This distinction is stated directly
in the technical documentation.

**2. Pisces Moon Linux**

**The Tablet Platform --- Debian 13 on the Fujitsu Q508**

Pisces Moon Linux is the second tier of the three-tier ecosystem. Where
the T-Deck Plus collects data in the field, the Fujitsu Stylistic Q508
tablet --- a 10-inch Windows tablet repurposed to run Debian 13 Linux
--- receives, stores, analyzes, and visualizes that data.

The Q508 runs an Intel Atom X5 processor with 4GB RAM. It has a 1280×800
capacitive touchscreen. It is modest hardware by current desktop
standards, but compared to the T-Deck it is a supercomputer: capable of
running full Linux applications, local AI inference, Wireshark-level
packet analysis, and any software available in the Debian ecosystem.

  -----------------------------------------------------------------------
  **Component**               **Specification**
  --------------------------- -------------------------------------------
  **Processor**               Intel Atom X5 (quad-core, 2.4GHz)

  **RAM**                     4GB DDR3

  **Storage**                 64GB eMMC + MicroSD slot

  **Display**                 10" 1280×800 capacitive touchscreen

  **OS**                      Debian 13 (Trixie) + XFCE minimal

  **App platform**            29 HTML apps via Chromium \--app=
                              (proto-Trojan Horse)

  **AI integration**          Gemini API + Phantom API over Tailscale

  **Cost**                    $150-200 used
  -----------------------------------------------------------------------

**2.1 Problem: Installing on Contaminated Hardware**

The initial installation attempt ran on a system with leftover
configuration from a previous install attempt --- an old launcher
process autostarting, OpenBox session fragments competing with XFCE,
mystery packages from an earlier approach. The resulting system was
difficult to debug because the problems could originate anywhere in the
layered history.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Fresh Minimal Install Protocol**                    |
|                                                                       |
| Decision: wipe and reinstall from scratch with Debian 13 minimal XFCE |
| --- only XFCE checked during installation, no LibreOffice, no extras, |
| nothing. A minimal install has no mystery leftovers. If something     |
| breaks, it is attributable to the current install script, not to      |
| historical contamination. This is now the documented baseline for the |
| Pisces Moon Linux install.                                            |
+-----------------------------------------------------------------------+

**2.2 Problem: Q508 Detection Over SSH**

The install script detected the Q508's 1280×800 screen to determine
whether to apply touch configuration. This detection used xrandr --- a
display management tool that requires an active X display session. When
running the installer via SSH (the natural approach, since it allows a
real keyboard from another machine), no DISPLAY is set, xrandr fails
silently, and the Q508 is not detected.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Direct Question Over Silent Detection**             |
|                                                                       |
| When no $DISPLAY is set, the installer asks directly: "Is this a    |
| Fujitsu Q508?" The user types yes or no. No clever detection         |
| required. Simple input beats clever inference when the clever         |
| inference fails on the most common use case.                          |
+-----------------------------------------------------------------------+

**2.3 Problem: XFCE Menu Not Showing Pisces Moon Apps**

After the initial install ran successfully --- 37 desktop entries
created, apps deployed, edge bridge installed --- the Pisces Moon
category did not appear in the XFCE right-click menu. The apps existed
on disk. The .desktop files existed in /usr/share/applications. The menu
simply did not show them.

Root cause: the custom menu file was placed in /etc/xdg/menus/ rather
than /etc/xdg/menus/applications-merged/, which is the directory XFCE
actually reads for custom menu additions.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Correct Menu Registration Path**                    |
|                                                                       |
| Menu file moved to                                                    |
| /etc/xdg/menus/applications-merged/pisces-moon.menu.                  |
| update-desktop-database run after installation. App launch method     |
| changed from xdg-open (which requires a browser to be configured) to  |
| chromium \--app= (which opens each HTML app as a standalone           |
| borderless window --- a functional proto-Trojan Horse wrapper).       |
+-----------------------------------------------------------------------+

**2.4 The Proto-Trojan Horse Discovery**

The chromium \--app= flag, used as the Exec= command in XFCE .desktop
entries, strips all browser chrome --- no tabs, no address bar, no menu
bar --- and presents the HTML application as a standalone window. Each
app opens as its own window. It looks and feels native.

This was not the intended final architecture. It was a pragmatic
deployment decision. But it turned out to be a working preview of Trojan
Horse --- the forthcoming WebKitGTK wrapper that will provide full
native filesystem access to HTML apps.

> *The \--app= flag was a placeholder that accidentally became a
> demonstration of the final architecture. When Trojan Horse ships,
> swapping chromium \--app= for the WebKitGTK host in the .desktop files
> upgrades every app to full native capability without changing a single
> line of HTML.*

**3. The Phantom**

**Local AI Agent Framework --- The Body the Brain Never Had**

Local AI language models --- DeepSeek, Gemma, Llama, Qwen --- are
extraordinary reasoning engines. They are frozen in time: trained on
data with a cutoff date, unable to access the internet, unable to save
files, unable to remember yesterday's conversation. They are, in the
Phantom's own framing, a genius locked in a room with no windows and no
doors.

The Phantom is the infrastructure that opens those windows. It is a
Python wrapper around Ollama (the local model server) that intercepts
every conversation turn, performs whatever research or data retrieval is
needed before the model sees the prompt, injects the results as context,
and handles all post-processing after the model responds.

+-----------------------------------------------------------------------+
| **📘 Plain English: What The Phantom does**                           |
|                                                                       |
| When you ask a local AI "what was the Dodgers score last night?",   |
| the model has no idea --- it was trained months ago and cannot access |
| the internet. The Phantom solves this by checking the score BEFORE    |
| asking the model. It finds the answer, writes it into the question,   |
| and hands the model a message that says "given that the Dodgers won  |
| 4-2, tell me about the game." The model looks like it knows. The     |
| Phantom actually knew.                                                |
+-----------------------------------------------------------------------+

**3.1 Problem: The Memory Problem**

A local AI model wakes up with amnesia every session. It does not know
your name, your preferences, what you worked on last week, or that you
have already explained three times how you want your code structured.
Cloud AI products simulate continuity through user profiles and
conversation summaries maintained on corporate servers. Local AI has
none of this by default.

+-----------------------------------------------------------------------+
| **⚠ THE PROBLEM: Local AI Has No Memory**                             |
|                                                                       |
| Each conversation session starts completely fresh. Context            |
| established in previous sessions is lost. The model cannot recall     |
| corrections, preferences, or ongoing projects. This makes local AI    |
| feel significantly less capable than cloud alternatives, regardless   |
| of actual model quality.                                              |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: ChromaDB Vector Memory**                            |
|                                                                       |
| ChromaDB runs entirely locally --- no cloud, no API key. After every  |
| conversation turn, each message is converted to a mathematical vector |
| representing its meaning and stored. Before every turn, the system    |
| retrieves the five most semantically similar past exchanges --- not   |
| by keyword search but by meaning. "How has Miguel Rojas been         |
| performing?" retrieves conversations about Dodgers players even if   |
| they used different words. Combined with automatic conversation       |
| distillation (after 60 turns, older exchanges are summarized into a   |
| compact memory snapshot), The Phantom maintains genuine continuity    |
| across sessions.                                                      |
+-----------------------------------------------------------------------+

+-----------------------------------------------------------------------+
| **📘 Plain English: How vector memory works**                         |
|                                                                       |
| Keyword search asks "does this document contain the word             |
| 'baseball'?" Vector search asks "is this document about the same  |
| subject as the question I'm asking?" It finds related content even  |
| when the words are completely different. This is how human memory     |
| actually works --- you remember things by association, not by exact   |
| word match. ChromaDB gives a local AI something close to that.        |
+-----------------------------------------------------------------------+

**3.2 Problem: The Frozen Brain Problem**

Local models cannot access the internet. When asked about current
events, recent sports scores, news, or anything that changed after their
training cutoff, they either confabulate (make up plausible-sounding but
wrong answers) or admit they do not know. This limits their practical
utility for daily use significantly.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Two-Stage Web Intelligence**                        |
|                                                                       |
| Before every conversation turn, two detection stages run. First: a    |
| keyword check for trigger phrases ("search", "score", "who is", |
| "latest"). If keywords are present, web search runs. Second: if     |
| keywords miss, the model itself is asked in a single 5-token query    |
| whether it needs web access for this question. This catches queries   |
| that keyword matching would miss --- "How has Miguel Rojas performed |
| this season?" contains no trigger keyword but the model correctly    |
| identifies that it needs current data. Web results are scraped in     |
| full (not just snippet summaries), with a paywall bypass cascade      |
| using 12ft.io.                                                        |
+-----------------------------------------------------------------------+

**3.3 Problem: Sports Data Accuracy**

Early testing revealed that asking local models about baseball
statistics produced a specific failure mode: confusing at-bats with
plate appearances. These are different statistics. At-bats exclude
walks, hit-by-pitches, and sacrifice flies; plate appearances include
them. The distinction matters for batting average calculation. No
general-purpose web scraper would catch this.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: MLB Stats API Integration**                         |
|                                                                       |
| The Phantom integrates directly with the official MLB Stats API ---   |
| free, no key required --- for real-time scores, full season           |
| statistics, and player biographical data. Every player lookup         |
| auto-writes to phantom_databases/mlb_players.json in a format         |
| compatible with Pisces Moon apps on the tablet and T-Deck. The        |
| database export function minifies JSON for device transfer. The       |
| distinction between at-bats and plate appearances is correctly        |
| maintained.                                                           |
+-----------------------------------------------------------------------+

**3.4 The Phantom as Tier Zero**

When The Phantom was built, it was conceived as a standalone AI
assistant. Over time it became clear that it was actually the data
preparation layer the entire Pisces Moon ecosystem had always needed ---
Tier Zero feeding all other tiers.

  -----------------------------------------------------------------------
  **Tier**                    **Role**
  --------------------------- -------------------------------------------
  **Tier 0 --- The Phantom    Web intelligence, MLB API, vector memory,
  (Mac)**                     scout daemon writing fresh JSON to disk

  **Tier 1 --- T-Deck Plus**  Field collection: wardrive, BLE, GPS, LoRa,
                              Ghost Engine, Ghost Partition

  **Tier 2 --- Q508 Tablet**  29 HTML apps, edge bridge, Smelter
                              analysis, Tailscale mesh node

  **Tier 3 --- Gemini API**   Optional cloud intelligence when
                              connectivity and privacy requirements
                              permit

  **Mesh layer ---            Private WireGuard mesh connecting all
  Tailscale**                 nodes; Phantom API reachable from tablet at
                              Tailscale IP
  -----------------------------------------------------------------------

The baseball app on the tablet initially queried Gemini for scores and
received Gemini's training-data approximation of current events. After
Tailscale integration, the same app queries The Phantom's MLB API
endpoint and receives live, structured data --- no API key, no cloud
subscription, no data leaving the local network.

**4. Spadra Threat Intelligence System**

**Distributed Network Security Monitoring on Commodity Hardware**

The Spadra Threat Intelligence System is a four-tier distributed
security monitoring platform built on commodity hardware using
open-source tools. It does what enterprise Security Operations Centers
do --- centralized threat detection, cross-system correlation, automated
response --- at a cost of a few dollars a month in cloud credits.

Within twenty minutes of the honeypot going live on a public Google
Cloud VM, the first known threat actor knocked on port 80. Not
simulated. Not a test. A real IP address already flagged in community
threat databases, probing a machine it had never encountered before.

  -----------------------------------------------------------------------
  **Component**               **Description**
  --------------------------- -------------------------------------------
  **Tier 1 --- Threat Intel   REST API giving all components a shared
  Broker**                    language. The equivalent of MISP (Malware
                              Information Sharing Platform) built from
                              scratch in an afternoon.

  **Tier 2 --- Log            Pulls data from all nodes, merges into one
  Aggregator**                timeline, runs cross-system correlation. A
                              single sensor reports suspicious activity.
                              The Aggregator reports that two independent
                              sensors on different networks saw the same
                              IP.

  **Tier 3 --- Response       Three-gate automated block decision:
  Engine**                    minimum hit count + IPsum threat database
                              confirmation + cross-system correlation
                              match. All three required. Automated
                              response without conservative criteria is
                              more dangerous than no automation.

  **Tier 4 --- Dashboard**    Translation layer between the system and
                              humans who need to understand it. API
                              endpoints live; visualization in
                              development.

  **Hardware**                GCP Cloud VM (Debian 12) + Intel N150
                              Spadra Server (Ubuntu 24.04) + Raspberry Pi
                              400

  **Stack**                   Python, Flask, Scapy, Tailscale, tmux,
                              IPsum threat feed
  -----------------------------------------------------------------------

**4.1 Problem: The Cross-System Correlation Gap**

Any single sensor can tell you it saw something suspicious. What it
cannot tell you is whether that suspicious activity was also seen by a
completely separate sensor on a different network. This cross-reference
is qualitatively different information --- the difference between a
witness and a corroborated account.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: The Log Aggregator**                                |
|                                                                       |
| A single aggregation service pulls from all nodes on a schedule,      |
| merges events into one chronological timeline, and runs correlation   |
| checks automatically. The question "did the IP that hit our cloud VM |
| at 5:38 AM also appear in local network traffic?" was unanswerable   |
| before the Aggregator. Now it is answered automatically every 30      |
| minutes without human intervention.                                   |
+-----------------------------------------------------------------------+

**4.2 The Democratization Argument**

Enterprise security tools are financially inaccessible to individuals
and small organizations. Splunk Enterprise starts at tens of thousands
of dollars per year. Commercial honeypot platforms cost similarly. The
result: a security knowledge gap that maps almost exactly onto an
economic gap. Large organizations understand their threat environment.
Small organizations and individuals do not.

The Spadra system was built for essentially nothing --- open-source
tools, a few dollars a month in cloud credits --- and does the same
fundamental job as tools that cost hundreds of thousands of dollars
annually. The knowledge exists. The tools are freely available. The
barrier is understanding, not resources.

> *"The internet is not a safe place. It never was. The question has
> never been whether someone is trying to get in. The question is
> whether you are paying attention."*

**5. WozBot**

**The Demonstration Problem --- And Its Solution**

The Phantom is a sophisticated local AI agent framework. It is also hard
to explain to someone who has never run a local AI before. The full
description --- multi-project sandboxing, document libraries, file
generation, streaming chat, model switching, web scouting, podcast
generation --- produces glazed eyes in approximately thirty seconds.

WozBot is the explanation. It strips The Phantom to its essential proof:
one language model, one system prompt, one web interface, one server.
The character is Steve Wozniak's 1970s Dial-A-Joke phone line, rebuilt
for 2026 on a quantized language model.

+-----------------------------------------------------------------------+
| **📘 Plain English: Why WozBot exists**                               |
|                                                                       |
| Nobody shares a productivity assistant with their friends. But a      |
| boisterous AI that tells terrible 1970s hardware puns and recounts    |
| Apple Computer history from the inside? That gets posted on Reddit.   |
| When someone finds the GitHub repo, they find Fluid Fortune, and they |
| find The Phantom, and they think: hm. The joke bot is the Trojan      |
| Horse. The Phantom is what's inside.                                 |
+-----------------------------------------------------------------------+

**5.1 Problem: The Demonstration Gap**

Technical projects often fail not because the technology is inadequate
but because the demonstration is inadequate. A sophisticated framework
requires sophisticated context to appreciate. Most people who might
benefit from The Phantom lack that context.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: The Minimal Proof of Concept**                      |
|                                                                       |
| WozBot demonstrates the core architecture --- local model + system    |
| prompt + web interface + server --- in a form anyone can experience   |
| without technical knowledge. It is live at wozbot.fluidfortune.com,   |
| running on bare metal in the Fluid Fortune private perimeter,         |
| tunneled to the public internet via Cloudflare. No cloud compute. No  |
| corporate data center. No ongoing cost beyond electricity. The joke   |
| is the point. The architecture behind the joke is also the point.     |
+-----------------------------------------------------------------------+

**5.2 Technical Architecture**

  -----------------------------------------------------------------------
  **Component**               **Detail**
  --------------------------- -------------------------------------------
  **Model**                   Qwen2.5-1.5B-Instruct Q4_K_M ---
                              approximately 1GB on disk, \~2GB RAM

  **Inference**               llama.cpp server on port 8080, CPU-only

  **Server**                  Intel N150 MicroPC, 16GB DDR5 RAM, Ubuntu
                              24.04

  **Routing**                 Cloudflare Tunnel → Nginx on port 8181 →
                              llama.cpp on port 8080

  **Privacy**                 No data persists. No logs. Stateless.
                              Session exists only in browser memory.

  **Cold start**              Startup script fires warmup request after
                              launch --- first user never experiences
                              47-second cold start

  **Cost**                    $0/month (home server + Cloudflare free
                              tier)
  -----------------------------------------------------------------------

WozBot demonstrates that a publicly accessible AI service requires
neither AWS nor a venture capital term sheet. The entire infrastructure
fits in a MicroPC the size of a paperback book.

**6. Fluid Fortune Publishing Infrastructure**

**Punky, Static, and the End of WordPress**

Publishing a blog post in 2026 typically involves: choosing a CMS
platform, creating an account, configuring a hosting environment,
installing and updating plugins, paying a monthly subscription, and
remaining dependent on a company whose interests are not necessarily
aligned with yours.

Punky asks a simpler question: what does publishing a blog post actually
require? A text file. Somewhere to put it. A way to tell people it
exists. That is the complete list. Everything else is overhead someone
else built because overhead is how you charge a monthly fee.

+-----------------------------------------------------------------------+
| **📘 Plain English: The WordPress problem**                           |
|                                                                       |
| WordPress is bloat. Not a little bloat --- architectural bloat. A     |
| blog post is a text file. WordPress requires a database server, a PHP |
| runtime, a web server, a plugin ecosystem with its own security       |
| vulnerabilities, and a hosting provider. Punky replaces the entire    |
| stack with one HTML file and a GitHub account. The blog that results  |
| is faster, cheaper (free), more durable, and owned completely by the  |
| person who wrote the posts.                                           |
+-----------------------------------------------------------------------+

**6.1 Problem: Blog Publishing Requires a Server**

Every major blogging platform --- WordPress, Ghost, Substack ---
requires server infrastructure. Either you pay someone to run it for
you, or you run it yourself. Either way, there is a server, a database,
a monthly cost, and a dependency on a company that can change its terms.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Punky --- GitHub as the Server**                    |
|                                                                       |
| Punky is a single HTML file. Open it in Chrome. Write a post using a  |
| rich text editor. Click publish. Punky calls the GitHub API directly  |
| from your browser, creates the post file in your repository, updates  |
| the JSON manifest, regenerates the RSS feed and sitemap. Your GitHub  |
| token stays in your browser. Your posts go to your repository. The    |
| cost of running a blog on Punky: $0/month.                           |
+-----------------------------------------------------------------------+

**6.2 Problem: Podcast Publishing Requires a Hosting Platform**

Podcast platforms --- Anchor, Buzzsprout, Podbean --- host audio files,
generate RSS feeds, and distribute to Apple Podcasts and Spotify. They
also own the relationship with your audience, can change their terms,
and charge monthly fees that increase with audience size.

+-----------------------------------------------------------------------+
| **✓ THE SOLUTION: Static --- Archive.org as the Host**                |
|                                                                       |
| Static is Punky's sibling, built for audio. Audio files go to        |
| archive.org --- the Internet Archive, a non-profit digital library    |
| that has been preserving the web since 1996 and has no business model |
| that requires monetizing your audience. Episode pages and RSS feeds   |
| are generated by Static and served by GitHub Pages. The shows cannot  |
| be deplatformed by a hosting company changing its terms. Total cost:  |
| $0/month.                                                            |
+-----------------------------------------------------------------------+

**6.3 The Durability Argument**

A blog post published through Punky is a file. Files can be copied,
moved, archived, and read in any browser forever, without dependency on
any company's continued operation. A Substack post requires Substack to
exist, to keep your account active, and to not change the terms under
which your content is hosted.

The Fluid Fortune publishing infrastructure was designed with a specific
test: could we migrate off every dependency we use without losing our
work, our tools, or our audience? For GitHub: yes --- everything is
files, files can be moved. For archive.org: audio files are redundantly
preserved by one of the most committed archival organizations in
existence. For Chrome: any browser renders HTML.

This is the Clark Beddows Protocol applied to publishing: local first,
no gatekeepers, you own everything.

**7. VaporwareOS**

**Satire That Compiles --- A Technical Critique in Embedded Form**

VaporwareOS is eight applications for a $30 Waveshare ESP32-S3
development board, each satirizing a different failure mode of the
modern technology industry. The applications are polished, animated, and
technically rigorous. They are also, mostly, fake.

Cloud Sync Pro hangs at 99% forever while rotating corporate jargon. The
AI Tokenizer charges $0.14 per consultation and returns randomly
selected profound-sounding nonsense. Banana Quest simulates a future
where Apple sells neural mesh implants billed per pound of
telekinetically moved objects. RetroPad is a fully functional BLE
gamepad that works with RetroArch and Steam --- included to demonstrate
what "actually works" looks like among everything that does not.

+-----------------------------------------------------------------------+
| **📘 Plain English: Why build a satirical embedded OS**               |
|                                                                       |
| The most credible criticism of the technology industry comes from     |
| inside it. A developer who has shipped production code earns the      |
| right to criticize production code. VaporwareOS is built with         |
| professional-grade embedded engineering --- PSRAM framebuffers, BLE   |
| HID, passive RF monitoring --- so the critique cannot be dismissed as |
| the work of someone who does not understand the engineering. The joke |
| is built from the same materials as the thing it is joking about.     |
| That is what makes it land.                                           |
+-----------------------------------------------------------------------+

**7.1 Problem: The Screen Does Not Work**

VaporwareOS's display --- a Waveshare 2.41-inch AMOLED panel using the
RM690B0 controller over QSPI --- does not display anything. The serial
monitor reports successful initialization at every stage. The screen
remains dark.

The debugging process revealed that the board exists in at least two
hardware revisions with identical product names and different pin
assignments. The QSPI command format embeds MIPI DCS commands in the
middle byte of a 24-bit address field. Pixel write transactions require
different SPI flags than command write transactions. Large pixel
transfers must be chunked at 16,384 pixels per transaction using an
extended transaction structure. None of this is documented in a single
accessible location.

+-----------------------------------------------------------------------+
| **📘 Plain English: Why the irony is the point**                      |
|                                                                       |
| The firmware that mocks progress bars that lie has its own serial     |
| monitor reporting "init complete" while the display shows nothing.  |
| This is not a failure. It is the most honest thing VaporwareOS does.  |
| The court jester is lying on the floor of the throne room. The bit    |
| landed anyway. When the screen eventually works, the first image will |
| be VAPOR OS, then EVERYTHING, then a subtitle: "Alpha Version:       |
| Whatever Your Mind Believes It To Be." This is the most honest thing |
| the device will ever say.                                             |
+-----------------------------------------------------------------------+

**7.2 The Court Jester Framework**

The medieval court jester occupied a structurally unusual position: the
only person in the room permitted to tell the king he was wrong. The
costume --- the bells, the motley --- was not decoration. It was armor.
Absurdity was the license that made honesty survivable.

The technology industry has a sophisticated immune response to
criticism. Criticism from outside is dismissed as ignorance. Criticism
from inside is absorbed as self-deprecating charm and changes nothing.
VaporwareOS sidesteps both failure modes by being technically rigorous
(it cannot be dismissed as ignorant) and performed as entertainment (it
cannot be absorbed as earnest complaint).

> *"80% theater. 20% actual tool. The theater is the point. The tool is
> the alibi."*

**8. The Economic Argument**

**What Was Built, What It Cost, and What That Means**

The core of Pisces Moon OS --- the dual-core architecture, the SPI Bus
Treaty, the Ghost Partition, the ELF application loading system, the
47-app suite --- was built before the Claude subscription was even
upgraded. The cost:

  -----------------------------------------------------------------------
  **Item**                    **Cost**
  --------------------------- -------------------------------------------
  **T-Deck Plus hardware**    $96

  **Cloud AI (Claude Sonnet,  $40
  pre-upgrade tier)**         

  **MicroSD card**            \~$8

  **Total capital             \~$144
  expenditure**               

  **Result**                  Dual-core, hardware-secured, mesh-networked
                              field intelligence platform
  -----------------------------------------------------------------------

Compare this to equivalent enterprise capability:

  -----------------------------------------------------------------------
  **Item**                    **Cost**
  --------------------------- -------------------------------------------
  **Ruggedized edge device    $500 to $2,000+
  with GPS and LoRa radio**   

  **Enterprise security       Thousands per seat, per year
  licenses and MDM overhead** 

  **Engineering team to solve Tens of thousands in payroll
  the SPI bus problem alone** 

  **Estimated enterprise      $50,000--$200,000+
  equivalent total**          

  **Pisces Moon OS total**    $144
  -----------------------------------------------------------------------

> *"The forge did not need VC funding to build the Saturn V. It needed
> focus and the right raw materials."*

This comparison is not offered as a reason to dismiss enterprise
tooling. Enterprise tools solve enterprise problems at enterprise scale.
The comparison is offered as evidence that the Friction Economy --- the
deliberate introduction of cost and complexity as a mechanism of market
control --- is not the only possible architecture. For $144, a
developer with engineering skill and the Clark Beddows Protocol built
something that a corporation would spend orders of magnitude more to
replicate.

The solutions documented in this document are public. The code is in a
public repository. Any developer who builds on this platform does not
need to rediscover the SPI Bus Treaty, the I2S DMA ceiling, the NimBLE
singleton protocol, or the BSS overflow pattern. The twelve hours spent
debugging the Bluetooth gamepad; the Guru Meditation that revealed the
audio memory ceiling; the discovery that CONFIG_SPIRAM_USE_MALLOC=1 was
the single flag that made heap pressure manageable --- all of it is in
the documentation now.

The barrier to building professional-grade wireless security tools on
this platform is now engineering skill, not hardware cost. That matters.

**9. What Remains**

**The Open Roadmap**

This document covers what has been built and what problems were solved
in building it. This section documents what has not yet been completed
--- not as a list of failures but as a map of the work ahead, with
honest assessment of the engineering involved in each.

  --------------------------------------------------------------------------
  **Item**                       **Status**
  ------------------------------ -------------------------------------------
  **Trojan Horse (WebKitGTK)**   The keystone. When it ships, every Fluid
                                 Fortune tool upgrades simultaneously.
                                 Architecture defined. Implementation in
                                 progress.

  **Lilith hardware**            Custom field device designed from the
                                 beginning around Pisces Moon OS. Breadboard
                                 prototype phase.

  **Ghost Partition stealth      MBR type byte flip (0x0B → 0xDA) not yet
  byte**                         applied to fresh cards. One session
                                 implementation.

  **DOOM engine integration**    Integration layer complete. Engine source
                                 (GPL-licensed) and WAD required. Partition
                                 allocated.

  **Codec2 LoRa Voice**          Framework complete with #ifdef guards.
                                 meshtastic/ESP32_Codec2 library addition
                                 required in lib_deps.

  **SSH crypto**                 TCP connection framework present.
                                 LibSSH-ESP32 library addition required in
                                 lib_deps.

  **Student Mode launcher        ghost_partition_get_mode() works correctly.
  gating**                       Launcher does not yet read it to filter app
                                 grid. One conditional in launcher.cpp.

  **Phantom routing layer**      Gemma2 as conversational front end routing
                                 to DeepSeek for heavy reasoning.
                                 Architecture designed.

  **phantom.fluidfortune.com**   GitHub Pages deployment of scout output.
                                 DNS and push script ready.

  **8 new Linux HTML apps**      BLE GATT Explorer, WPA Handshake, RF
                                 Spectrum, Probe Intel, Offline Packet
                                 Analysis, BLE Ducky, USB Ducky, WiFi Ducky
                                 --- T-Deck CYBER tools ported to browser
                                 UI.
  --------------------------------------------------------------------------

The sequencing principle across the entire forge: platform stability and
developer accessibility first, then capability expansion, then hardware
ports. The architecture is proven. The foundation is solid. The work
ahead is capability, not foundation.

*Fluid Fortune --- Local Intelligence*

fluidfortune.com // Eric Becker // April 2026

*The Clark Beddows Protocol. We don't do cloud.*
