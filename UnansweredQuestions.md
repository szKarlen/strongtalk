# Unanswered Questions #

This page is intended to hold questions about the VM or Smalltalk image code to which we have no answers. Typically these will reflect design decisions made by the original developers who are no longer actively involved in the project. As answers to these questions are supplied this page will be updated accordingly.

**Q:** Some of the klasses use Universe::allocate\_tenured() rather than a simple Universe::allocate() eg. klassKlass, associationKlass, and symbolKlass (but only in allocateSymbol() - it doesn't override the inherited allocateObject()). I guess I can understand why classes and symbols are allocated tenured, by why associations? And if classes are allocated tenured, why not mixins and methods?

**A:**

**Q:** The Disassembler has a comment "excluded - provide empty methods" and some empty implementations. Does this refer to exclusion from the Sun release, or something else? Why were these excluded?

**A:**