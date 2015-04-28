These instructions supercede the prior instrcutions now held in the [Old Build Instructions](http://code.google.com/p/strongtalk/wiki/OldBuildInstructions).

# Building Under Windows using VS 2008 Express #

Check out the project using the gcc-linux branch (for the time being). Follow the instructions on the Source tab, replacing svn/trunk in the URL with svn/branches/gcc-linux. Despite the name of this branch, it also includes support for Windows and OSX. In effect, this branch forms the current HEAD for the project. When time permits the code from this branch will be merged into the trunk.

In addition to the VM code checked in to SVN you will also need an up-to-date image and sources directory. These can be found in the image and sources download for the latest build from the downloads page - [image and sources](http://strongtalk.googlecode.com/files/strongtalk-r162-image%2Bsources.tgz).

You should unpack the sources folder and the strongtalk.bst file from this archive into the directory formed by your checkout. This should replace the strongtalk.bst image file from the repository with the current one, and add a new source directory.

Once this is done, you should open the VS solution file found at build.win32/strongtalk.sln in your checkout.

The first time that you build the project, and any time that include file dependencies are changed, you should modify the properties of the strongtalk project found within the solution to rebuild the include files using makedeps. There is a Pre-Build Event found in the strongtalk project to do this. Change the Excluded From Build flag from Yes to No the first time that you build the project, and back again afterwards.

You should now be able to build the solution from within VS 2008 Express. All being well VS should compile the project code and run the existing CPP tests and you should see a summary at the bottom of the screen showing something like

```
1>SUMMARY
1>Test summary: SUCCESS
1>Number of test cases ran: 28
1>Test cases that succeeded: 28
1>Test cases with errors: 0
1>Test cases that failed: 0
```

If this does not appear, please post a message to the list describing the problem and any error messages that you receive during the build.


---


A few bits of additional information follow.

#### Building information on the primitives (prims.src and prims.inc) ####

The virtual machine has an extensive set of Smalltalk/Strongtalk primitive operations.  Information about these primitives is needed by the VM and by the Smalltalk-side code for several reasons:

  * The bytecode compiler needs to know the structure of the primitives so that primitive calls (which are in-line expressions in Strongtalk, unlike other Smalltalks) can be correctly compiled.
  * Similarly, the typechecker needs to know the type signatures of the primitives, so the calls can be typechecked (although that is an optional operation).

Rather than require this information to be separately and redundantly maintained for each primitive, Strongtalk has facilities for automatically extracting and generating the necessary information from the primitive source code.

The primitive information is dealt with as follows:

##### The primitive source code #####

In the C++ source code, the primitives are defined in include files in the vm/prims directory.  Here is an example from double\_prims.hpp:

```
  //%prim
  // <Float> primitiveFloatLessThan:  aNumber   <Float>
  //                         ifFail: failBlock <PrimFailBlock> ^<Boolean> =
  //   Internal { doc   = 'Returns whether the receiver is less than the argument'
  //              flags = #(Pure DoubleCompare LastDeltaFrameNotNeeded)
  //              name  = 'doubleOopPrimitives::lessThan' }
  //%
  static PRIM_DECL_2(lessThan, oop receiver, oop argument);
```

As you can see, a macro (PRIM\_DECL\_2) is used to define the entry point itself.  The additional information necessary is put in a special syntax in the associated comment.

##### Generating prims.src #####

To extract information in the source code, the primDefFilter.bat script is used.  It parses the .hpp files, and produces a file called prims.src.

Basically, all primdeffilter does is find the comments that start with '//%prim', and extract the special syntax (stripping off the '//'s) up to a comment starting with '//%', into the prims.src file, terminating each entry with a line containing only '!'.  The end of the file is indicated by another line containing only '!'.

##### Generating prims.inc #####

Once the primitive annotations have been extracted or hand-entered into prims.src, a Strongtalk tool, DeltaPrimitiveGenerator, is used to parse that and produce the prims.inc file.

To run this tool, you must place the prims.src file in the in the startup directory for Strongtalk, run Strongtalk, and evaluate 'DeltaPrimitiveGenerator doit'.  This will generate prims.inc in that same directory.  prims.inc should then be moved to the vm/prims directory.

#### Precompiled headers in MS Visual Studio ####

Under Visual Studio, the build system uses precompiled headers.  While some people think that precompiled headers are just an optimization to reduce compile time, in fact under VC++ that is not the only impact.  Experiments show that the size of the executable is significantly impacted, demonstrating that precompiled headers under VC++ definitely increase the code sharing (and probably therefore impact code locality and working set size) for the code generated for the precompiled header.

The project file (bin\strongtalk.vcproj) is already set up to generate and use the precompiled headers.  However, if you would like to understand how the project file was set up to do this, following is a bit more information.

The precompiled header is based on a single generated include file, `bin\incls\_precompiled.incl`, which in turn includes all the include files that are to be part of the precompiled header.  `_precompiled.incl` is generated by makedeps, which puts into that any include file which is included more than some threshold number of times in the system.

The actual precompiled header itself is generated by compiling one source file that uses `_precompiled.incl` using the /Yc (create precompiled header) flag.  All the other source files instead are compile with the /Yu (use precompiled header) flag.

Any file that does **not** include `_precompiled.incl` must be compiled with the "Not using Precompiled Header" option set in the project.  In addition, any such file must have the keyword "no\_precompiled\_headers" listed as one of its dependencies in the deps files.  Currently, there is just one file like this, runtime\os.cpp (now renamed os\_nt.cpp with the factoring out of a linux/gcc compileable version).

Since the precompiled headers need to be built before they are used, the first file compiled must be the one using the /Yc option.  That file is runtime\shell.cpp.

So if you look at the project properties, in the default properties for all files, you will see the /Yu option set under precompiled headers.  Then, shell.cpp overrides that with /Yc, and os\_nt.cpp overrides that with "Not using precompiled headers"

# Building under Linux #

Building under Linux is pretty simple. I use Eclipse CDT as my IDE, but using standard make, so you can build from the command line without Eclipse.

First you need to check out the gcc-linux branch from subversion - http://strongtalk.googlecode.com/svn/branches/gcc-linux.

The make files reside in a directory named build, immediately beneath the checkout directory. cd to the build directory and run the following

```
   make ROOT_DIR=<absolute-path-to-checkout> default
```

If everything goes well this should result in a strongtalk executable in the build directory. You should expect to see a number of warnings but no errors (at least under gcc 4.1.3).

To build with gcc optimizations enabled add OPT=-O[1, 2 or 3] to the build command line.

There are four build files in the build directory. You only need to be interested in three of them - makefile, makefile-macros.incl and makefile.incl. The fourth - makefile-asm.incl - is no longer used. It was used to build the assembler files. Since prunedtree translated the TASM assembler source to be dynamically generated using the builtin strongtalk assembler the raw assembler is redundant. This formed the basis for the gcc build (together with some changes from Dave Raymer) on which the linux port is based.

I have loaded a base linux compatible image together with the required source directory on to the downloads page. You should untar it in the strongtalk checkout directory.

The Strongtalk image maps a short form of the shared object/DLL name to the file name (under Windows 'kernel' maps to ' kernel32.dll' etc.).

Under Linux the equivalent mapping is currently to map libc to libc.so.6. The obvious mapping to libc.so doesn't work because, at least under my Ubuntu environment, this is a linker script that is not understood by dlopen :( If this file does not exist in your environment you may need to create a link with this name to libc.so. We need a better way of doing these mappings than the hard-coded approach currently in use, but that's the way it works for now.

Once you have made that change, you should be able to launch the image from the root of the strongtalk checkout, using something like the following.

```
   build/strongtalk -f tools/strongtalkrc-forInterpretedTests -script tools/test.dlt
```