---
title: "Designing a 2.4GHz FMCW Radar on a Single PCB"
date: 2026-07-24
categories: [Projects, Embedded Systems]
tags: [rf, pcb, kicad, radar, stm32]
math: true
---

## Introduction

This project documents the design of a 2.4GHz FMCW (Frequency Modulated
Continuous Wave) radar, built as a single 4-layer PCB in KiCad. The board
covers the full signal chain — chirp generation, power division,
amplification, transmit and receive front ends, and a baseband section that
extracts range and speed data from a target.

## How a Radar Works

A radar transmits electromagnetic waves and listens for the echo reflected
back by a target. Measuring the time between transmission and reception is
what lets us calculate distance.

### Continuous Wave (CW) Radar

In optics and acoustics, motion between a wave source and an observer causes
an apparent shift in frequency — the Doppler effect. This is the operating
principle behind CW radar.

A CW radar transmits an unmodulated signal, and the target reflects a portion
of that energy back toward the antenna. If the transmitted signal has
frequency `f0`, and the target's motion shifts the returning signal by `±fd`,
the received echo arrives at `f0 ± fd` — a `+` for an approaching target, and
a `-` for a receding one.

The echo is mixed with a copy of the transmitted signal to produce a
Doppler beat note at frequency `fd`, which gives us the target's radial
velocity. What CW radar *can't* give us is range: because the transmitted
signal is a single unmodulated tone, there's no way to time-stamp any
particular part of it, and no bandwidth available to encode that timing
information.

### Frequency Modulated Continuous Wave (FMCW) Radar

FMCW radar extends the CW principle to solve exactly that problem. Instead of
transmitting a single frequency, the signal's frequency is swept
(modulated) over time — this sweep is called a **chirp**, and it's what
creates the bandwidth needed to recover timing information, and therefore
range.

The core range equation is:

$$
d = \frac{c \times t}{2}
$$

where `d` is the distance to the target, `c` is the speed of light, and `t`
is the round-trip time for the signal to travel to the target and back. We
divide by 2 because `t` accounts for the full there-and-back trip, not just
the one-way distance.

### Radar Cross Section and the Radar Equation

How much energy actually comes back from a target isn't just a function of
size — it depends on the target's **radar cross section (RCS, σ)**, a
measure of how "visible" an object is to radar based on its size, shape,
material, and orientation relative to the radar. Two objects of similar
physical size can have very different RCS values depending on how they
reflect the incoming wave.

This feeds into the **radar range equation**, which relates the power
actually received back at the radar (`Pr`) to the transmitted power (`Pt`),
antenna gain (`G`), wavelength (`λ`), target RCS (`σ`), and range (`R`):

$$
P_r = \frac{P_t \, G^2 \, \lambda^2 \, \sigma}{(4\pi)^3 \, R^4}
$$

The detail worth calling out is the **R⁴ in the denominator** — received
power falls off with the *fourth* power of range, not the square. Double
the distance to a target, and the returned signal is 16× weaker. This is
part of why the receive chain's noise performance matters so much: at
longer ranges, the wanted echo can be extremely close to the noise floor,
so every dB of noise figure in the LNA and mixer stages directly affects
how far out a target can still be detected.

A more detailed treatment of RCS and the radar equation as applied to FMCW
systems specifically is covered in Analog Devices' write-up on building a
24 GHz FMCW radar (linked in the references below).

