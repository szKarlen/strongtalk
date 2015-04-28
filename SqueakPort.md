## Possible Approaches ##

  1. Emulation of the Squeak VM
  1. Porting the Smalltalk code
    1. Minimal: changing

## Differences to Investigate in the Language Models or VM Interface/Architecture ##
  * Language Model
    * Traits vs. Mixins?
    * Namespaces?
    * Lack of class instance variables in Strongtalk
    * differences in closure support:
  * VM
    * support for #become:, use of Obsolete classes in Squeak?
    * plugins
    * process model
    * Does Strongtalk VM have enough reflective facilities now to support debugger?

## Other Issues to Investigate ##

  * [Relevant Notes on the Squeak VM Architecture](SqueakVMNotes.md)