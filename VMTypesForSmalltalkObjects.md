### The Oop Hierarchy: Smalltalk Object Pointers ###

In Strongtalk VM terminology, an _Oop_ (Object Oriented Pointer) is a [tagged](Tagging.md) reference to a Smalltalk object.  These are the actual Smalltalk object references that comprise the majority of the data on the Smalltalk heap.  Since the Oop is tagged, it cannot be directly dereferenced as a pointer to gain access to the object structure; it must be detagged first (and checked to make sure that it isn't a non-pointer reference like a SmallInteger, for example).    To dereference an Oop, the addr() function is used; since Oops are pointer types, this very frequent operation looks like _myoop_->addr().  The addr() function will remove the tagging information if necessary and return a usable C++ object pointer.

The code that manages Oop-s is contained in the vm/oops directory.  The hierarchy of Oop types (and their associated OopDesc classes, described below) is outlined in [oopsHierarchy.hpp](http://strongtalk.googlecode.com/svn/trunk/vm/oops/oopsHierarchy.hpp).

### The OopDesc Hierarchy: C++ Structures that map to Smalltalk object structures ###

When addr() is used to detag an Oop, a subtype of `(OopDesc * )` is returned.  There is an appropriate OopDesc class for each Oop type defined in [oopsHierarchy.hpp](http://strongtalk.googlecode.com/svn/trunk/vm/oops/oopsHierarchy.hpp).

The OopDesc hierarchy consists of C++ classes that map directly to the memory layout of Smalltalk objects.  This direct mapping allows the VM to be designed in a high-level, object-oriented fashion, unlike many other virtual machines, since the various kinds of Smalltalk objects can be directly manipulated through an abstract C++ interface.

However, there is a catch to this approach: since an OopDesc is an overlay on the actual Smalltalk object structure, it has a Smalltalk object header, rather than a C++ object header (i.e. the vtable).  Since the C++ vtable field is necessary for dispatching C++ virtual functions, this means that OopDesc classes _cannot_ contain virtual methods.

Since lack of virtual methods would severely restrict the usefulness of these C++ classes, a work-around for this has been provided.  One obvious solution to this problem would be to simply allocate a vtable field in every Smalltalk object.  But this would be an unacceptable space overhead, since most Smalltalk objects have only a couple of instance variables and two header words, meaning that addition of even one more word to every object would considerably increase the size of the Smalltalk heap, as well as increase pressure on the memory subsystem.

As a more reasonable alternative, consider that Smalltalk objects already have a Smalltalk class that is shared across all instances, so the Smalltalk class is an obvious place to locate a C++ vtable that could be used to delegate instance dispatch.  However, Smalltalk classes are also regular Smalltalk objects, so they too have only a Smalltalk header.  So where could a C++ vtable be stored?

### Klass: A special C++ object imbedded in a Smalltalk class object ###

The Strongtalk VM does this by imbedding a C++ Klass object (named with a K to avoid conflict with reserved words) in the Smalltalk class structure, which maps to a KlassOopDesc in C++.  Unlike OopDesc-s, Klass-es _do_ have a vtable entry, and thus the Klass hierarchy can contain virtual methods.   When a virtual function on an OopDesc class needs to be defined, it is defined instead on the associated !Klass class, and then an inlined non-virtual function is defined on the OopDesc that delegates to the virtual function on the Klass.

So to reiterate: like all kinds of Smalltalk objects, each Smalltalk class object maps directly to a instance of a subclass of OopDesc, or more specifically, a subclass of KlassOopDesc , which like other OopDesc-s does not have a vtable.  But the KlassOopDesc contains as its body an imbedded Klass C++ object, which is _not_ an OopDesc, so it may have a C++ vtable.  So basically KlassOopDesc simply acts as the Smalltalk object wrapper for an imbedded C++ Klass object.

In practice what this means in terms of the actual object structure is that each Smalltalk class object has an imbedded C++ vtable field directly after the Smalltalk object header.    The KlassOopDesc and the Klass go together; together they comprise a class object that has a Smalltalk object header followed by a C++ vtable header, so that by simply adding a constant offset to the address of the Smalltalk class we can get a C++ object with a vtable header.  This in effect allows us to treat Smalltalk class objects as either Smalltalk objects or full C++ objects, as needed.

The Klass hierarchy is also described in [oopsHierarchy.hpp](http://strongtalk.googlecode.com/svn/trunk/vm/oops/oopsHierarchy.hpp).

This brings up an additional issue that illustrates the advantages of the Strongtalk VM design: since the vtable for a Klass is an untagged value that is _not_ an Oop, but is imbedded as a field in each Smalltalk class object in the Smalltalk heap, it must be known to the garbage collector as a non-Oop, so that it is not accidentally treated as a Smalltalk object reference.  This illustrates that while most fields in Smalltalk objects are Oops, some internal fields are not.

In most VMs this would introduce additional complexity into the garbage collector, since it would have to know the intimate details about the structure of all the kinds of Smalltalk object layout (regular objects, doubles, classes, arrays, bytearrays, classes etc.) to know how to locate the oops during pointer traversal.  But since as we have shown we have a method of indirectly defining virtual functions on the OopDesc classes for each object type, we can instead design the garbage collector in an object-oriented fashion, so that each OopDesc class can indirectly define a virtual method for enumerating its imbedded Oops for the garbage collector.  This makes the garbage collector independent of the detailed structure of the various Smalltalk object types, and makes it easy to modify the internal VM format of Smalltalk objects.

[Next: VM classes for non-Smalltalk objects](NonOopVMTypes.md)