The project is inspired by [Henrik Forsten's 6GHz FMCW radar](https://hforsten.com/6-ghz-frequency-modulated-radar.html).

## Design Overview

The board is organized into four sections: **power supply, microcontroller,
baseband, and RF.**

Components were chosen by working backward from the target system
requirements:

- Operating in the 2.4 GHz ISM band keeps signal generation simple, since
  well-documented, easily sourced parts exist for this range. The **MAX2750**
  voltage-controlled oscillator (VCO) was chosen for this — it generates
  2.4–2.48 GHz from a 0.4–2.2V control input.
- That gives a usable bandwidth of 80 MHz (2.48 GHz − 2.4 GHz).
- Targeting a maximum range of ~50 m, with enough velocity resolution to
  distinguish a slow-moving object from one moving at ~6 m/s, worked out to a
  **5 ms chirp period** (the full derivation is below).
- Generating that chirp needs a DAC to drive the VCO's control voltage, and
  the receiver needs an ADC fast enough to sample the resulting IF signal.
- Doing the range/Doppler processing **on-board** meant needing a
  microcontroller capable of a 256-point FFT.
- Power needed to come over USB, with the same USB connection also used to
  stream data and program the MCU.

These requirements led to the **STM32G473**: a 170 MHz Cortex-M4 with USB
support and DAC/ADC fast enough for both the chirp generation and baseband
sampling.

With the core signal-generation and processing platform chosen, the rest of
the signal chain was picked around it:

- **Power division** (splitting the chirp into a transmit copy and a local
  mixing reference) needed to be low-loss, low-cost, and easy to implement
  directly in copper on the PCB — a Wilkinson power divider fits all three,
  with no external inductors or lumped components required.
- **Transmit power amplification** (RFX2401C) needed to cover the
  2.4 GHz ISM band in a small SMD footprint, with enough output power to
  reach the 50 m target range, without requiring exotic biasing or matching
  networks. It's driven directly by the WPD's transmit arm, with no
  preceding gain stage.
- **LO reference amplification** (GALI-84+) boosts the WPD's local
  oscillator arm up to a level suitable for driving the mixer, followed by
  a resistive attenuator pad to bring that level into the mixer's
  characterized operating range (details below).
- **Receive amplification** (BGA2866) needed a low noise figure above all
  else, since this is the first active stage the (very weak) echo signal
  sees, and it sets the noise floor for the entire receive chain.
- **Mixing** (ADE-25MH+) needed to be a double-balanced mixer suited to
  this frequency range, to combine the LO reference and RX signal into a
  clean IF output.
- **Baseband amplification** (TLV4314) needed to be low-power, low-noise,
  and ideally include some built-in filtering, to avoid needing a separate
  discrete anti-alias filter stage ahead of the ADC.
- **Adjustable gain control** (MCP4017) needed to let gain be tuned in
  software rather than fixed at design time, since the right amount of
  baseband gain changes with target range.


### Deriving the Chirp Period

The chirp period isn't an arbitrary choice — it falls out of the velocity
resolution requirement. In an FMCW radar, velocity is recovered from the
**Doppler shift between successive chirps**, and like any sampled signal,
that shift can only be measured unambiguously up to the Nyquist limit of
how often it's sampled — in this case, once per chirp.

The Doppler shift for a target moving at velocity `v` is:

$$
f_d = \frac{2v}{\lambda}
$$

At the center frequency of this band (2.44 GHz), `λ = c / f0 ≈ 0.123 m`
(12.3 cm). For the target maximum velocity of 6 m/s:

$$
f_d(6\text{ m/s}) = \frac{2 \times 6}{0.123} \approx 97.6\text{ Hz}
$$

To sample this shift without aliasing, the chirp repetition rate (1 /
T_chirp) must be at least twice this Doppler frequency (the Nyquist
criterion applied to chirp-to-chirp sampling):

$$
\frac{1}{T_{chirp}} \geq 2 \times f_d(v_{max})
$$

$$
T_{chirp} \leq \frac{1}{2 \times f_d(v_{max})} \approx \frac{1}{2 \times 97.6} \approx 5.12\text{ ms}
$$

Rounding down gives the **5 ms chirp period** used in this design — chosen
so that a 6 m/s target sits right at (just under) the maximum unambiguous
velocity the system can report.

## System Signal Chain

### Chirp Generation

The MCU's internal DAC produces the 0.4–2.2V ramp over the 5 ms chirp period
and feeds it to the VCO, which converts it into the swept 2.4–2.48 GHz RF
signal.

### Power Division

This design uses a **heterodyne receiver** architecture, where a copy of the
transmitted signal is mixed with the received echo to produce the IF signal.
That means the chirp signal needs to be split into two identical copies —
one to transmit, one to keep locally as a mixing reference. This split is
done with a **Wilkinson power divider (WPD)**, implemented directly as
microstrip traces on the PCB.

A Wilkinson divider splits an input signal into two equal-amplitude,
equal-phase outputs, while keeping all three ports matched to the system
impedance (50Ω) and providing isolation between the two outputs, so a
mismatch on one branch doesn't leak into the other. It's built from two
quarter-wavelength (λ/4) transmission line sections (impedance √2 × Z₀,
≈70.7Ω for a 50Ω system) and a single isolation resistor (2 × Z₀ = 100Ω)
bridging the two output ports — no inductors or lumped capacitors required,
which is why it's such a common choice for simple RF PCB designs.

Each λ/4 arm on this board measures approximately **18mm** (total trace
length, including any bends). 
This was confirmed directly using KiCad's built-in **PCB Calculator**, using
this board's actual stackup — a top dielectric height of **0.2104mm**, with a target
impedance of 70.7Ω (`√2 × 50Ω`) and an electrical length of 90° (a quarter
wavelength) at
2.44 GHz. The calculator synthesized a trace width of **0.1907mm** and a
physical length of **18.1393mm**, closely matching the as-built ~18mm arm
length.

![KiCad PCB Calculator with W/L results](/assets/img/posts/FMCW-radar/WPD-arms-kicad-calculator.png)_KiCad PCB Calculator with W/L results_

This implementation also adds a small (1Ω) series resistor on each output
arm, between the arm itself and the shared 100Ω isolation resistor, rather
than connecting the isolation resistor directly across the two arms. These
are **test/probe points** — the small series resistance lets each arm be
probed individually (e.g. with a spectrum analyzer or power meter) during
bring-up, without needing to break the trace or otherwise disturb the
divider's electrical characteristics.

### LO Reference Amplification and Attenuation

The WPD's two outputs feed two different destinations: one goes straight
to the transmit chain (below), and the other — the local oscillator
reference — is amplified by the **GALI-84+** MMIC gain block before
reaching the **ADE-25MH+** mixer's LO port. The GALI-84+ contributes
roughly 17–19dB of gain, taking the WPD's ~0dBm output up to somewhere
around +17 to +19dBm.

The ADE-25MH+'s datasheet characterizes its conversion loss, isolation,
and VSWR at three specific LO drive levels — +10dBm, +13dBm, and +16dBm —
this is the range the part is actually validated to perform well at, and
is a separate spec from the mixer's generic port power rating (200mW /
+23dBm), which is only a damage threshold, not a guarantee of good
performance below it. Since the GALI-84+'s output sits above all three
characterized curves, a **6dB pi-pad attenuator** sits between the
GALI-84+ and the mixer's LO port, bringing the drive down to roughly
+13dBm — centered in the mixer's characterized operating range.

