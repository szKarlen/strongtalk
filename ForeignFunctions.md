Strongtalk supports both calling out to foreign functions, and callbacks into Smalltalk from foreign functions, in a simple and natural way.  Smalltalk and foreign code may be mixed on the stack arbitrarily; i.e. callouts can call callbacks which can callout, et cetera.

## ExternalProxy ##

The critical data type needed for interaction with external libraries is a way to encapsulate an unsigned, untagged word.  This is modeled by the class `ExternalProxy`.  Instances of `ExternalProxy` (or a subclass; using this class as a mixin does not work currently) are "magic" to the VM, in that they have a hidden, unsigned, untagged word instance variable.  There is a rich variety of access methods defined in `ExternalProxy` for setting and getting that word via standard Smalltalk basic types such as `SmallInteger`, and `Boolean`.

When the word represents a pointer, there are convenient methods for allocating and freeing on the C heap, as well as methods for indexing into it as an array, whether by byte, word, or double byte.

Since it is easy to define subclasses of `ExternalProxy`, you are encouraged to use subclasses for specific external structure types.  C/C++ structures should be subclasses of the subclass `CStructure`, which declares the method `structureSize ^<Int>`.  Ideally there would be tools for parsing header files to compute structure size and member offsets automatically, but that is not currently there yet.  So, at the moment those must be computed by hand.  The convention is to create a subclass of `CStructure` for each external structure type, with the category "offsets" defining methods returning the offset of each member.  Then, the "accessing" category defines getting and setting methods for each member, converting types as appropriate using methods inherited from `ExternalProxy`.  See the class `WinPOINT` for a simple canonical example.

The `ExternalProxy` subclass `CString` is provided for accessing external strings.

`ExternalProxies` are then used to represent both arguments and return values for external functions.

## Callouts ##

A callout in Strongtalk is simply an expression of the form **{{<_libname_ _returnClass_ _functionName_> keyword1: _exp1_ keyword2: _exp2_ ...}}**

All arguments must be either `ExternalProxies`, or `SmallIntegers` (which are automatically converted to signed C integers).  The _returnClass_ is the subclass of `ExternalProxy` that you wish the VM to allocate for the return value.  `SmallInteger` may not be used for the return class.  The arguments are specified in the familiar Smalltalk keyword format; the keywords are irrelevant and are provided solely for readability.  The arguments can be arbitrary Smalltalk expressions.

Here is an example from a class method in the Win32-specific class `HBITMAP` (used internally by the UI class `Image`, which presents an OS-independent interface):

```
fromWin32File: f <FilePath> ifFail: blk <[^X def]> ^<HBITMAP | X>
    | inst <HBITMAP> nm <CString> |
    nm := CString for: f name.
    [	inst := {{<user HBITMAP LoadImageA>
                    hinstance: Win32 NULL
                    name: nm
                    type: Win32 IMAGE_BITMAP
                    xSize: 0    "We will use the size in the file"
                    ySize: 0
                    flags: (Win32 LR_DEFAULTCOLOR
                                externalBitOr: Win32 LR_LOADFROMFILE)
                }}.
        inst isNull
            ifTrue: [ ^blk value ].
    ] ensure: [ nm free ].
    ^inst
```

There are several things to note:
  * because callouts are expressions, they can appear anywhere.
  * `user` is the name of the external dll.
  * `HBITMAP` is the subclass of `ExternalProxy` that will be allocated to encapsulate the return value.
  * Note the use of `CString` to allocate a C string for the Smalltalk string, and the use of an #ensure: clause to guarantee that it is freed on return.  #ensure: should always be used when allocating temporary external memory, to prevent memory leaks.  For example, here the string would not otherwise be freed if the return value was NULL.
  * Note that the argument keywords improve readability considerably, although they have no semantic content.
  * Note the use of arbitrary expressions as arguments, yielding both `ExternalProxies` and `SmallIntegers` as needed.
  * Note that `ExternalProxy` defines convenient methods like #externalBitOr: and #externalBitAnd: to do boolean operations on unsigned, untagged words.
  * By convention, external constants should be defined elsewhere as constant methods; in this case the class Win32 is used for Win32 constants.  This helps readability considerably.  Note that the Strongtalk compiler will in-line all such sends to literal classes- they are in effect free at equilibrium.

## Callbacks ##

Callbacks also are very simple.  In effect, all you do is register a block with the VM, which allocates a corresponding external function, which is represented in Smalltalk as an instance of a subclass of `Callback`, which is a kind of `ExternalProxy`.   You can pass the callback to external code.  When the external function is called, the block is invoked.

`Callback` currently has two subclasses, `CCallBack`, which uses normal C calling conventions, and `APICallBack`, which uses Pascal calling conventions, which are needed for Win32 event handlers, for example.

A `Callback` is created by sending the message #register:parameters: to the appropriate `Callback` subclass.  The first argument is the block that the callback should invoke, and the second argument is a `SequencableCollection` holding the Smalltalk **classes** that should be instantiated to represent the corresponding arguments to the block, i.e. either subclasses of `ExternalProxy`, or `SmallInteger`.

Here's an example of creating a callback for a Windows event handler (this example is a class method of `Session`, which caches the event handler in the class variable `WndProc`):

```
wndProc ^<APICallBack>

    WndProc isNil
        ifTrue: [WndProc := APICallBack 
                    register:
                        [ :hwnd <ExternalProxy> :msg <SmallInteger> :wParam <ExternalData> :lParam <ExternalData> |

                            (Session forProcess: Processor activeProcess)
                                message: msg for: hwnd wParam: wParam lParam: lParam ]
                    parameters: (OrderedCollection[Class]
                        with: ExternalProxy with: SmallInteger with: ExternalProxy with: ExternalProxy).	].
    ^WndProc
```

Notes:
  * The type `ExternalData` used above is defined as `(ExternalProxy | SmallInteger)`.