---
layout: post
title: "Emulating a Computer: The CHIP-8 Interpreter"
tags: [programming, emulation, games]
date: 2020-12-04 16:42 -0800
description: How to write a CHIP-8 emulator.
---

{% include mp4_embed.html file='tetris.mp4' %}

For several reasons, emulation has always fascinated me. A program that executes other programs sounds like such a cool concept. It really feels like you're getting your money's worth out of writing it! Beyond that, it definitely feels like you're *building* a computer within software. I really enjoyed learning about computer architecture and writing some basic HDL code, but emulation is a much more straightforward way of achieving a similar feeling of generating a machine. I've also always had this goal of knowing exactly how *Super Mario World* worked, ever since I first saw it as a kid. Because of this, writing a SNES/SFC emulator has been on my mind for a while. I decided recently that it was time to take a [step forward](https://github.com/rivergillis/chip-8) towards making this happen.

So let's take a look at writing an emulator. A simple, but complete example would involve CHIP-8.

[CHIP-8](https://en.wikipedia.org/wiki/CHIP-8) is actually a programming language. It's really simple too, there's only [35 opcodes](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM#3.0). To write an interpreter for it, we pretty much just need to write a program that can execute all 35 different instructions. The *emulation* aspect of this comes from the bits you wouldn't normally find in a programming language interpreter. We need a way to display graphics, process user input, play audio, and we need to simulate the hardware mechanisms of a CHIP-8 machine. Things like registers and memory need to be taken into account during execution, and we also need to be careful about timing. 

Let's start! For this project, we'll be using C++. This should be fairly trivial to translate into other languages. If you want to take a look at the complete source, see the [project repository](https://github.com/rivergillis/chip-8).

First, a basic main loop. We'll ignore emulating timing for now.

```cpp
// main.cpp

void Run() {
  CpuChip8 cpu;
  cpu.Initialize("/path/to/program/file");
  bool quit = false;
  while (!quit) {
    cpu.RunCycle();
  }
}

int main(int argc, char** argv) {
  try {
    Run();
  } catch (const std::exception& e) {
    std::cerr << "ERROR: " << e.what();
    return 1;
  }
}
```

Our `CpuChip8` class will encapsulate the state of our virtual machine and interpreter. Now if we implement `RunCycle` and `Initialize` we'll have ourselves a basic emulator skeleton. We now need to discuss the phsyical system we're emulating.

Our CHIP-8 system will be the [Telmac 1800](https://en.wikipedia.org/wiki/Telmac_1800). We've got ourselves a pool of 4K of memory, a 64x32 1-bit display, and the ability to beep. *Nice*. The CHIP-8 interpreter itself is implemented via a virtual machine. We need to keep track of a stack, sixteen 8-bit registers (named V0 through VF), a 12-bit index register (named I), a program counter, two 8-bit timers, and a 16-frame stack.

The canonical memory map looks like this:
```
0x000 |--------------------|
      | Interpreter memory |
      |                    |
0x050 | Built-in fontset   |
0x200 |--------------------|
      |                    |
      |                    |
      | Program memory     |
      | and dynamic allocs |
      |                    |
      |                    |
0xFFF |--------------------|
```

You'll notice there's no explicit stack here. The program actually doesn't have a stack to address, that's only used by the interpreter to implement jumping to functions and back. With this in mind we can draw up a header.

```cpp
// cpu_chip8.h

class CpuChip8 {
 public:
  public Initialize(const std::string& rom);
  void RunCycle();

 private:
  // Fills out instructions_.
  void BuildInstructionSet();

  using Instruction = std::function<void(void)>;
  std::unordered_map<uint16_t, Instruction>> instructions_;

  uint16_t current_opcode_; 

  uint8_t memory_[4096];  // 4K
  uint8_t v_register_[16];

  uint16_t index_register_;
  // Points to the next instruction in memory_ to execute.
  uint16_t program_counter_;

  // 60Hz timers.
  uint8_t delay_timer_;
  uint8_t sound_timer_;

  uint16_t stack_[16];
  // Points to the next empty spot in stack_.
  uint16_t stack_pointer_;

  // 0 when not pressed.
  uint8_t keypad_state_[16];
};
```

We use excplicit integer types to ensure values are over/underflowed correctly. We need to use 16-bit types for 12-bit values. We also have 16 digital input keys, which we store as either on or off within this class. When we hook up input, we'll find a way to feed that into the class between cycles. The opcode is made easy by the fact that all CHIP-8 instructions are 2 bytes long.

So that gives us 0xFFFF=64k possible instructions (though many are unused). We can actually store every possible instruction in a map so that when we fetch an opcode we are able to immediately execute it by calling the associated `Instruction` in `instructions_`. Since we don't bind much data to the functions, which should be able to fit the entire instruction map in cache!

Our `Initialize` function is where to set up the memory map described above:

```cpp
// cpu_chip8.cpp

CpuChip8::Initialize(const std::string& rom) {
  current_opcode_ = 0;
  std::memset(memory_, 0, 4096);
  std::memset(v_registers_, 0, 16);
  index_register_ = 0;
  // Program memory begins at 0x200.
  program_counter_ = 0x200; 
  delay_timer_ = 0;
  sound_timer_ = 0;
  std::memset(stack_, 0, 16);
  stack_pointer_ = 0;
  std::memset(keypad_state_, 0, 16);
  
  uint8_t chip8_fontset[80] =
  { 
    0xF0, 0x90, 0x90, 0x90, 0xF0, // 0
    0x20, 0x60, 0x20, 0x20, 0x70, // 1
    0xF0, 0x10, 0xF0, 0x80, 0xF0, // 2
    0xF0, 0x10, 0xF0, 0x10, 0xF0, // 3
    0x90, 0x90, 0xF0, 0x10, 0x10, // 4
    0xF0, 0x80, 0xF0, 0x10, 0xF0, // 5
    0xF0, 0x80, 0xF0, 0x90, 0xF0, // 6
    0xF0, 0x10, 0x20, 0x40, 0x40, // 7
    0xF0, 0x90, 0xF0, 0x90, 0xF0, // 8
    0xF0, 0x90, 0xF0, 0x10, 0xF0, // 9
    0xF0, 0x90, 0xF0, 0x90, 0x90, // A
    0xE0, 0x90, 0xE0, 0x90, 0xE0, // B
    0xF0, 0x80, 0x80, 0x80, 0xF0, // C
    0xE0, 0x90, 0x90, 0x90, 0xE0, // D
    0xF0, 0x80, 0xF0, 0x80, 0xF0, // E
    0xF0, 0x80, 0xF0, 0x80, 0x80  // F
  };
  // Load the built-in fontset into 0x050-0x0A0
  std::memcpy(memory_ + 0x50, chip8_fontset, 80);

  // Load the ROM into program memory.
  std::ifstream input(filename, std::ios::in | std::ios::binary);
  std::vector<uint8_t> bytes(
         (std::istreambuf_iterator<char>(input)),
         (std::istreambuf_iterator<char>()));
  if (bytes.size() > kMaxROMSize) {
    throw std::runtime_error("File size is bigger than max rom size.");
  } else if (bytes.size() <= 0) {
    throw std::runtime_error("No file or empty file.");
  }
  std::memcpy(memory_ + 0x200, bytes.data(), bytes.size());

  BuildInstructionSet();
}
```

Don't worry about trying to read that file loading code, the C++ iostream library is kind of ridiculous. The gist of it here is that we set everything to 0 and load things into memory that need to be loaded. The fontset here is a series of 16 built-in sprites that programs can reference as they want. We'll go over how that memory forms sprites later on when we worry about graphics. Our goal is that once `Initialize` complete we're set up to execute a user program.

Let's build out a basic `RunCycle` so that we have a better idea of out to write `BuildInstructionSet`. If you remember any basic computer architecture, a cycle has a few phases. First you fetch the instruction, then you decode it, then you execute it.

```cpp
// cpu_chip8.cpp

void CpuChip8::RunCycle() {
  // Read in the big-endian opcode word.
  current_opcode_ = memory_[program_counter_] << 8 |
    memory_[program_counter_ + 1];

  auto instr = instructions_.find(current_opcode_);
  if (instr != instructions_.end()) {
    instr->second();
  } else {
    throw std::runtime_error("Couldn't find instruction for opcode " +
      std::to_string(current_opcode_));
  }

  // TODO: Update sound and delay timers. 
}
```

This is pretty much just a map lookup to find the function to execute. The one weird bit here is how we read in the next opcode. CHIP-8 uses a big-endian archicture, which means the most-significant part of the word comes first, followed by the least significant part of the word. This is reversed in modern x86-based systems.

```
Memory location 0x000: 0xFF 
Memory location 0x001: 0xAB

Big endian interpretation:    0xFFAB
Little endian interpretation: 0xABFF
```

Note that we don't alter the program counter within `RunCycle`. This is done on a function-by-function base, so we leave that to the implementation of the particular `Instruction`. Also, since we chose to define `Instruction` as a function pointer without any arguments, we're going to have to bind those to the function itself. This requires more work in the initial set-up, but means we completely remove the instruction-decode phase on `RunCycle`.

Let's dig into the meat of the interpreter, `BuildInstructionSet`. I wont list the implementations for every function here, but you can find that in the [repository for this project](https://github.com/rivergillis/chip-8). I highly recommend coding this alongside something like [Cowgod's technical reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM#3.1).

```cpp
// cpu_chip8.cpp

#define NEXT program_counter_ += 2
#define SKIP program_counter_ += 4

void CpuChip8::BuildInstructionSet() {
  instructions_.clear();
  instructions_.reserve(0xFFFF);

  instructions_[0x00E0] = [this]() { frame_.SetAll(0); NEXT; }; // CLS
  instructions_[0x00EE] = [this]() {
    program_counter_ = stack_[--stack_pointer_] + 2;  // RET
  };

  for (int opcode = 0x1000; opcode < 0xFFFF; opcode++) {
    uint16_t nnn =  opcode & 0x0FFF;
    uint8_t kk =    opcode & 0x00FF;
    uint8_t x =     (opcode & 0x0F00) >> 8;
    uint8_t y =     (opcode & 0x00F0) >> 4;
    uint8_t n =     opcode & 0x000F;
    if ((opcode & 0xF000) == 0x1000) {
      instructions_[opcode] = GenJP(nnn);
    } else if ((opcode & 0xF000) == 0x2000)) {
      instructions_[opcode] = GenCALL(nnn);
    }
    // ...
}
```

Each instruction may encode some parameters, which we decode and use when needed. We could use `std::bind` here to generate the `std::function`s, in this case I chose to define `Gen[INSTRUCTION_NAME]` functions which will return the functions as lambdas with all of the data bound. Lets look at some of the more interesting functions:

```cpp
// cpu_chip8.cpp

CpuChip8::Instruction CpuChip8::GenJP(uint16_t addr) {
  return [this, addr]() {  program_counter_ = addr; };
}
```

When we JP to an address, we just set the program counter to the address. That'll cause the next cycle to execute the instruction at that point.

```cpp
// cpu_chip8.cpp

CpuChip8::Instruction CpuChip8::GenCALL(uint16_t addr) {
  return [this, addr]() {
    stack_[stack_pointer_++] = program_counter_;
    program_counter_ = addr;
  };
}
```

We do the same thing when we CALL a function at an address. Here, however, we need to provide a way to later return from the callsite. To do this, we store the current program counter onto the stack.

```cpp
// cpu_chip8.cpp

CpuChip8::Instruction CpuChip8::GenSE(uint8_t reg, uint8_t val) {
  return [this, reg, val]() {
    v_registers_[reg] == val ? SKIP : NEXT;
  };
}
```

SE mean "skip if the immediate value is equal to the value in the provided register". The instruction receives the V register to dereference, and we set the program counter accordingly.

```cpp
// cpu_chip8.cpp

CpuChip8::Instruction CpuChip8::GenADD(uint8_t reg_x, uint8_t reg_y) {
  return [this, reg_x, reg_y]() {
    uint16_t res = v_registers_[reg_x] += v_registers_[reg_y];
    v_registers_[0xF] = res > 0xFF; // set carry
    v_registers_[reg_x] = res;
    NEXT;
  };
}
CpuChip8::Instruction CpuChip8::GenSUB(uint8_t reg_x, uint8_t reg_y) {
  return [this, reg_x, reg_y]() {
    v_registers_[0xF] = v_registers_[reg_x] > v_registers_[reg_y]; // set not borrow
    v_registers_[reg_x] -= v_registers_[reg_y];
    NEXT;
  };
}
```

When adding or subtracting registers, we need to keep track of overflow. If we detect it, we set VF.

```cpp
// cpu_chip8.cpp

CpuChip8::Instruction CpuChip8::GenLDSPRITE(uint8_t reg) {
  return [this, reg]() {
    uint8_t digit = v_registers_[reg];
    index_register_ = 0x50 + (5 * digit);
    NEXT;
  };
}
```

Our sprite loading function is fairly trivial, it is used by the program to figure out where a certain digit is within the built-in fontset. Remember we stored our fontset at `0x50` and each character is 5-bytes wide. So we set `I` to `0x50 + 5 * digit`.

```cpp
// cpu_chip8.cpp

CpuChip8::Instruction CpuChip8::GenSTREG(uint8_t reg) {
  return [this, reg]() {
    for (uint8_t v = 0; v <= reg; v++) {
      memory_[index_register_ + v] = v_registers_[v];
    }
    NEXT;
  };
}
CpuChip8::Instruction CpuChip8::GenLDREG(uint8_t reg) {
  return [this, reg]() {
    for (uint8_t v = 0; v <= reg; v++) {
      v_registers_[v] = memory_[index_register_ + v];
    }
    NEXT;
  };
}
```

When we interface directly with memory, the user provides the maximum register they'd like to use. For instance if they want to load registers V0, V1, V2 with the values stored sequentially in `MEM[I]` they'd pass in V2 after setting up `I`.

With that, we've got ourselves a CHIP-8 interpreter! Sure, there's no sound or graphics hooked up, but as long as you don't use those functions you should be able to execute some basic test ROMs. [In the next part of this series]({% post_url 2020-12-06-emulating-a-computer-part-2 %}), we'll look at drawing, the most complex operation that the interpreter performs.