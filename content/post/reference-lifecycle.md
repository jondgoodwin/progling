---
title: "The Lifecycle of a Reference"
date: 2020-05-25T11:08:02+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Memory"
  - "Types"
  - "Permissions"
tags:
  - "Cone"
---

[References](/post/reference-types) are easily the most complex and innovative type in
]Cone's](http://cone.jondgoodwin.com/) type system.
The complexity of references results largely from the interrelated
mechanisms that it is composed from: regions, 
[permissions](/post/race-safe-strategies/), [lifetimes](/post/lifetimes/),
and the type of the value it points to.

Although these mechanisms have all been described separately
in several places, I thought it might be valuable to provide
a detailed description of how they work together, as choreographed by the compiler.
The best way to do this is focus on each of the distinct
events that a reference experiences from birth to death.

This post will focus on simple, regular references.
Array and virtual references work similarly (but not identically).
This post will not cover non-pointer references used by first-class
arenas or pools, as their mechanisms work differently in important ways.

## Birth ##

For the time being, let's ignore global and local variables,
and focus only on heap-allocated objects.
Whenever we allocate a new object on the heap, we must specify
a region and an initial value or type initializer. 
We may also specify a permission, but if we do not, `uni` is assumed.

    imm newref = &rc 4
	
This allocates a new integer object using the ref-counted (`rc`) region,
with the `uni` permission and initialized with the value of 4.
What we get back is the very first reference to that new object.
Under the covers, this work unfolds in two steps: first allocation, then initialization.

### Allocation ###

To allocate memory, the compiler generates a call to the private `_allocate` method
of the selected region, and passes it the number of bytes to allocate.
This is a statically calculated number, offering enough space to hold the 
region's bookkeeping data, the permissions's bookkeeping data,
and the data for the object itself.
Here is an example of a region's _allocate method, suitable for (say) 
reference counting or single-owner regions:

    fn _allocate(size usize) *u8:
	   malloc(size)

Notice that allocate returns a pointer to a byte, with a null value
indicating that the allocation failed. The compiler will recast this into
a nullable reference to the correct type. The programmer decide whether
to explicitly handle an allocation failure or allow it to panic implicitly.

_allocate need not use malloc or equivalent general purpose allocator.
It may use mmap, allocate quickly out of a bump-pointer arena (e.g., a GC nursery),
or even allocate out of some dynamically allocated, first-class region.

Some regions will indicate, via an attribute, that they require run-time type
information (RTTI). This is not needed for regions whose reference events are all statically
invoked (e.g., single owner or reference counting), but is needed
where a region's runtime logic (e.g., an arena or garbage collector) 
requires information about the object to process certain events correctly.

If RTTI is required, a pointer to this information is passed as the second parameter
to _allocate. Examples of useful RTTI might include:

- The offset of the object data
- A nullable pointer to the object's type's finalizer, automatically invoked
  during object death (see below).
- A nullable pointer to the tracing function for the object, invoked by a collector.


### Initialization ###

If the allocation succeeds, the compiler then invokes the `_init` method
for the region and permission. Permissions generally only need to initialize
when they support runtime locks. Regions need to initialize to set up
runtime bookkeeping information for memory management. 
For example, the reference counting region trait has a field called
`count` which keeps track of the number of references to this object.
The `rc` region's init would need to initialize it:

    fn _init(self &wri rc):
	    self.count = 1   // initialize reference counter to 1

Initialization for a tracing garbage collector might initialize the tracing color to white.
For improved performance, it might be better to store color information as bits
independently of the object. In the case, it might be easier to initialize the
object's colors as part of _allocate rather than _init.

After initializing the region and permission data, 
the compiler initializes the allocated object itself.
If the allocator is given a value, this value is automatically
copied into the allocated area. If the allocator specifies some type's initializer method,
this method is called with `self` being a write-only
reference to the object's type. The method's logic must
store a valid value into the location.

Each of the initializers only sees and initializes its part of the allocated memory.
The compiler accomplishes this by automatically recasting the pointer returned by
allocation into a reference to the appropriate offset into that allocated area
prior to each initializer call.

