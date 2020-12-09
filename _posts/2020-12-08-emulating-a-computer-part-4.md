---
layout: post
title: "Emulating a Computer: Timing and Input"
tags: [programming, emulation, games]
date: 2020-12-08 18:44 -0800
description: Correctly emulating time in a CHIP-8 emualtor.
---

{% include video_embed.html file='vers' %}

When [last we left off]({% post_url 2020-12-07-emulating-a-computer-part-3 %}), we developed a full-featured CHIP-8 emulator. Unfortunately, it is **really** slow. Why is it slow? Well if you look back at the main loop you'll see that we're rendering to the screen after every cycle is executed. With vsync enabled, SDL will try to lock rendering to your monitor's refresh rate (probably 60Hz). What that means is that the `SDLViewer::Update` method is going to block for a long time, almost every time it is called, waiting for the monitor vsync to arrive.

So how fast *should* our emulator execute? Well, that's tricky to determine exactly. Each opcode has a different execution time in a real computer, though approximate timings are known. We could execute an instruction, lookup how long it *should* have taken on real hardware, and sleep until we're ready to go again. The problem with that is that we don't have access to CPU timings at that resolution. Most of these instructions should take a couple of microseconds, but we can only resonably sleep for **at least** a millisecond on modern systems.

We can, however, amortize it. We know that over the course of a second, an average of 540 operations should execute. This will of course not be the case if every instruction is something complex like a draw, but it will work for real-world programs. We also know that CHIP-8 machines target 60Hz refresh rates. That means that every `540 / 60 = 9` instructions our emulated machine ought to wait until the next vsync.

So we ought to be able to alter our own main loop to execute 9 instructions before every call to `SDLViewer::Update`. We'd then be exploiting our own diplay's refresh rate to correct our emulation timings. Let's give it a shot:

```cpp
// main.cpp

void Run() {
  ...
  while (!quit) {
    for (int i = 0; i < 9; i++) {
      cpu.RunCycle();
    }
    cpu.GetFrame()->CopyToRGB24(rgb24, /*r=*/255, /*g=*/0, /*b=*/0);
    viewer.SetFrameRGB24(rgb24, height);
    auto events = viewer.Update();
    ...
  }
}
```

With that, there's a pretty good chance that your emulator is now executing 540 operations per second, and is displaying graphics at 60Hz!

There is an issue if you try to run this on a machine using a display with a refresh rate of more than 60Hz, our utilization of our own hardware's refresh rate is no longer accurate to the emulated machine, causing the emulation to run too fast. If this is an issue for you, you can emulate the vsync as well. You know that the 9 intructions + the render needs to take `1000/60 = 16.67` ms, so you can execute those operations, measure how long they take, and sleep as long as necessary. Because that sleep won't be the most accurate, you can measure the execution time of 540 operations (60 emulated refresh cycles) as well, and sleep up to the second boundary in order to provide any correction to the vsync sleeps. This is exactly what I do in the [original version of this emulator](https://github.com/rivergillis/chip-8) that I based these articles on. That project also uses a seperate thread for CPU execution, whish is likely unnessary (but cool).

Now that we're executing at 540 instructions per second, supporting the CPU timers is pretty easy. The CHIP-8 features two timers, a delay timer and an audio timer. Both of these timers decrement at 60Hz, and the machine beeps while the audio timer is not zero. We can now simply decrement these timers once every 9 instructions in order to emulate a 60Hz clock:

```cpp
// cpu_chip8.cpp

void CpuChip8::RunCycle() {
  ... 
  // Update timers
  num_cycles_++;
  if (num_cycles_ % 9 == 0) {
    if (delay_timer_ > 0) delay_timer_--;
    if (sound_timer_ > 0) {
      std::cout << "BEEPING" << std::endl;
      sound_timer_--;
    }
  }
}
```

One last thing I have yet to mention is input. This is a simple matter of reading the events returned from `SDLViewer::Update`. Like I mentioned, the CHIP-8 uses a 16-key keypad, and you can map those to whatever keys you'd like on the keyboard. Here's a snippet:

```cpp
// main.cpp

...
auto events = viewer.Update();
uint8_t cpu_keypad[16];
for (const auto& e : events) {
  if (e.type == SDL_KEYDOWN || e.type == SDL_KEYUP) {
    if (e.key.keysym.sym == SDLK_1) {                        
      cpu_keypad[0] = e.type == SDL_KEYDOWN;
    } else if (e.key.keysym.sym == SDLK_2) {                        
      cpu_keypad[1] = e.type == SDL_KEYDOWN;
    } else if (e.key.keysym.sym == SDLK_3) {                        
      cpu_keypad[2] = e.type == SDL_KEYDOWN;
    } else if (e.key.keysym.sym == SDLK_4) {                        
      cpu_keypad[3] = e.type == SDL_KEYDOWN;
    } else if (e.key.keysym.sym == SDLK_q) {                        
      cpu_keypad[4] = e.type == SDL_KEYDOWN;
    }
    ...
  }
}
cpu.SetKeypad(cpu_keypad);

// cpu_chip8.cpp
void CpuChip8::SetKeypad(uint8_t* keys) {
  std::memcpy(keypad_state_, keys, 16);
}
```

Finally, our emulator is complete. We've built a complete CHIP-8 interpreter with support for writing a mono frame buffer, a system for converting that buffer to RGB images that get streamed to the GPU as textures, and have now corrected the timings and hooked up input.

I don't know about you, but I learned quite a bit from building this project. It felt really good to build something from the ground up and finish it, and this project felt especially good because now I can actually play through these ROMs! With this completed, I'm one step closer to understanding the magic behind *Super Mario World*.