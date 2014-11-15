biicodizer
==========

A biicode blocks generator

Why biicodizer?
---------------

[biicode]() is a great dependency manager for C and C++. Its cmake based, so taking an existing library and build up a biicode block to deploy the library is not a hard process, since cmake is a well known and widely adopted system for build configuration.  
Using cmake means your library could be portable simply tunning your sources and your `CMakeLists.txt` properly. And when that works biicode brings you the opportunity of sharing and using the library across multiple platforms in a simple way, just doing `#include <your_biicode_user/your_library/header.hpp>`.

But that *biicodization* process is not always as simple as it could be. 

You should create a block, add the sources of your library, add your custom `CMakeLists.txt` file... A boring and error-prone process done manually and, in most of the cases, which includes copying your sources to the block and adapting the `#include`s across your codebase to work with the biicode `#include <username/block/header.h>` convention.
So, except you maintain your library entirely as a biicode block, having the biicode block up to date to your library sources means copying sources again. So more manual, boring, and error-prone sources copying and adapting. 

What we propose here is a simple-to-use python script which manages the process, given a block description file written by the developer.

Anatomy of a biicode block
--------------------------

A biicode block contains all the sources plus some configuration files, such as the `CMakeLists.txt` file or the biicode settings files located under the `bii/` folder, which biicode needs to correctly build and link a block within other blocks. 

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

Note how sources are placed inside the block root directory, instead of the common `include/` and `src/` folders you may have in your codebase. Thats done to follow the `#include <user/block/foo.hpp>` convention, instead of the more ugly (This can be subjective...) `#include <user/block/include/foo.hpp>`.

Also the block usually contains a `LICENSE` file, and a `README.md` file, both at the block root directory.

The biicodization process
-------------------------

Imagine you have a C++ library called *"stackie"* which provides the most basic data structures ala Standard Library, such as a linked list, an array, a stack, etc. This is your codebase:

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

If someone wants to use your library, should download it, include the headers, and then compile and link all the `.cpp` files:

``` shell
$ ls
  use_stackie.cpp
$ git clone https://github.com/developer/stackie.git
  Resolving deltas, blah, blah...
$ ls 
  stackie/    use_stackie.cpp
$ g++ ...
$ g++ ...
$ g++ ...
$ ...
```

So boring. **Lets biicodize it**.

First of all, this is the kind of `#include` we want the users write to use stackie:

``` cpp
#include <developer/stackie/stack.hpp>

int main()
{
    stk::stack stack;
}
```

Then this is how the `developer/stackie` block should look like:

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

What things we should do to create/update that block?

 - Create/open the block.
 - Copy the contents of the `include/` directory of stackie codebase into the block root directory.
 - Copy the `src/` folder into the block root directory.
 - Replace all references to `include/` on `#include` directives with "developer/stackie/".
 - Copy the `README.md` and `LICENSE` files into the block root directory.
 - Copy the biicode setup files (`CMakelists.txt`, `paths.bii`, etc) you created for the block into their corresponding locations. *At this time the tool is not smart enough to write that files automatically, so you should write them.*

The biicodize.yml file
----------------------

biicodizer provides a way to automatize the biicodization process shown above with a simple YAML description file called `biicodize.yml`. That file describes all the information and transformations needed to translate your codebase into a biicode block.

Lets see an example for stackie:

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

Most of the biicodization job consists on copying source files to a new location inside the block and update the `#include` directives to take care of that new locations. 

For that purpose, the `biicodize.yml` file uses patterns of the form `[SOURCE] : [DEST]`, where `SOURCE` is a path (File or folder) from your codebase, and `DEST` is its new location relative to the block root directory (Note the `/`). Those paths include the full name, so you can use them to translate and change the file/folder name.
For example, in the stackie codebase there is a `CMakelists.txt.biicode` file which will act as the cmake file for our block. We named it as `.biicode` to not confuse with a real cmake file for the library. With the translation pattern, we place it in the block renaming the file to `CMakelists.txt` too.

This translation works for folders too, note how the headers are biicodized translating the `include/` folder directly instead of translating each file one per one.

Note the second `DEST` field is completely optional. If you write the source only, biicodizer will suppose that the path is exactly the same, but relative to the block root directory. For example:

```yaml
block:
  source:
    src:
      - src/foo.cpp
      - src/bar.cpp
```

biicodizer will place both the `src/foo.cpp` and `src/bar.cpp` files at `username/blockname/src/`.

### The `global` entry

This entry provides information about the block: Its name, the biicode user who maintains the block, its readme file, license, etc.

The `username:` and `blockname:` fields are required, but the others are optional.

### The `source:` entry

The `source:` entry of the description file does the mapping of your C/C++ source files, with two different `include:` and `src:` subentries for headers and source files respectively.  
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

### The `data:` entry

This entry allows you to include files that are not source files into your blocks, such as assets, binaries, etc.

Since data dependencies are usually required by source files as a dependency, the entries of `data:` extend the translation pattern with specific syntax to specify what file depends on that data. Then biicode understands that data should be retrieved if the file is used/requested.

The syntax is as follows:

```
[TRANSLATION PATTERN] -> [FILE/FOLDER]
```

Consider an example: An user have written a game engine and deploys it via biicode, and now the same user writes an example block with a simple game. That game uses some assets such as sprites, sound effects, etc.

This is the game structure:

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

As you can see, the assets folder is organized by examples, with one folder per example. We should specify biicode that the assets of the pong example should be downloaded only if the pong example is used, and the same for tetris.   
How can we do that? Easy: `pong.cpp` depends on `assets/pong/`, and `tetris.cpp` depdends on `assets/tetris/`:

``` yaml
block:
  data:
    - assets/pong/ -> pong.cpp
    - assets/tetris/ -> tetris.cpp
```