Although it may seem expensive to call four separate methods to allocate
and initialize an object, these methods are all inlined, and thereby
generate lean logic. For the original example, this means a malloc, setting count to 1,
and then storing a copy of 4 into the value area of the object.

## Dereference ##

Dereferencing is used to either retrieve (load) the value the reference points to,
or to replace it with (store) a new value.

Before it generates the appropriate load or store logic, it goes through
a gauntlet of tests to ensure that it is safe to perform the requested load or store.
Any violation of these rules triggers a compile-time error:

- The reference's permission must allow read access for load, and mutability for store.
- The reference's permission must not be a locked permission.
- Any value being stored must have the same type as the reference points to
- The lifetime of any value being stored must be the "same" or last longer
  than the lifetime of the reference.
  
Prior to generating the appropriate load (or store), 
the compiler will invoke the _readBarrier (or _writeBarrier) method,
if defined for the region.

If the region implements the `isAlive` method for weak references,
dereferencing no longer generates a simple load or store.
Load-based dereferencing will return a nullable value, the referenced
value (if it is still alive according to _isAlive) or null otherwise.
Similarly, store-based dereferencing only succeeds if alive,
and panics otherwise. isAlive may be directly invoked on a weak reference.

If the value loaded from a dereference has move semantics,
the compiler may freeze out subsequent use of the variable that holds the reference
(or may disallow the dereference altogether).

## Copy (or Move) ##

A reference has move semantics if its region has move semantics (e.g., single owner
or the linear flavor of reference counting) or the permission has move semantics
(the `uni` permission). Otherwise the reference has copy semantics.
Notice that the move/copy semantics of the pointed-at object's type
has no bearing on the reference.

Certain operations nearly always copy a reference (except when move semantics forbid it), 
particularly assignment and passing a reference on some function call.
If the new binding offers access by another thread, a compile-time error
will be produced if the permission does not permit thread sending.

The binding the reference is copied into need not have exactly the same reference signature,
so long as the coercion is safe. For example, a `uni` reference may be coerced to
any other permission, and most permissions will safely coerce to `const`.
The region must be the same (unless we are doing a borrow, which is described later).
The value type may often be coerced to a safe supertype.
If the no safe coercion is possible, a compiler error is produced.

A reference copy makes a copy of the reference into the new binding.
Additionally:

- Copy-based references will invoke the region's _alias method, if defined.
  Reference counting might use this method to increment the reference count.
- Move-based references will freeze out the source variable for the original reference,
  so that it may never be used again. It will not invoke the _alias method,
  since there is still only one live reference to the object.
  
Note that storing a copied/moved value into a variable that has been frozen due to a move or borrow,
effectively undoes the freeze.

### Borrowed Reference ###

There are two key ways to create a lifetime-constrained borrowed reference:

- Borrow a reference from a global or local variable or function (e.g., `&mut glovar`)
- Make a borrowed copy of an existing reference (e.g., `&*intref`)

Although creating a borrowed reference essentially involves making a copy of
some existing reference, it invokes several additional mechanisms.

- The lifetime constraint of the borrowed reference is captured.
  When borrowing from a global name, the lifetime is `static`. Otherwise,
  the borrowed reference's lifetime is the block scope where the borrow took place.
  The borrowed reference is prohibited from surviving beyond than its lifetime.
  
- The variable binding that is the source of the borrowed reference is frozen.
  Its value cannot be accessed while frozen. This freeze is thawed when all borrowed references
  expire, either at the end of the scope, or when the frozen binding is given a new binding.
  If the binding is unfrozen, it is an error to use any borrowed reference
  obtained from it.
  
- The borrowed reference may obtain a different, but safe permission from the reference
  it borrows from. Multiple borrows may be obtained from the same binding,
  so long as the constellation of permissions obtained are safe.
  
- If the original reference uses a locked permission, the permission's
  _acquire method is invoked to obtain the borrowed reference.
  When the borrowed reference expires (see below), the permission's _release method is invoked.

- If the reference points to a "dependent" typed value, one whose 
  shape can change due to mutation, such as resizeable vectors and variant types.
  [Interior references](/post/interior-references-and-shared-mutable/)
  may be borrowed only for certain permissions.

## Reference Death ##

## Object Death ##
