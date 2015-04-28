## Introduction ##

The Strongtalk virtual machine is a large, complex C++ program.  To manage that complexity, the VM design is based around the highly structured use of a number of core C++ classes and coding techniques, which you will see used over and over again in the code.  Much of this basic structure is inherited from the Self virtual machine design.  To understand the VM design, it is first necessary to understand the use of these core classes.

The fundamental difficulty in designing a high-performance virtual machine is caused by the conflict between the need for an abstract, high-level coding style to manage the VM complexity, with the requirement that the resulting code be very fast.  The Strongtalk VM accomplishes this through extensive use of C++ inline functions, preprocessor macros, and optimized allocation of VM objects (i.e. avoiding the C heap when possible).   While inline functions and macros certainly can make debugging more difficult, they nonetheless are indispensable in allowing the Strongtalk VM design to be very high-level and abstract without sacrificing performance.

### Smalltalk vs. C++ objects ###

First of all, we need to deal with a fundamental ambiguity: when we say object or class, are we talking about C++ objects/classes in the VM code, or are we talking about Smalltalk objects that the VM is simulating?  So it is necessary to be clear about the distinction.  In the Strongtalk VM, a fair amount of effort has been made to map Smalltalk objects into C++ objects, so that the VM can operate on the simulated objects in a natural and object-oriented way.

When trying to understand why the VM classes are structured the way they are, try to keep in mind a number of fundamental issues:

  * **Allocation**: Smalltalk objects are always allocated in the garbage-collected heap, whereas C++ objects are generally allocated elsewhere, either on the C heap, the stack, or in one of a number special areas used by the VM.  Two of the most important of these other areas are:
    * The **Resource Area**: Temporary C++ objects that cannot be stack allocated but which are only needed until the end of the current VM operation are allocated in the Resource area, which is a special chunk of contiguous memory that is tail-allocated for very high allocation speed.  When the current VM operation completes, the end-pointer for the allocation area is simply reset to the beginning, instantly freeing all resource objects.   However, note that such objects will **not** have their destructors called, so destructors are not used for classes whose instances will be allocated in the resource area.
    * The **Zone**: The zone is used to hold compiled code objects.
  * **Write Barrier**: Since Smalltalk objects are managed by a generational garbage collector that uses card-marking, all stores of Smalltalk references into Smalltalk object fields _must_ be recorded for the garbage collector.
  * **GC relocation**: Care must be taken to make sure that VM references to objects on the garbage-collected heap are not be maintained across a garbage collection, since they may be relocated, invalidating the reference.
  * **Dispatch**: Smalltalk objects have a different dispatch model and header format than C++ objects.  Because there is no vtable in the Smalltalk object header, C++ objects that map to Smalltalk objects cannot contain virtual functions.  This issue is discussed in the section on C++ classes for Smalltalk objects.

The above issues dictate that the C++ classes in the VM be divided into two major groups:

  * [C++ objects related to accessing Smalltalk heap objects from C++](VMTypesForSmalltalkObjects.md)
  * [Other VM objects that do not map to Smalltalk objects](NonOopVMTypes.md)

Let's start with the simpler case:

### VM objects not related to mapping to Smalltalk objects ###

All classes in the virtual machine (other than the classes in the OopDesc hierarchy and related classes, that map to the Smalltalk heap) must be subclasses of one of the following allocation classes:

  * For objects allocated in the resource area:
    * ResourceObj
      * PrintableResourceObj

  * For objects allocated in the C-heap (managed by: free & malloc):
    * CHeapObj
      * PrintableCHeapObj

  * For objects allocated on the stack:
    * StackObj
      * PrintableStackObj

  * For embedded objects:
    * ValueObj

  * For classes used as name spaces:
    * AllStatic

The printable subclasses are used for debugging and define virtual member functions for printing. Classes that avoid allocating the vtbl entries in the objects should therefore not inherit from the printable subclasses

[Next: C++ objects related to accessing Smalltalk heap objects from C++](VMTypesForSmalltalkObjects.md)

