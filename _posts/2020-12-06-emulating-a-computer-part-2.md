---
layout: post
title: "Emulating a Computer: Images and Rendering"
tags: [programming, emulation, games]
date: 2020-12-06 18:21 -0800
description: Adding images to our CHIP-8 emulator.
---

{% include mp4_embed.html file='invaders.mp4' %}

In the [last part of this series]({% post_url 2020-12-04-emulating-a-computer-part-1 %}) we built a CHIP-8 interpreter to execute all opcodes except for one, `Dxyn - DRW Vx, Vy, nibble`. To make this process easier, we'll encapsulate our image memory and operations in an `Image` class. Our 64x32 frame will be represented as a single chunk of data in memory, where each pixel is a single byte:

```
0x000:|--------------------------------------------------------------|
0x040:|                                                              |
0x080:|                                                              |
0x0C0:|                                                              |
      ...
0x7C0:|--------------------------------------------------------------|
```

For our image above, we need to store three pieces of data: the number of rows, number of cols, and the starting address of the image memory (which `malloc` gives us). With this address pointing to the top-left corner of the memory, addressing individual pixels is fairly trivial. Some examples:

```
img[col=0, row=0] = img[0]
img[col=0, row=1] = img[width]
img[col=1, row=3] = img[3*width+1]
```

Okay, we should be ready to write the header.

```cpp
// image.h

class Image {
  public:
    // Allocs and de-allocs in ctor and dtor.
    Image(int cols, int rows);
    ~Image();

    uint8_t* Row(int r);

    // Returns a pixel that can be changed.
    uint8_t& At(int c, int r);

    void SetAll(uint8_t value);

  private:
    int cols_;
    int rows_;

    uint8_t* data_;
};
```

What we need to pay attention to is that we're dynamically allocating data that will be owned by this class. In a larger system, we may choose to use `std::unique_ptr` in conjuction with a specified allocation function, but here we'll just `malloc` at construction time and `free` at destruction time.


```cpp
// image.cpp

Image::Image(int cols, int rows) {
  data_ = static_cast<uint8_t*>(malloc(cols * rows * sizeof(uint8_t)));
  cols_ = cols;
  rows_ = rows;
}
Image::~Image() {
  free(data_);
}

uint8_t* Image::Row(int r) {
  return &data_[r * cols_];
}

uint8_t& Image::At(int c, int r) {
  return Row(r)[c];
}

void Image::SetAll(uint8_t value) {
  std::memset(data_, value, rows_ * cols_);
}

void Image::DrawToStdout() {
  for (int r = 0; r < rows_; r++) {
    for (int c = 0; c < cols_; c++) {
      if (At(c,r) > 0) {
        std::cout << "X";
      } else {
        std::cout << " ";
      }
    }
    std::cout << std::endl;
  }
  std::cout << std::endl;
}
```

I like to use rows and columns here rather than x and y because I rarely forgot what a row is, but I often forget what an "x" is.  Our `At` function returns a `uint8_t&` so that we can use it for both getting and setting individual pixels. This is bad for encapsulation, but is common among image APIs. We also added a handy `DrawToStdout` so that we can view the image in our console until we set up graphical output later. Now we can add our frame to the CPU and work out an implementation.

```cpp
// cpu_chip8.h

class CpuChip8 {
 public:
  constexpr innt kFrameWidth = 64;
  constexpr innt kFrameHeight = 32;

  CpuChip8() : frame_(kFrameWidth, kFrameHeight) {}
  ...
 private:
  ...
  Image frame_;
};
```

Finally we'll talk about how CHIP-8 performs drawing. All drawing is done in terms of an XOR operation of a sprite onto the current and only frame buffer. All sprites are defined as images with a depth of 1-bit (each pixel on or off), a width of 8, and a variable height. The width limitation is due to the fact that each pixel in the sprite is only a single bit. Let's take a look at the fontset earlier and try to decode a digit.

```
0xF0, 0x90, 0x90, 0x90, 0xF0, // 0

0xF0 is 1111 0000 -> XXXX
0x90 is 1001 0000 -> X  X
0x90 is 1001 0000 -> X  X
0x90 is 1001 0000 -> X  X
0xF0 is 1111 0000 -> XXXX
```

Pretty cool! So when I said that drawing is done as an XOR operation, the only way to remove a sprite from the screen is to draw over it (1 ⊕ 1 is 0). This is why you often see flickering in CHIP-8 programs, the sprite is continuously drawing and un-drawing to depict movement.

Okay we're just about ready to write a sprite drawing function. We're going to need a starting point and a sprite (a chunk of memory). Since sprite height is variable, we'll receive that parameter too. One thing that took me quite a long time to debug is that the CHIP-8 interperter supports drawing out of bounds. When that happens, the draw continues as it wraps around. This also works for the starting coordinates of the (so drawing at 255,255 for 15 rows is perfectly valid). Also, the interpreter needs to receive back whether or not a pixel was turned off (this is often used for collision detection).

```cpp
// image.cpp

// Returns true if the new value unsets the pixel.
bool Image::XOR(int c, int r, uint8_t val) {
  uint8_t& current_val = At(c, r);
  uint8_t prev_val = current_val;
  current_val ^= val;
  return current_val == 0 && prev_val > 0;
}

bool Image::XORSprite(int c, int r, int height, uint8_t* sprite) {
  // Wrap around the screen as we draw.
  bool pixel_was_disabled = false;
  for (int y = 0; y < height; y++) {
    int current_r = r + y;
    while (current_r >= rows_) { current_r -= rows_; }
    uint8_t sprite_byte = sprite[y];
    for (int x = 0; x < 8; x++) {
      int current_c = c + x;
      while (current_c >= cols_) { current_c -= cols_; }
      // Note: We scan from MSbit to LSbit
      uint8_t sprite_val = (sprite_byte & (0x80 >> x)) >> (7-x);
      pixel_was_disabled |= XOR(current_c, current_r, sprite_val);
    }
  }
  return pixel_was_disabled;
}
```

We need to take care to extract the bits as either 1 or 0. Since our `Image` class supports \[0..255] our XORs could get messy without this restriction. With this in place our CPU instruction is pretty easy, we just need to extract the parameters needed to call `XORSprite`.

```cpp
// cpu_chip8.cpp

CpuChip8::Instruction CpuChip8::GenDRAW(uint8_t reg_x, uint8_t reg_y, uint8_t n_rows) {
  return [this, reg_x, reg_y, n_rows]() {
    uint8_t x_coord = v_registers_[reg_x];
    uint8_t y_coord = v_registers_[reg_y];
    bool pixels_unset = frame_.XORSprite(x_coord, y_coord, n_rows,
      memory_ + index_register_);
    v_registers_[0xF] = pixels_unset;
    NEXT;
  };
}
```

At this stage you should be able to execute some ROMs! Set up a call to `DrawToStdout` after a cycle execution and watch the output in your console. You'll have to run a program that doesn't expect any user input, however.

In the next part of this series, we'll hook up SDL to get some actual graphics onto the screen. We'll also finally wire up the input!