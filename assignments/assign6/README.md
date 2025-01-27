---
released: true
permalink: /assignments/assign6/
title: 'Assignment 6: Graphics Library and Console'
toc: true
---
{% comment %}
Task list to copy/paste when creating PR for this assign:

__Before releasing assign6:__
- [ ] Review writeup/starter code (instructor)
- [ ] Review consistency/completeness of grading info published to students relative to grading tests used, consider changes clarity/ease of grading (TA)
- [ ] Followup on issues from previous quarter postmortem ()

__To prep for assign6:__
- [ ]

{% endcomment %}

*Written by Philip Levis, updated by Julie Zelenski*

{% include duedate.html n=6 %}

![Wargames Console](images/wargames.jpg)

A console provides a command-line text interface for entering commands and seeing output. Today we have fancy shell programs that support scripts, process
control, and output redirection. But simpler consoles can be
powerful too. One very famous console is [Joshua in WarGames](https://www.youtube.com/watch?v=ecPeSmF_ikc).

## Goals

In this assignment, you will add graphics capability to your Pi and use it to create a snazzy graphical display for your shell. This will unleash your Raspberry Pi from the shackles of its laptop minder and elevate it into a standalone personal computer running a console that allows the user to enter and execute commands. Neat!

In completing this assignment you will have:

- learned how a framebuffer is used as a bitmap to drive a video display
- implemented simple drawing routines
- unlocked achievement of wizard-level proficiency with C pointers and multi-dimensioned arrays

After finishing this assignment, all that remains for assignment 7 is polish up a few details and improve the performance and your top-to-bottom system will be complete. This is all bare-metal code you wrote yourself -- what a great achievement and sense of satisfaction you have earned with all your hard work!

## Get starter files

To ensure that your courseware files are up to date, do a pull in the `cs107e.github.io` repo. 

```console
$ cd ~/cs107e_home/cs107e.github.io
$ git pull
```

Change to your local assignments repo, switch to the `dev` branch, and update your repo with the starter code by pulling from the remote repo:

```console
$ cd ~/cs107e_home/assignments
$ git checkout dev
$ git pull --allow-unrelated-histories starter-code assign6-starter
```

The final setup step is to change the symbolic link to point `Makefile` to the `assign6/makefile`.

```console
$ cd ~/cs107e_home/assignments
$ ln -sf makefiles/assign6.makefile Makefile
```

You can edit the `MY_MODULES` list in the `assign6.makefile` to choose which modules of yours to build on. (See instructions for [use of MY_MODULES](/assignments/assign3/#mymodules) in assignment 3.) Building on all your past modules is a great way to get a preview of your progress toward achieving the full system bonus.

The source files added to your repo for assignment 6 are:

- `src/lib/fb.c`
  + library module for framebuffer
- `src/lib/gl.c`
  + library module for graphics
- `src/lib/console.c`
    + library module for shell
- `src/tests/test_gl_console.c`
  + test program with your unit tests
- `src/apps/console_shell.c`
  + application program that runs your shell, reading input from the PS2/ keyboard and displaying output to the console. You will use this program unchanged.

In this rest of this writeup, we refer to these files by their basename, e.g. just `fb.c` or `test_gl_console.c`, but keep in mind each file is located in its respective subdirectory depending on the type of source file (application, library, or test).

The `make run` target builds and runs the sample application `build/console_shell.bin`. Use this target as a final step to confirm the full integration of your `fb`, `gl`, and `console` modules.  The `make test` target builds and run the test program `build/test_gl_console.bin`. This test program is where you will add all of your unit tests. You will make heavy use of this target throughout your development. You can run the debugger in simulator mode on the corresponding ELF file, e.g. `arm-none-eabi-gdb build/test_gl_console.elf`.


## Core functionality

### 1) Framebuffer

The base layer of graphics support is implemented in the `fb` module which manages  the framebuffer and communicates with the GPU using our provided `mailbox` module.

Start by reviewing the header file `fb.h` (available in `$CS107E/include` or [browse here](/header)).

+ `void fb_init(unsigned int width, unsigned int height, unsigned int depth_in_bytes, fb_mode_t mode)`
+ `void* fb_get_draw_buffer(void)`
+ `void fb_swap_buffer(void)`
+ simple getters for fb settings: `fb_get_width`, `fb_get_height`, ...

The starter version of `fb_init()` contains the code from lab6 to configure the framebuffer in single-buffered mode. In this mode, there is only one buffer. All drawing takes place in that one buffer and is immediately displayed on screen. The framebuffer's virtual size is configured to be equal to the physical size.

You will extend `fb_init` to support configuring the framebuffer in double-buffered mode. In this mode, the virtual height is set to be twice the physical height; this makes space for two screen-size buffers in the one virtual framebuffer. The lower half corresponds to one buffer and the upper half is the other. One of the buffers will be the one currently displayed on-screen; the other buffer that is currently off-screen will be used as the 'draw' buffer. In double-buffered mode, all drawing is done to the draw buffer and when ready, swapping the two buffers gives a smooth on-screen transition. 

To implement `fb_swap_buffer` you change which half of the virtual framebuffer is on-screen by changing the Y offset from 0 to the physical height (or vice versa). To make this change, set the `y_offset` in the fb struct and use `mailbox_request` to inform the GPU of the change.

There is not that much functionality to the `fb` module; but you do want to stop and test here to ensure that this critical functionality is solid before moving on to the `gl` layer. The `test_gl_console.c` test program shows a simple `test_fb` that configures the framebuffer, set the value of pixels, and exercises double-buffering.  Confirm that all of these operations are working correctly by running the test program and watching the display.

### 2) Graphics primitives
The graphics library layers on the framebuffer and provides higher-level drawing primitives to set and get the color of a pixel, draw filled rectangles, and display text. 

Start by reviewing the header file `gl.h` (available in `$CS107E/include` or [browse here](/header)) to see the documentation of the basic drawing functions:

+ `void gl_init(unsigned int width, unsigned int height, gl_mode_t mode)`
+ `void gl_draw_pixel(unsigned int x, unsigned int y, color_t c)`
+ `color_t gl_read_pixel(unsigned int x, unsigned int y)`
+ `void gl_draw_rect(unsigned int x, unsigned int y, unsigned int w, unsigned int h, color_t c)`
+ `void gl_clear(color_t c)`
+ `color_t gl_color(unsigned char r, unsigned char g, unsigned char b)`
+ simple getters for gl settings: `gl_get_width`, `gl_get_height`, ...

Review the provided `gl_init` code to initialize the framebuffer for 32-bit depth. Each pixel stores a 4-byte BGRA color. 

Start by knocking out the simple getter functions that wrap the underlying `fb`functions to provide a consistent `gl` interface for the client. The graphics routines call into `fb`, but the client doesn't need to know this. The client calls `gl_init` and `gl_draw_...`, without any direct use of `fb`.

Next, move on to implement the `gl_draw_pixel` and `gl_read_pixel` functions. These set and get the color of an individual pixel. It is convenient to treat the framebuffer pixel data as a 2-D array and access pixels by x,y coordinates.  Review the syntax for multi-dimensioned arrays in the framebuffer lecture and the exercises of [Lab 6](/labs/lab6). Take care to compute the location of a pixel's data in the framebuffer based on 
__pitch__, not __width__, because the GPU may have made each row a little wider than requested for reasons of alignment.

When accessing the pixel data, be ever mindful that C provides no bounds-checking on array indexes. If you write to an index outside the bounds of the framebuffer, you step on other memory in active use by the GPU, with various dire consequences to follow. It is imperative that your functions detect and reject an attempt to read or write a pixel location that is out-of-bounds.

Filling a rectangle (`gl_draw_rect`) can be implemented as a nested loop to draw each pixel and clearing the screen (`gl_clear`) fills a screen-sized rectangle.  Be sure to draw only those pixels which lie within the framebuffer bounds. You can enforce clipping by calling `gl_draw_pixel` for each pixel and letting that function sort out whether the pixel is in bounds. This simple approach is easy to get correct, but will be a bit pokey because of the repeated checks.

An alternate approach for clipping would be for `gl_draw_rect` to first intersect the rectangle bounds with the screen bounds and proceed to color only the pixels in the intersection, obviating the need for a repeated per-pixel check for in bounds and drawing large rectangles much faster.

Some testing is now in order! The `test_gl_console.c` test program defines a simple `test_gl` to get you started. The provided tests cases are very rudimentary, you will need to supplement with many additional test cases of your to confirm the full range of functionality for the module. You can rig up an assert-based test that makes a `draw_xxx` call followed by some calls to `read_pixel` to confirm the pixel color at certain locations, but this can get unwieldy at scale. You may find it easier to "eyeball" drawing tests -- draw shapes in fixed location/color and visually confirm the results on the display. Be sure to include test cases that confirm the correct clipping behavior too.

Use your `gl` module to draw something that makes you happy: [SMPTE color bars](https://en.wikipedia.org/wiki/SMPTE_color_bars), the [Mandelbrot set](https://en.wikipedia.org/wiki/Mandelbrot_set), [Sierpinski's carpet](https://en.wikipedia.org/wiki/Sierpi%C5%84ski_carpet), crazy psychedelic patterns, ...

__A note on performance__: Thus far you have likely not given much thought to performance tuning as your programs have run acceptably fast without special effort.  Now that you are writing graphics code, there can be speed bumps that cry out for attention. The bottlenecks will typically show up in the inner loops. With so many pixels to process (e.g., 2 million in a 1920 x 1080 display), streamlining the work per pixel has a big impact. For example, a `gl_clear` that executed 30 instructions per pixel would keep the Pi occupied for a full 3 seconds! Squeezing out 10 instructions of the loop body would cut 20 million instructions overall and save a full second -- big bang for the buck!

For this assignment, we continue to prioritize correct functionality over efficiency, so it's fine to go with a simple-and-slow approach for now. We expect that your drawing will be sluggish and your console will miss keys that are typed while it is in the middle of drawing. You'll fix this up in assignment 7 by employing a mechanism for sharing the CPU during a long-running operation.

### 3) Fonts and text-drawing
The final two functions to implement for the graphics library are:

+ `void gl_draw_char(unsigned int x, unsigned int y, char ch, color_t c)`
+ `void gl_draw_string(unsigned int x, unsigned int y, const char* str, color_t c)`

The last exercise of [lab 6](/labs/lab6) introduced you to the provided `font`
module that manages the font image data.  A font has one combined bitmap consisting of glyphs for all characters, from which it can extract individual character images.

`gl_draw_char` will use `font_get_glyph` to obtain the glyph image and then draws each 'on' pixel.  

`gl_draw_string` is simply a loop that calls `gl_draw_char` for each character, left to right in a single line.

Just as you did previously, ensure that you clip all text drawing to the bounds of the framebuffer.

Edit the test program to draw yourself a congratulatory message and add a variety of tests that exercise text drawing.  As with the earlier drawing tests, you're likely to confirm the results visually. Be sure to have test cases that confirm that drawn characters are correctly clipped to the display bounds.

You're now ready to tackle the console!

### 4) Console
The console module uses the text-drawing functions of the graphics library to drive a monitor as a graphical output device. 

Review the header file `console.h` (available in `$CS107E/include` or [browse here](/header)). The console has these public functions:

+ `void console_init(unsigned int nrows, unsigned int ncols, color_t foreground, color_t background)`
+ `void console_clear(void)`
+ `int console_printf(const char *format, ...)`

The console module is a layer on top of the graphics library, which itself is a layer on top of the framebuffer. The client interfaces with `console` by calling `console_init` and then `console_printf`, without any direct use of `gl` or `fb`.

The console implementation stores the console contents, i.e., rows of text currently displayed, most likely using a 2-D array. The console also tracks the position of the cursor (insertion point). `console_init` initializes the contents to empty, `console_printf` adds text at the cursor position, and `console_clear` resets the contents to empty.

The `console_printf` function should call use your `vsnprintf` to prepare the formatted output. Once prepared, process the characters in the formatted output one-by-one. Each ordinary character is inserted at the cursor position and the cursor advances. There are three special characters that require unique processing:
- `\b` : backspace (move cursor backwards one position)
- `\n` : newline (move cursor down to first column of next row)
- `\f` : form feed (clear contents and move cursor to home position in upper left)

As the console makes changes to the text contents, it must update the display to match. "Printing" to the console is handled by calling on the gl layer to draw text. You can make an incremental update as part of processing each character or postpone into a single refresh after processing a string of characters. 

The console should also handle the operations for:
- Horizontal wrapping: if there are too many characters to fit on the current row,
  automatically wrap the overflow to the next row. It is a nice touch for backspace
  to work correctly on a wrapped row, but we won't test this specific case in grading.
- Vertical scrolling: filling the bottommost row and starting a new one scrolls 
  the text upwards, that is, all rows are shifted up by one. The top row scrolls
  off and the bottommost row now contains the text just added.

Trying to implement all of the console functionality, including special cases, in one go is a tough way to approach. Instead, break the work down into manageable step and test as you go. For example, start by implementing a console of a single row. Add simple test cases to `test_gl_console.c` that call `console_printf()` to output ordinary characters in a single row. Observe the results our display to confirm the display is what you expect. 

Add processing for handling backspace within that single line. Review how your shell uses backspace to delete and re-type characters and be sure that your console correctly handles those edit sequences. Add test cases that call `console_printf` on format strings containing backspaces to confirm.

Once your single row console is working, think about how you want to represent multiple rows. What kind of data structure will you use? How will you process newline and form feed?  Wrapping and scrolling add some unique complications. Think through what needs to happen for each and sketch out a plan before tackling that code. 

Some ways of structuring the data will make the tasks much easier than others. We encourage you to talk with your fellow students to discuss design tradeoffs.  Don't feel bound to one design: if you start implementing your approach and it starts seeming very difficult, with many hard edge cases, you may want to consider a new design. Like much code in this class, 80% of the effort is figuring out exactly what your code should do. Once you have that worked out, it can be direct matter to write it. Throwing away a messy first attempt and using what you learned from it to restart with a clean design is often the best path forward; far better than grinding along with a flawed approach.

You will need to add new test cases for scrolling, line wrap, and form feed in isolation, as well as the conjunction of these special features. There is a lot to test! Although the intention is for console to eventually be the output device for the shell, we recommend putting off that final integration test to the very end.  Plan to do all of your console development and testing in isolation using `console_printf` and unit tests in `test_gl_console.c`.  Trying to debug console from within the context of the full shell program unnecessarily complicates the process and is not recommended.

### 5) Shell + console = magic!

The final step is an easy but very satisfying conclusion: use your console as the output display for your shell.  The `make run` target builds the application program `console_shell.c`. This program calls your `shell_init` passing `console_printf` in place of the uart `printf`.  Simply by changing which function pointer is supplied, the shell you wrote in assignment 5 now springs to life in graphical form! You don't need to write any new code for this, just run the `console_shell` program and enjoy seeing your modules all work together in harmony.

If the console shell feels slow or drops keys as you're typing, don't worry. We'll fix that problem in the next assignment. Why might the console shell be slow to process keys?

The video below demonstrates of our reference console. The shell is running on the Pi, user is typing on the PS/2 keyboard, and the output is displaying on the HDMI monitor.

<iframe width="625" height="400" src="https://www.youtube-nocookie.com/embed/Emfmc0fAJXY?modestbranding=1&version=3&playlist=Emfmc0fAJXY" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Testing and debugging
The `fb` and `gl` modules can be exercised by use of `assert` with `gl_read_pixel` can be used to confirm that expected color at a given pixel location.  You can also run the test program and observe what is displayed to the monitor and visually confirm correctness.

For `console`, almost all of the testing can be done by unit tests that call `console_printf`. Start with simple outputs and work your way up to correct handling of special characters, wrapping, and scrolling. After confirming success with a full battery of unit tests, switch to the `console_shell` program as a final interactive test to see that `console_printf` also works correctly in the context of the graphical shell. You will need to type slowly on your PS/2 keyboard to avoid missed keys.

The primary source of debugging woes on this assignment are due to incorrect access to memory -- uninitialized pointers, indexes out of bounds, wrong level of indirection, incorrect typecast, misunderstanding about units or layout -- there be dragons here!  Be mindful of the differences between a pointer and an array. Know the bounds on your arrays and always respect those bounds. Be conscious of the automatic scaling applied for pointer arithmetic/array access. Keep track of the units a value is expressed in (bits? bytes? pixels?). One area to be especially vigilant is when accessing the framebuffer memory. The neighboring memory to the framebuffer contains data critical to the GPU and should you erroneously corrupt it, this particular transgression can be punished in mysterious ways (screen garbage, a surprise reshowing of the Pi test pattern, crash/lockup of the GPU requiring you to reset your Pi). If you observe a consequence of this ilk, it suggests you need to review how your code accesses the framebuffer memory.

## Extension: Line and triangle drawing
Extend the graphics library so that you can draw anti-aliased lines and triangles. Implement the function prototypes as given in `gl.h`.

This extension is true extra credit and requires you to learn about line drawing algorithms. A good starting point is the 
[Wikipedia entry on line drawing](https://en.wikipedia.org/wiki/Line_drawing_algorithm).

Your line drawing function should draw *anti-aliased* lines:

![Anti-aliased Lines](images/antialiased.png)

Implement triangle drawing by using your line-drawing routine to draw the anti-aliased outline of the triangle and then fill the interior with the user's specified color.

In your `assign6-readme.md`, give a brief explanation of how your line and triangle drawing algorithms operate. 

Please note that pasting-and-modifying code you find online is not in the spirit of doing an extension and misrepresenting the work of others as your own is a violation of the Honor Code. You may read conceptual information and skim pseudocode to develop an understanding of an algorithm, but after reading, you should be able to put aside these references and write the C code yourself based on your own authentic understanding.

## Submit
The deliverables for `assign6-submission` are:

  - Implementation of the `fb.c`, `gl.c` and `console.c` library modules
  - Your comprehensive tests in `tests_gl_console.c`
  - `assign6-readme.md` (possibly empty)

Submit the finished version of your assignment by tagging `assign6-submission` and making a pull request. The steps to follow are given in the [Assignment 0 writeup](/assignments/assign0/#submit).


## Grading
To grade this assignment, we will:

- Verify that your submission builds correctly, with no warnings. Warnings and/or
  build errors result in automatic deductions. Clean build always!
- Run automated tests that exercise the functionality of your `fb`, 
  `gl`and `console` modules.
- Go over the tests you added to `test_gl_console.c` and evaluate them for
  thoughtfulness and completeness in coverage.
- Review your code and provide feedback on your design and style choices.

Our highest priority tests will focus on the core features for this assignment:

- Essential functionality of your library modules
  - `fb`
    - correct configuration of framebuffer
    - double buffering
  - `gl`
    - drawing pixels, rects, characters
    - correct clipping of all drawing
  - `console`
      - display ordinary characters
      - handling of special chars (\n \b \f)
      - horizontal line wrap
      - vertical scrolling

The additional tests of lower priority will examine less critical features, edge cases, and robustness. Make sure you thoroughly tested for a variety of scenarios!
