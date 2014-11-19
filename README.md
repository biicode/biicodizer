biicodizer
==========

A biicode blocks generator. It is a simple-to-use python script to manage the adaptation process, given a block description file written by the developer. 

[biicode](http://www.biicode.com/) is a dependency manager for C and C++. It's CMake based, so taking an existing library and building up a biicode block to deploy the library is quite simple.

When to use biicodizer?
-------------------------

biicodizer is great when you have libraries you want to automatically publish to biicode. Follows automated steps to adapt existing libraries to biicode. Moves source files into the block and updates the `#include` directives to match current locations. 

Just focus on your `CMakeLists.txt`. Use CMake to make libraries portable: tune your sources and `CMakeLists.txt`. Once working, share and use the library across multiple platforms with biicode. As simple as: `#include <your_biicode_user/your_library/header.hpp>`.

* When you know the steps to adapt the library
* When following up on a previous lib version


When not to use biicodizer?
----------------------------
biicodizer possibilities seem endless, but there are some scenarios where using this feature at first doesn't make sense. 

* When a library requires a full personalized adaptation.
* When there's too much stuff to manage. (as 3rd party libs)


Anatomy of a biicode block
--------------------------

A biicode block contains all the sources plus some configuration files, such as the `CMakeLists.txt` file or biicode's setting-files located under the `bii/` folder, biicode needs those to correctly build and link a block within other blocks. 

This is the typical folder structure of a biicode block:

```
+-- user/block
|    +-- bii
|    |    +-- paths.bii
|    |    +-- parents.bii
|    |    etc
|    |
|    +-- foo.hpp
|    +-- foo.cpp   
|    +-- CMakelists.txt
|    |
|    +-- README.md
|    +-- LICENSE     
```

Note how sources are placed inside the block root directory, instead of the common `include/` and `src/` folders you may have in your codebase. That's to follow the `#include <user/block/foo.hpp>` convention, instead of the "uglier" (this can be subjective...) `#include <user/block/include/foo.hpp>`.

There's usually a `LICENSE` file, and a `README.md` file, both at the block root directory.

The biicodization process
-------------------------

Here's a C++ library called *"stackie"* which provides the most basic data structures *alla* Standard Library, such as a linked list, an array, a stack, etc. This is its codebase:

```
+-- stackie
|    +-- include
|    |    +-- stack.hpp
|    |    +-- array.hpp
|    |    +-- list.hpp
|    |    +-- vector.hpp
|    +-- src
|    |    +-- stack.cpp
|    |    +-- array.cpp
|    |    +-- list.cpp
|    |    +-- vector.cpp
|    +-- test
|    |    +-- test.cpp
|    |
|    +-- README.md
|    +-- LICENSE
```

The headers, the sources, and a simple unit testing file. 

Traditional way to use **stackie** involves: download appropriate version, include the headers, and then compile and link all the `.cpp` files:

``` shell
$ ls
  use_stackie.cpp
$ git clone https://github.com/developer/stackie.git
  Resolving deltas, blah, blah...
$ ls 
$ g++ ...
$ g++ ...
$ g++ ...
$ ...
```

Boring. **Let's biicodize stackie** to make its use simpler.

First of all, this is the kind of `#include` users will write to use **stackie**:

``` cpp
#include <developer/stackie/stack.hpp>

int main()
{
    stk::stack stack;
}
```

And this is how the `developer/stackie` block will look like:

```
+-- developer/stackie
|    +-- bii
|    |    +-- paths.bii
|    |
|    +-- stack.hpp
|    +-- array.hpp
|    +-- list.hpp
|    +-- vector.hpp
|    |
|    +-- src
|    |    +-- stack.cpp
|    |    +-- array.cpp
|    |    +-- list.cpp
|    |    +-- vector.cpp
|    |
|    +-- README.md
|    +-- LICENSE
|    +-- CMakelists.txt
```

These are the steps to create/update `developer/stackie` block:

 - Create/open the block.
 - Copy the contents of the `include/` directory of stackie codebase into the block's root directory.
 - Copy the `src/` folder into the block's root directory.
 - Replace all references to `include/` to `#include` directives with "developer/stackie/".
 - Copy `README.md` and `LICENSE` files into the block root directory.
 - Copy biicode setup files (`CMakelists.txt`, `paths.bii`, etc) into their corresponding locations. *At this time the tool only lets you specify those files manually.*

The biicodize.yml file
----------------------

biicodizer provides a way to automatize the biicodization process shown above with a simple YAML description file called `biicodize.yml`. `biicodize.yml` describes all information and transformations needed to translate your codebase into a biicode block.

Let's see an example for stackie:

``` yaml
block:
  global:
    username: developer
    blockname: stackie
    readme: README.md : /README.md
    license: LICENSE : /LICENSE
  source:
    include:
      - include/ : /
    src:
      - src/ : /src/
  build:
    cmakelists: CMakeLists.txt.biicode : /CMakeLists.txt
    paths: paths.bii : /bii/paths.bii
``` 

### biicodizer translation patterns

biicodization mostly consists on copying source files to a new location inside the block and updating the `#include` directives to match that new locations. 

`biicodize.yml` file uses patterns like `[SOURCE] : [DEST]`, where `SOURCE` is a path (file or folder) from your codebase, and `DEST` is its new path relative to the block's root directory (Note the `/`). 

For example, in stackie's codebase there is a `CMakeLists.txt.biicode` file which stands as the CMake file for our block. Named as `.biicode` not to be confused with the original library CMake file. With the translation pattern, it is placed in the block and renamed to `CMakelists.txt`.

This translation works on folders too, note how headers are biicodized, the `include/` folder is directly translated instead of translating each file one by one.

Second `DEST` field is completely optional. Just with the `SOURCE`, biicodizer supposes the path is exactly the same, just relative to the block's root directory. For example:

```yaml
block:
  source:
    src:
      - src/foo.cpp
      - src/bar.cpp
```

biicodizer will place both `src/foo.cpp` and `src/bar.cpp` files at `username/blockname/src/`.

### `global` entry

`global` entry provides information about the block: Its name, the biicode user who maintains the block, its readme file, license, etc.

Only `username:` and `blockname:` fields are required, others are optional.

### `source:` entry

`source:` entry does the mapping of your C/C++ source files, with two different subentries `include:` and `src:` for headers and source files respectively.  
Each subentry contains a list of files specified via translation patterns.

``` yaml
block:
  source:
    include:
      - include/foo.hpp : /foo.hpp
      - include/bar.hpp : /bar.hpp
    src:
      - src/foo.cpp
      - src/bar.cpp
```

### `data:` entry

`data:` entry allows you to include more than just source files into your blocks, such as assets, binaries, etc.

Entries of `data:` extend the translation pattern with specific syntax to specify which file depends on which data. With this, biicode knows what data it has to retrieve if the file is used/requested.

Syntax is as follows:

```
[TRANSLATION PATTERN] -> [FILE/FOLDER]
```
An Example
-----------

Manu writes a game engine and deploys it via biicode. Manu also writes an example block with a simple game, that game uses some assets like sprites, sound effects, etc.

This is Manu's game structure:

```
+-- game
|    +-- assets
|    |    +-- pong
|    |    |    +-- pong.wav
|    |    |    +-- ball.png
|    |    |
|    |    +-- tetris
|    |         +-- tetris_song.mp3
|    |         +-- spritesheet.png
|    |     
|    +-- pong.cpp
|    +-- tetris.cpp  
```

As you can see, assets folder is organized by examples, there is one folder per example. 

To specify to biicode to download pong's assets only if pong example is used, and exactly the same for the tetris example.   

As : `pong.cpp` depends on `assets/pong/`, and `tetris.cpp` depends on `assets/tetris/`: just write:

``` yaml
block:
  data:
    - assets/pong/ -> pong.cpp
    - assets/tetris/ -> tetris.cpp
```

