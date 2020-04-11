---
title: "I wrote a keyboard remapper for wayland"
date: 2020-04-11T17:40:00Z
thumbnail: /images/posts/writing-keyboard-remapper-wayland/hero.jpg
categories:
- Linux
tags:
- linux
- wayland
---

Recently I bought a new laptop with the intention of running Linux with Sway as my WM. This is the setup I run on my workstation where I have a 60% keyboard on which I use caps lock as a function key for a second layer.

Because muscle memory is a thing I have been using Karabiner Elements to remap the keyboard on my work macbook pro but I could not find a suitable alternative for Linux when using a Wayland compositor.

### Why this is difficult on Wayland

In the old X11 system any application could read the keyboard input from the X server (I do not have enough knowledge about X11 to claim this fact but that's how I understand it). This makes it easier to write keyboard remappers for it, as well as more neferious things like key loggers.

However, with the design on Wayland it was decided that for security's sake only the active window should be able to read the incoming keyboard events. This is to protect against key loggers but makes it difficult to write a keyboard remapper at the compositor level until some protocol has been developed for it.

### Let's move up the chain

It is however possible to take the Display server/Compositor out of the equation entirely. If you inspect the directory `/dev/input` you will find a bunch of files called `eventX` (X being some number). These are your input devices and they can be read to get all events from them. Traditionally, distros make these files owned by the input group, so if your user is a part of that group you can read and write to those devices. You can even claim a device so that you get exclusive access to it, that is to say only you will get the events and unless you do something with those events the keyboard will appear to not work until it is released again.

So that's reading from the keyboard taken care of, what about sending those events back into the system? That's where uinput comes in.

The kernel documentation writes:

> uinput is a kernel module that makes it possible to emulate input devices from userspace. By writing to /dev/uinput (or /dev/input/uinput) device, a process can create a virtual input device with specific capabilities. Once this virtual device is created, the process can send events through it, that will be delivered to userspace and in-kernel consumers.

To simplify, it is used to create virtual input devices that you can programmatically send events to so that it seems to the system that a real input device is being used. The module `uinput` has to be loaded and a udev rule can be written so that it can be used by a user (by default the file is owned by root with permissions `600`).

What all this means is that a software can be written that claims an input device, reads the events, manipulates them in some way (remap) and writes them to a virtual keyboard. This is the approach used by the keyboard rebinding daemon [Hawck][1] which I did consider using but due to its complexity and bigger scope than I needed (it supports writing macros in lua) I decided against it. The complexity of the software especially makes it difficult to audit the software which in this case I think is very important as this is a prime location to write in a key logger, remember this software by design has to be able to read every single event your keyboard emits.

So naturally I wrote my own.

## Waybind

With the help of a couple of libraries (they had to be audited as well) it was really not a difficult task to support a two key combination remapping. It ended up being less than 500 lines of go code, not including the go modules I used to read the keyboard events and write to uinput, so it should be trivial to audit. It does not and will not support more complicated macros like Hawck.

All it needs is a YAML config file similar to the following:

```yaml
device: /dev/input/event0 # Path to the input device to be read
rebinds:
  # Binds KEY_GRAVE to KEY_ESC
  # If modifier KEY_CAPSLOCK is also pressed then it's still KEY_GRAVE but KEY_CAPSLOCK is removed
  # If modifier is KEY_LEFTSHIFT then it's KEY_LEFTSHIFT + KEY_GRAVE
  - from: KEY_GRAVE
    to: KEY_ESC
    with_modifiers:
      - modifier: KEY_CAPSLOCK
        to: KEY_GRAVE
      - modifier: KEY_LEFTSHIFT
        to: SKIP

  # Binds KEY_CAPSLOCK + KEY_BACKSPACE to KEY_DELETE
  - from: KEY_BACKSPACE
    with_modifiers:
      - modifier: KEY_CAPSLOCK
        to: KEY_DELETE

  # KEY_CAPSLOCK + KEY_F12 will make waybind exit
  - from: KEY_F12
    with_modifiers:
      - modifier: KEY_CAPSLOCK
        to: EXIT

  # Completely unbind KEY_CAPSLOCK
  - from: KEY_CAPSLOCK
    unbind: true
```

Each rebinding prioritizes `with_modifiers` highest in the order of the list it is given, then it checks for if `unbind` is true and finally goes with whatever is in `to`. It goes with the first match it encounters and then moves on to the next rebinding. In `with_modifiers` it is possible to set `to` to `SKIP` to stop looking for matches and move on to the next rebinding and `EXIT` to make waybind exit.

I use `SKIP` in my rebinding for grave, I want grave to rebind to escape when it is pressed on its own, keep being grave when pressed with caps lock (but only grave, caps lock is removed) and when pressed with shift to remain shift + grave. This is to mimic [Grave Escape][2] in QMK that's running on my 60% keyboard.

I wrote the `EXIT` feature as a safeguard due to my lack of confidence in my ability to write correct code. If waybind starts to act funky I can press the key combination and make waybind exit, systemd will handle restarting it if configured with `Restart=always`.

All available key codes can be found [here][3].

But feel free to inspect the [code][4].

Here's how I run it:

- Create group `uinput`.
- Create user `waybind` and add it as a member of `uinput` and `input`.
- Add a [udev rule][5] to make `uinput` group be able to read and write to uinput.
- Create a systemd service that runs waybind as user `waybind`. Additionally I add `PrivateNetwork=true` to the service so that it runs in its own network namespace, disconnected from the internet. More info [here][6].

I have packaged waybind for Nix in my local repository [here][7] and made a module to set up the aformentioned setup [here][8].

I hope this is useful to someone else.

**Discuss on [Hacker News][9] or [Reddit][10]**

[1]: https://github.com/snyball/Hawck
[2]: https://beta.docs.qmk.fm/using-qmk/advanced-keycodes/feature_grave_esc
[3]: https://github.com/arnarg/waybind/blob/master/src/ecodes.go
[4]: https://github.com/arnarg/waybind
[5]: https://github.com/arnarg/waybind/blob/master/udev/99-uinput.rules
[6]: https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateNetwork=
[7]: https://github.com/arnarg/config/blob/master/packages/waybind/default.nix
[8]: https://github.com/arnarg/config/blob/master/modules/programs/waybind/default.nix
[9]: https://news.ycombinator.com/item?id=22843070
[10]: https://www.reddit.com/r/linux/comments/fzc1bg/waybind_dead_simple_remapper_for_wayland_based_on/
