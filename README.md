# WOLF_OS
Low Level Custom Linux Distro For Rasberry PI 5 
This is where you move from being a "user" of an OS to being its architect. Building a DIY real-time stack for the Raspberry Pi 5 requires a shift from the older Xenomai 3 (Cobalt) to **Xenomai 4 (EVL)**.

Unlike the dual-kernel approach of the past, EVL is more "integrated" and designed to work with modern 64-bit ARM kernels (like the ones the Pi 5 uses).

---

## 1. The Kernel: Patching for EVL

Since standard Linux isn't designed for sub-millisecond guarantees, you must patch the Pi 5's kernel.

1. **Download Source:** Grab the official Raspberry Pi Linux source (currently targeting kernel 6.6 or 6.1).
2. **Apply EVL Patch:** Download the EVL core patch from the [Xenomai 4 project](https://www.google.com/search?q=https://v4.xenomai.org/).
3. **Cross-Compile:** You’ll want to build this on a faster PC using an ARMv8 toolchain.
* Enable `CONFIG_EVL` in the kernel config.
* Disable CPU scaling and frequency switching (`CONFIG_CPU_FREQ`), as these introduce unpredictable latency (jitter).



---

## 2. The Audio Driver (The RTDM Challenge)

This is the "real work." To get hard real-time audio, you cannot use the standard ALSA drivers because they rely on the "slow" Linux kernel to schedule interrupts. You need an **Out-of-Band (OOB)** driver.

* **The Problem:** The Pi 5 uses the **RP1 I/O controller**. You’ll need to write a driver that handles the I2S clock and data transfers in the EVL domain.
* **The Shortcut:** Check the [Elk Audio rpi-evl-audio-driver]() on GitHub. While originally for Pi 4, the logic for handling I2S interrupts in an EVL context is the best reference available. You will need to adapt the register offsets for the Pi 5’s hardware.

---

## 3. Integrating JUCE with EVL

Since JUCE doesn't have an "EVL Audio Device" out of the box, you have to write one.

### The Custom `AudioIODevice`

You must subclass `juce::AudioIODevice`. Instead of calling ALSA functions, your `open`, `start`, and `stop` methods will interact with EVL device files (e.g., `/dev/evl/pcm0`).

### The Real-time Loop

In your audio callback thread:

1. **Thread Promotion:** Use `evl_attach_self()` to promote the JUCE audio thread to "In-Band" (real-time) status.
2. **Avoid Mode Switches:** This is critical. If your JUCE code calls a standard Linux function (like `std::cout` or `new`), the EVL core will "demote" the thread back to slow Linux, causing an audio glitch.
3. **Communication:** Use **EVL Flags** or **EVL Queues** to send MIDI or UI data between your real-time synth engine and the "slow" GUI/Management thread.

---

## Why go through this "Pain"?

For an electrician and developer like yourself, this is the digital equivalent of wiring a high-voltage industrial panel: you are managing the raw flow of energy (data) with zero room for error.

* **Deterministic Latency:** You can achieve a buffer size of **8 or 16 samples**. On a Pi 5, this means your polysynth will respond to a MIDI note in less than **0.5ms**.
* **CPU Efficiency:** Because the OS isn't "guessing" when to run your code, you can push the CPU to 90%+ load with complex FM or physical modeling synthesis without a single crackle.

[Real Time Audio Digital Signal Processing DAC with Raspberry Pi 5](https://www.youtube.com/watch?v=hN1atN5JsJY)

This video demonstrates a high-performance audio setup on the Raspberry Pi 5, providing a visual guide to the hardware and software integration required for low-latency DSP.
