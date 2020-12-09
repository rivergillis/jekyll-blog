---
layout: post
title: "Emulating a Computer: Graphics and Texture Streaming"
tags: [programming, emulation, games]
date: 2020-12-07 19:51 -0800
description: Displaying graphics using SDL for our emulator.
---

{% include video_embed.html file='ufo' %}

When [last we left off]({% post_url 2020-12-06-emulating-a-computer-part-2 %}) we developed ourselves a chip-8 interpreter with working frame drawing. Now, we want to get those images out of our consoles and onto our screens.

We're going to accomplish this using [SDL](https://www.libsdl.org/), a library to blit graphics to the screen, capture user input, and play sounds. Setting up an SDL project can be a little tricky, so I recommended reading [my post on it]({% post_url 2020-11-29-setting-up-a-cross-platform-sdl-project %}) in order to get things building correctly.

There are many ways to get graphics onto the screen using SDL. For most games, complete images are not made on the CPU like we're doing. However for emulation and (more commonly) video playback, a (potentially compressed) image sits on the CPU and needs to be uploaded to the GPU in order to be displayed. Once the image is on the GPU, we call it a "texture", and we call this whole process "texture streaming".

Now we have a particular image format that `Image` uses, but SDL provides a [list of pixel formats](https://wiki.libsdl.org/SDL_PixelFormatEnum) that it understands. `SDL_PIXELFORMAT_RGB24` seems like a fine enough format. Lets set up an `SDLViewer` class that will stream images of this format, and figure out later how to convert our frame buffer to `RGB24`.

```cpp
// sdl_viewer.h

// RAII hardware-accelerated SDL Window.
// Optimized for RGB24 texture streaming.

class SDLViewer {
  public:
    // Width and height must be equal to the size of images uploaded
    // via SetFrameRGB24.
    SDLViewer(const std::string& title, int width, int height, int window_scale = 1);
    ~SDLViewer();

    // Renders the current frame, returns a list of all events.
    std::vector<SDL_Event> Update();

    // Assumes 8-bit RGB image with stride equal to width (no padding).
    void SetFrameRGB24(uint8_t* rgb24, int height);

  private:
    SDL_Window* window_ = nullptr;
    SDL_Renderer* renderer_ = nullptr;
    SDL_Texture* window_tex_ = nullptr;
};
```

We plan on using this class as an RAII SDL window that receives texture updates, and performs rendering. Our constructor will take a window scale because if we tried to draw a 64x32 image to the screen it would be very small.

```cpp
SDLViewer::SDLViewer(const std::string& title, int width, int height, int window_scale) : 
      title_(title) {
  if(SDL_Init(SDL_INIT_VIDEO) < 0) {
    throw std::runtime_error(SDL_GetError());
  }
  // Create the SDL window with scaled dimensions.
  window_ = SDL_CreateWindow(title.c_str(), SDL_WINDOWPOS_UNDEFINED,
      SDL_WINDOWPOS_UNDEFINED, width * window_scale, height * window_scale, SDL_WINDOW_SHOWN);
  // Set up the hardware renderer and the texture we'll stream to.
  renderer_ = SDL_CreateRenderer(window_, -1, SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC);
  SDL_SetRenderDrawColor(renderer_, 0xFF, 0xFF, 0xFF, 0xFF);

  window_tex_ = SDL_CreateTexture(renderer_, SDL_PIXELFORMAT_RGB24,
    SDL_TEXTUREACCESS_STREAMING, width, height);
}

SDLViewer::~SDLViewer() {
  SDL_DestroyTexture(window_tex_);
  SDL_DestroyRenderer(renderer_);
  SDL_DestroyWindow(window_);
  SDL_Quit();
}


std::vector<SDL_Event> SDLViewer::Update() {
  std::vector<SDL_Event> events;
  SDL_Event e;
  while (SDL_PollEvent(&e)) { events.push_back(e); }

  // Render our texture to the screen.
  SDL_RenderCopy(renderer_, window_tex_, NULL, NULL );
  SDL_RenderPresent(renderer_);

  return events;
}

void SDLViewer::SetFrameRGB24(uint8_t* rgb24, int height) {
  void* pixeldata;
  int pitch;
  // Lock the texture and upload the image to the GPU.
  SDL_LockTexture(window_tex_, nullptr, &pixeldata, &pitch);
  std::memcpy(pixeldata, rgb24, pitch * height);
  SDL_UnlockTexture(window_tex_);
}
```

We need to perform some boilerplate SDL initialization and destruction in the constructor and destructor. The `Update` method will present whatever the last image we sent to `SDLViewer` was. It also performs the important task of receiving input events.

`SetFrameRGB24` is where we upload the texture to the GPU. It receives some image memory in the correct format along with a height. `SDL_LockTexture` returns some CPU memory that we need to copy to. It also returns the pitch, or row byte length, of the image. Once we copy the image to the alloted chunk we call `SDL_UnlockTexture` which will upload that image to the GPU as the new texture memory.

Now we ought to update our main loop to make use of this new window.

```cpp
// main.cpp

void Run() {
  int width = 64;
  int height = 32;

  SDLViewer viewer("CHIP-8 Emulator",  width, height, /*window_scale=*/8);
   uint8_t* rgb24 = static_cast<uint8_t*>(std::calloc(
      width * height * 3, sizeof(uint8_t)));
  viewer.SetFrameRGB24(rgb24, height);

  CpuChip8 cpu;
  cpu.Initialize("/path/to/program/file");
  bool quit = false;
  while (!quit) {
    cpu.RunCycle();
    cpu.GetFrame()->CopyToRGB24(rgb24, /*r=*/255, /*g=*/0, /*b=*/0);
    viewer.SetFrameRGB24(rgb24, height);
    auto events = viewer.Update();

    for (const auto& e : events) {
      if (e.type == SDL_QUIT) {
        quit = true;
      }
    }
  }
}
```

We initialize our `RGB24` image to a blank (zero/black) image. Note the size is not width * height, but width * height * 3. Since this is an RGB image, we have 3 channels. We upload the texture every cycle and present it the screen. With vsync this will cause the emulator to run very slowly, but this will be fixed when we go over timing. So now we just need to figure out what this `RGB24` image format is and implement `Image::CopyToRGB24`.

When we make RGB images, we often interleave the red, green, and blue channels within memory. That is, adding 1 to a memory location no longer necessarily leads to the next pixel value.

```
0x000  :|RGBRGBRGB...----------------------------------------|
0x040*3:|RGBRGBRGB...                                        |
0x080*3:|RGBRGBRGB...                                        |
        ..
0x7C0*3:|RGBRGBRGB...----------------------------------------|
```

We now need some new terminology to discuss this. "Stride"  (or "pitch") often refers to the byte width of a row in an image. In this case, our RGB stride is 3 * width_px. We can also talk about stride as it relates to channels. To move from one red pixel to the next red pixel (the 0-dimension stride), we add 3. This is true of blue and green channels as well. Each individual value is still 8 bits (0 to 255) but a pixel as a whole now requires 3 values (the "24" in `RGB24` is thus 3 channels * 8 bits). Okay, we should now have all we need to generate this image from our current mono image.

```cpp
// image.cpp

 void Image::CopyToRGB24(uint8_t* dst, int red_scale, int green_scale, int blue_scale) {
  int cols = Cols();
  for (int row = 0; row < Rows(); row++) {
    for (int col = 0; col < cols; col++) {
      dst[(row * cols + col) * 3] = At(col, row) * red_scale;
      dst[(row * cols + col) * 3 + 1] = At(col, row) * green_scale;
      dst[(row * cols + col) * 3 + 2] = At(col, row) * blue_scale;
    }
  }
 }
```

We basically just iterate over the entire image and copy it over to `dst`, though for every pixel in the original image we need to copy three bytes over to the new image, one for each channel. Here we can take advantage of the fact that our original image values are either 0 or 1 to provide some color to the new image using some scaling parameters.

And just like that, we've got emulator output onto the screen! We're even utilizing the GPU! Your program output should start looking at lot like the videos at the top of these articles, though likely a bit slower. In [next article]({% post_url 2020-12-08-emulating-a-computer-part-4 %}) we'll dive into timing in order to get everything running at the right speed.