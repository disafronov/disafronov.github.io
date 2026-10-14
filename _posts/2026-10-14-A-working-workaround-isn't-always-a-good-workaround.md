---
title: "A working workaround isn't always a good workaround"
date: "2026-10-14 13:56:00 +0400"
tags: [linux, kernel, alsa, debugging]
---

Another Linux kernel update broke the internal speakers on my Lenovo Legion 7.

Again.

This wasn't the first time a kernel update had left me without working speakers.

Headphones worked. The audio devices were detected. PipeWire was routing audio correctly. But the internal speakers remained completely silent.

This time, the regression appeared when moving from kernel `7.0.0-34-generic` to `7.0.0-38-generic`.

Fortunately, I didn't have to spend hours digging through the Linux kernel source code.

I found an existing Ubuntu Launchpad bug report: [2163143](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2163143)

And this is where I'd like to give credit to the people who investigated the issue before me.

The original reporter had done some impressive detective work. By comparing HDA quirk tables between kernel versions, they identified that a newly introduced PCI subsystem ID quirk (`17aa:3811`) was taking precedence over the correct Realtek ALC287 codec subsystem ID quirk (`17aa:3855`).

As a result, the driver selected the wrong fixup, the two Cirrus Logic CS35L41 speaker amplifiers weren't properly initialized, and the internal speakers remained silent.

Other contributors identified the corresponding upstream Linux fix, and the Launchpad discussion documented a working workaround:

`options snd-hda-intel model=17aa:3855,17aa:3855`

I applied the published workaround, tested it on my laptop, and confirmed that the speakers worked again.

Problem solved?

Functionally, yes.

But there was one aspect of the workaround I wanted to improve.

The `model` parameter of `snd-hda-intel` is an array, with entries corresponding to HDA controller instances.

My laptop has two HDA controllers: Intel and NVIDIA. Their registration order can change between boots, so the existing workaround assigns the same model override to both.

That works for my configuration, but it applies the override more broadly than necessary.

I wanted to preserve the workaround's behavior across changes in the registration order of these two controllers, while limiting the actual override to the affected codec.

And this is where the work already done by other engineers really paid off.

Instead of spending time rediscovering the root cause, I could focus on making the existing solution more targeted.

Linux already provides a mechanism for this: HDA early patching.

It allows the driver to load a configuration file that identifies a specific codec and specifies the model to use.

My patch file looks like this:

```ini
[codec]
0x10ec0287 0x17aa3855 0

[model]
17aa:3855
```

And the driver configuration becomes:

```modprobe
options snd-hda-intel patch=lenovo-legion-alc287.fw,lenovo-legion-alc287.fw
```

The same patch file is supplied to both existing HDA controller instances, but the `[codec]` section ensures that the model override is applied only to the matching Realtek ALC287 codec.

I also added a small `initramfs-tools` hook to make the firmware file available during early boot, then rebuilt the initramfs.

After rebooting into `7.0.0-38-generic`, I verified the results:

- The Realtek ALC287 selected the correct fixup.
- Both CS35L41 amplifiers loaded their DSP firmware and initialized successfully.
- The internal speakers worked.
- Neither the Intel nor the NVIDIA HDMI codec received a model override.
- NVIDIA HDMI video and audio output to my TV still worked.

This is still a workaround, not a fix for the kernel regression itself.

The upstream fix already exists, and getting it into Ubuntu remains the proper long-term solution.

But for my current hardware configuration, the workaround now handles either registration order of the two HDA controllers while applying the model override only to the affected codec.

And the best part of this story isn't that my speakers work again.

It's that other engineers had already invested their time in investigating the issue, identifying the root cause, and sharing their findings publicly.

Their work saved me from repeating the same investigation.

Instead, I could spend that time improving a practical solution and verifying its behavior on my own hardware.

In the end, I got working speakers, a more targeted workaround, and another reminder of just how valuable well-researched public bug reports can be.

Many thanks to everyone involved in the Launchpad discussion for the investigation, the workaround, and the hours of debugging they saved me.

Bug report: [2163143](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2163143)

Upstream fix: [dd074f0](https://github.com/torvalds/linux/commit/dd074f04e04648d89d9d10ae9846cd057c97b385)

My workaround^2: [2163143/comments/9](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2163143/comments/9)