The pi-pad is a simple three-resistor resistive network — two shunt
resistors to ground and one series resistor — sized for 50Ω matching at
6dB of attenuation: **R1 = R3 = 150.5Ω** (shunt to ground), **R2 = 37.4Ω**
(series, between the two signal nodes). Because it presents a matched 50Ω
impedance looking into either port, it drops the signal level without
disturbing the match on either side of it.
This is taken from this article on Electronics Tutorials [Pi-pad Attenuator](https://www.electronics-tutorials.ws/attenuators/pi-pad-attenuator.html).

### Transmit Chain

One output of the WPD — the transmit copy — feeds directly into the
**RFX2401C**, before being routed to an SMA connector for connection to
the transmit antenna. No gain stage precedes it on this arm: the WPD's
~0dBm output already sits comfortably under the RFX2401C's +5dBm maximum
input rating, so no amplification is needed ahead of it.

The RFX2401C is a fully integrated CMOS front-end IC that natively
includes a PA, LNA, and TX/RX switch — but in this design it's configured
to operate as a **transmit-only** amplifier; the integrated LNA and RX
path aren't used here, since receive is handled separately by the
BGA2866/ADE-25MH+ chain described below. In its high-power TX setting, the
RFX2401C typically delivers on the order of +18 to +20 dBm of output power
from around 20dB of onboard gain — with the WPD's ~0dBm input, that puts
the SMA connector's output in roughly the same 20dBm level.

A connection is also brought from the MCU to the RFX2401C's **TX enable
(TXEN)** pin. This isn't required for basic operation, but it gives the
MCU a way to switch transmission off in software — useful for things like
duty-cycling the transmitter, or simply disabling TX during debugging
without needing to touch the board's power rails.

### Receive Chain

The receive path starts at a second SMA connector, feeding in from the
receive antenna via RF cable.

The incoming signal first passes through a **BGA2866** low-noise amplifier
(LNA). This stage matters because every amplifier in a receive chain adds
some of its own noise on top of the signal, and — per the Friis noise
formula — the *first* stage's noise figure dominates the noise contribution
of the entire chain, since everything after it gets divided down by the
gain that precedes it. Using a low-noise, high-gain part right after the
antenna boosts the wanted signal well above the noise floor before it
reaches any noisier stage further down the chain (like the mixer), which
preserves the receiver's overall sensitivity.

From the LNA, the signal goes to the **ADE-25MH+** mixer, which combines it
with the local oscillator reference (the copy of the chirp signal split off
by the WPD) to produce the IF signal — the difference-frequency signal that
actually carries the range and Doppler information.

The IF signal then passes through a **TLV4314** op-amp. This is a low-power,
low-noise (16 nV/√Hz), rail-to-rail op-amp with an internal RF/EMI filter,
acting here as the baseband gain stage: amplifying the (typically weak) IF
signal up to a level the STM32's ADC can resolve, while its internal
filtering helps reject out-of-band interference before sampling.

The TLV4314's gain is set via an **MCP4017** digital potentiometer over
I²C. A fixed gain wouldn't work well here, since the correct amount of
amplification depends on target distance (far echoes are much weaker than
near ones) and on board/antenna losses — the digital pot lets the MCU adjust
gain in software instead of locking in one setting at design time.

Finally, the amplified IF signal is sampled by the STM32's ADC at **50 kSPS**.

### Range and Speed Extraction

In an FMCW radar, the IF signal's **frequency** is proportional to target
range, and its **phase change from one chirp to the next** is proportional
to target velocity. Processing works in two stages:

1. An FFT across the samples *within* a single chirp turns that chirp into
   a range profile — the position of each peak corresponds to a target's
   distance.
2. A second FFT taken *across successive chirps*, at a given range bin,
   reveals the Doppler shift at that range — and therefore the target's
   speed.

This two-step range-Doppler FFT is why the chirp timing (5 ms) and FFT size
(256-point) were chosen around the target range and velocity resolution
from the start.

### Debug, Status, and Power

Five header pins (SWCLK, SWDIO, NRST, +3V3, GND) break out the MCU's SWD
interface for debugging — not the typical pinout/ordering seen on most
dev boards, but functional.

Two onboard LEDs indicate:
- **Power presence** — lit whenever power is actively flowing through the
  board.
- **Firmware status** — blinked at a low frequency by a simple line of
  firmware code, to confirm the MCU is running.

Power arrives over USB as 5V DC, and is regulated down to three separate
3.3V DC rails: **+3V3**, **+3V3_VCO**, and **+3V3_RF**. These are kept
separate — rather than sharing a single 3.3V rail — so that noisy digital
switching on the MCU side doesn't couple into the more sensitive VCO and RF
sections through a shared supply. R/L/C filtering is used throughout the
board for the same reason: keeping noise off the rails that feed
noise-sensitive RF components.

## PCB Layout

RF traces are routed with curved bends where possible, keeping current paths
smooth and reducing EMI — bend radii are kept to at least 3× the trace width
(0.35 mm).

The stackup was chosen from JLCPCB's published options for a 4-layer board
(see: [JLCPCB impedance reference](https://jlcpcb.com/impedance)).

![JLCPCB Impedance-controlled stackup](/assets/img/posts/FMCW-radar/JLCPCB-stackup.png)_JLCPCB Impedance-controlled stackup_ 

Trace impedance was calculated using DigiKey's [PCB trace impedance
calculator](https://www.digikey.in/en/resources/conversion-calculators/conversion-calculator-pcb-trace-impedance).

![DigiKey trace impedance calculator](/assets/img/posts/FMCW-radar/tracewidth-DigiKey.png)_DigiKey trace impedance calculator_ 

The transmit and receive SMA connectors are placed at a 90° angle to each
other, separated by a dense wall of grounded vias for **isolation**. 
With TX and RX on the same board in close proximity, transmitted energy 
can couple directly into the receive chain
rather than only being picked up after reflecting off an actual target. That
raises the receiver's noise floor and can desensitize or even saturate the
front end. A dense via fence between the two connectors acts as an RF
barrier, shorting out stray fields before they can couple across.

The overall design assumes the transmit and receive sections connect to
their antennas via RF cable, rather than mounting antennas directly on the
board — this lets the antennas be spaced further apart to reduce
interference. Careful antenna design is still needed to maximize isolation
between the two.

Board dimensions: **80mm × 40mm**, with the SMA connectors adding roughly
10mm of protrusion beyond the board edge.

## Next Steps

- Develop firmware using STM32CubeIDE / STM32CubeMX
- Design antennas for the transmit and receive sections
- Simulate system performance before fabrication

## Schematic and PCB Renders

### Schematics:
The schematic design involves  the use of hierarchical pages, divided as follows:

![Root Page showing connections between subpages](/assets/img/posts/FMCW-radar/Mainpage.png)_Root Page showing connections between subpages_

![Microcontroller Subsection Page](/assets/img/posts/FMCW-radar/MCU-subsection.png)_Microcontroller Subsection Page_

![RF Subsection Page](/assets/img/posts/FMCW-radar/RF-subsection.png)_RF Subsection Page_

![Baseband Subsection Page](/assets/img/posts/FMCW-radar/Baseband-subsection.png)_Baseband Subsection Page_

![Power Supply Subsection Page](/assets/img/posts/FMCW-radar/PowerSupply-subsection.png)_Power Supply Subsection Page_

### PCB 
The following screenshots depict the different layers of the PCB.

![All layers of the PCB](/assets/img/posts/FMCW-radar/PCB-AllLayers.png)_All layers of the PCB_

Note: the zones have not been filled in the above image.

![Upper layer routing](/assets/img/posts/FMCW-radar/PCB-F.Cu-Routing.png)_Upper layer routing_

Here, the routing of traces on the upper layer is shown. Most of the traces carry either RF or digital signals. The zone is not filled. 

![Filled Ground plane](/assets/img/posts/FMCW-radar/PCB-In1.Cu-FilledGNDPlane.png)_Filled Ground plane_

The zone has been filled in the above image. It shows the uniform ground plane present on the board's second layer. 

![Power Supply layer routing](/assets/img/posts/FMCW-radar/PCB-In2.Cu-Routing.png)_Power Supply layer routing_

The above image shows the distribution of the power signals on layer 3 of the board (+5V,+3.3V,+3.3_VCO,+3.3V_RF).

![Baseband Layer routing](/assets/img/posts/FMCW-radar/PCB-B.Cu-Routing.png)_Baseband Layer routing_

The back layer of the PCB holds the sensitive baseband components, maximising distance from RF and digital sections on the first layer.


### 3D Models

Using KiCad's 3D viewer option, we can see how the board physically looks:

![PCB, Top View](/assets/img/posts/FMCW-radar/PCB-3DModel-TopView.png)_PCB, Top View_

![PCB, Bottom View](/assets/img/posts/FMCW-radar/PCB-3DModel-BottomView.png)_PCB, Bottom View_

![PCB, Isometric View](/assets/img/posts/FMCW-radar/PCB-3DModel-IsometricView.png)_PCB, Isometric View_

## References

- [Henrik Forsten's 6GHz FMCW radar project](https://hforsten.com/6-ghz-frequency-modulated-radar.html)
- [How to Build a 24 GHz FMCW Radar System](https://www.analog.com/en/resources/technical-articles/how-to-build-a-24-ghz-fmcw-radar-system.html), Analog Devices
- [Pi-pad Attenuator](https://www.electronics-tutorials.ws/attenuators/pi-pad-attenuator.html), Electronics Tutorials
- [JLCPCB impedance-controlled Stackups reference](https://jlcpcb.com/impedance)
- *Introduction to Radar Systems*, Merrill I. Skolnik
- *High-Speed Digital Design: A Handbook of Black Magic*, Howard Johnson and Martin Graham

## Things I learnt:
- An earlier version of this board used two amplifiers for the transmit chain, one GALI-84+ connected to the RFX2401C input. However, the powerful output of the GALI-84+ was well above the RFX2401C recommended input level; therefore, the design had to be reworked to remove the GALI-84+. 
- The upper layer of the board initially did not have a copper pour. This meant that via stitching was not serving its purpose. I added the filled zone to the upper layer, providing a way for accidental leaks to find a low impedance return path.