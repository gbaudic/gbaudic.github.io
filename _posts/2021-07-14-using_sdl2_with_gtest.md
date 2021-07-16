---
layout: post
title: "Using SDL2 with gtest"
date: 2021-07-14 14:00:00 +0200
tags: SDL2 gtest tips-and-tricks c++ development
---

# Using SDL2 with the Google Test framework (gtest)

I have been spending time on a game project in C++. Being a game, this software builds upon the well-known [SDL2](https://libsdl.org) framework. But being a good software developer, I also wanted to make sure I could cover the software with some unit testing, so I chose to use Google Test as well.

To make things more interesting, the project targets both Windows and Linux. Turns out, both libraries support both. However, things started getting really interesting when I tried to assemble together in the same binary SDL2 and gtest for the Windows platform. Why? Because on this platform, SDL2 mandates the linkage with ``SDL2_main``, which acts as a wrapper around the actual ``main`` function.  But gtest also uses some kind of wrapper around the main function, and also imposes some constraints on it. Therefore, satisfying the requirements of both requires some care. I was unable to find any recommendation or example for this particular case, so this is my proposal. 

The solution I came up with was to keep using the linker flags for both libraries, and a small trick offered by the SDL library in the code. 

In my [CMakeLists.txt](https://github.com/gbaudic/linbound2/blob/master/CMakeLists.txt) file I put

    target_link_libraries(${PROJECT_TEST_NAME} ${GTEST_MAIN} ${SDL2_LIBRARIES}
            ${SDL2_IMAGE_LIBRARIES} 
            ${SDL2_TTF_LIBRARIES}
            ${SDL2_GFX_LIBRARIES}
            ${SDL2_MIXER_LIBRARIES}
            ${SDL2_NET_LIBRARIES}
            ${GUISAN_LIBRARIES}
            ${TINYXML2_LIBRARIES}
            ${SQLITE3_LIBRARIES}
            ${BOX2D_LIBRARIES} ${GTEST})

As you can see, I kept only the main for GTest here.  
In the [main file](https://github.com/gbaudic/linbound2/blob/master/test/testSettings.cpp) for my test executable, I added both headers

	#include <SDL2/SDL.h>
	#include "gtest/gtest.h"

and just before, this small trick hidden deep in SDL2 documentation to disable the duplicated ``main`` wrapper.

	#define SDL_MAIN_HANDLED

Finally, the ``main`` function:

    int main(int argc, char *argv[]) {
      // Also needed for my dual main problem
      SDL_SetMainReady();

      // Initializing SDL (with no audio because it will not work on Travis)
      // I have some tests working with SDL surfaces and textures, so I need this subsystem
      if (SDL_Init(SDL_INIT_VIDEO) != 0) {
          std::cout << "FATAL: Cannot init SDL: " << SDL_GetError();
          return -1;
      }

      ::testing::InitGoogleTest(&argc, argv);
      int result = RUN_ALL_TESTS();  // Store the results in a variable

      SDL_Quit();  // Quit SDL properly

      return result;  // Return the result, as required by google test
    }
