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
[Cone's](http://cone.jondgoodwin.com/) type system.
A reference is a safe-to-use pointer type that not only specifies the type of the
object it points to, but also the:

- Region, the memory management strategy which ensures the object's memory is automatically
  and safely reclaimed. In addition to the built-in global and stack-based regions,
  Cone supports a versatile range of heap-based region strategies,
  including single-owner (e.g., Rust's Box\<T\>), reference counting,
  tracing garbage collection, arenas, and pools. References lacking a
  region are called borrowed references.
- [Permission](/post/race-safe-strategies/), which ensures data race safety
  by granting and constraining how a reference may be used:
  can you read its value? mutate its value? make a copy of it? share it across threads?
- [Lifetime](/post/lifetimes/), used by borrowed references to
  constrain how long they may live before they must expire,
  thereby ensuring use of borrowed references is memory safe.

What makes these mechanisms challenging to support is that
regions and permissions are programmer-definable `struct`-like types,
that specify their own fields and methods.
In effect, Cone allows programs to define their own memory management
allocation and recycling capabilities in the form of regions,
and their own data race safety capabilities in the form of permissions.
It is the compiler's job to choreograph
how operations on references invoke programmer-defined capabilities
in custom (or library-offered) regions and permissions.

Although each of these mechanisms has been described separately
in several places, I thought it might be valuable to provide
a detailed walkthrough of how they work together.
The best way to do this is focus on each of the distinct
events that a reference experiences.

## Regular References ##

Let's begin by following the lifecycle of events that a regular reference
undergoes as it flows from birth to death.

### Birth ###

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

#### Allocation ####

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
indicating that the allocation failed. The compiler will later recast this into
a nullable reference to the correct type. The programmer decides whether
to explicitly handle an allocation failure or allow it to panic implicitly.

_allocate is not restricted to using malloc or equivalent general purpose allocator.
It may use mmap, allocate quickly out of a bump-pointer arena (e.g., a GC nursery),
or even allocate out of some dynamically created, first-class region.

A region definition may specify, using a compiler attribute, that it requires run-time type
information (RTTI). This is not needed for regions whose reference events are all statically
invoked (e.g., single owner or reference counting). However, it is required
whenever a region's runtime logic (e.g., an arena or garbage collector) 
requires information about its allocated object to process certain events
(e.g., finalization, tracing and freeing) correctly.

If RTTI is required, a pointer to this information is passed as the second parameter
to _allocate. Examples of useful RTTI might include:

- The offset of the object data
- A nullable pointer to the object's type's finalizer, automatically invoked
  during object death (see below).
- A nullable pointer to the tracing function for the object, invoked by a collector.
- A nullable pointer to dealiasing logic for references inside the object


#### Initialization ####

If the allocation succeeds, initialization proceeds in three steps: first the region data, then the permission data,
and finally initializing the object's value.

To initialize **region** data, the compiler invokes the `_init` method
for the region. Many (but not all) regions have object-specific bookkeeping data
to facilitate runtime-based memory management mechanisms.
For example, the reference counting region might define a field called
`count` which keeps track of the number of references to this object.
The region's init initializes this data:

    fn _init(self &wri rc):
	    self.count = 1   // initialize reference counter to 1

Initialization for a tracing garbage collector might initialize the tracing color to white.
Alternatively, for improved performance, it might choose to store color information as bits
independently of the object. In this case, initialization of the
object's colors could be performed by _allocate rather than _init.

Static-only **permissions** often do not carry any bookkeeping data, and therefore
will not require any initialization.
However, permissions that generate runtime logic (e.g., those that provide runtime locks) do.
The permission's `_init` method would be invoked to initialize this
runtime bookkeeping data (e.g., initializing an atomic lock).

After initializing the region and permission data, 
the compiler initializes the allocated **object** itself.
If the allocator is given a value, this value is automatically
copied into the allocated area. If the allocator specifies some type's initializer method,
this method is called with `self` being a write-only
reference to the object's type. The method's logic must
store a valid value into the location.

Each of these three initialization calls only see and initialize their part of the allocated memory.
The compiler accomplishes this by automatically recasting the pointer returned by
allocation into a reference to the appropriate offset into that allocated area
prior to each initializer call.

Although it may seem expensive to call four separate methods to allocate
and initialize an object, these methods are all inlined, and thereby
generate lean logic. For the original ref-counted example, this means a malloc, setting count to 1,
and then storing a copy of 4 into the value area of the object.

### Dereference ###

Dereferencing is used to either retrieve (load) the value the reference points to,
or to replace it with (store) a new value.

Before it generates the appropriate load or store logic, it goes through
a gauntlet of tests to ensure that it is safe to perform the requested load or store.
Any violation triggers a compile-time error:

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

### Copy (or Move) ###

A reference has move semantics if one of these is true:

- its region has move semantics (e.g., single owner
  or the linear flavor of reference counting) or,
- the permission has move semantics (the `uni` permission)

If not, the reference has copy semantics.
Notice that the move/copy semantics of the pointed-at object's type
has no bearing on the reference.

Certain operations nearly always copy a reference (except when move semantics forbid it), 
particularly assignment and passing a reference on some function call.
If the new binding enables the reference to be used by another thread, a compile-time error
will be produced if the reference's permission does not permit thread sending.

The binding the reference is copied into need not have exactly the same reference signature,
so long as the coercion is safe. For example, a `uni` reference may be coerced to
any other permission, and most permissions will safely coerce to `const`.
The region must be the same (unless we are doing a borrow, which is described later).
The value type may often be coerced to a safe supertype.
If the coercion is unsafe, a compiler error results.

A reference copy makes a copy of the reference into the new binding.
Additionally:

- Copy-based references will invoke the region's _alias method, if defined.
  Reference counting might use this method to increment the reference count.
- Move-based references will freeze out the source variable for the original reference,
  so that it may never be used again. It will not invoke the _alias method,
  since there is still only one live reference to the object.
  
Another way to create a copy of a reference involves using some public method of a region
on a region-based reference. For example, the `clone` method
might be used to create a counted copy of a move-semantics based, ref-counted reference.
Similarly, the `weak` method might be used to create a weak reference
from a strong ref-counted reference. The weak reference would belong to a companion
region, having different aliasing and de-referencing behavior
(but no allocate behavior).
  
Note that storing a copied/moved value into a variable that has been frozen due to a move or borrow,
effectively unfreezes that variable.

#### Borrowed Reference ####

There are two ways to create a lifetime-constrained borrowed reference:

- Borrow a reference to a variable or function (e.g., `&mut glovar`)
- Create a borrowed copy of an existing reference (e.g., `&*intref`)

Since the creation of a borrowed references copies an existing reference,
all of the above still applies. However, borrowing invokes additional 
compiler mechanisms:

- The lifetime constraint of the borrowed reference is implicitly captured.
  When borrowing from a global name, the lifetime is `static`. Otherwise,
  the borrowed reference's lifetime is the block scope where the borrow took place.
  The borrowed reference is prohibited from surviving beyond its lifetime.
  
- The variable binding that is the source of the borrowed reference is frozen.
  Its value cannot be accessed while frozen. This freeze is thawed when all borrowed references
  expire, either at the end of the scope, or when the frozen binding is given a new binding.
  If the binding is unfrozen, it is an error to use any borrowed reference
  obtained from it.
  
- The borrowed reference may obtain a different, but safe permission from the reference
  it borrows from. Multiple borrows may be obtained from the same binding,
  so long as the constellation of permissions obtained by all references are safe.
  For example, it is not safe to borrow two `uni` references to the same object.
  
- If the original reference uses a locked permission, the permission's
  _acquire method is invoked to obtain the borrowed reference.
  Later, when the borrowed reference expires (see below), 
  the permission's _release method is invoked.

#### Interior Reference ###

Borrowed references have an extra superpower that region-based references do not have:
a borrowed reference can point to some substructure within an allocated object
(e.g., `&point.x` for a field or `&mat[0,1]` for an array element).
These are called interior references.
Largely, interior borrowed references follow the rules just described.

However, when the reference points within a "dependently-typed" object,
[single-threaded data race safety issues](/post/interior-references-and-shared-mutable/)
can arise in the presence of mutation and multiple references to the same object.
This arises when mutation changes the shape of the object, 
invalidating a borrowed interior reference to the same object.
For example, reducing the size of array can invalidate a interior reference to an
element that no longer exists.
Similarly, if a sum-typed value changes its value to a different variant,
an interior reference pointing to the previous value (of a different type)
will be invalid to use.

To protect against this, an interior reference can only be obtained
from a dependently-typed reference only if its permission is `imm` or `uni`.
If multiple interior references are borrowed at the same time, they must all
be to non-dependent interior types. 
Because of these restrictions, and while the interior borrowed references are alive,
it is then impossible to change the shape of the data.
An array may not be resized and a value may not be mutated to a different variant type value.

### Reference Death ###

The compiler knows when a reference expires. If it is held by a local variable,
it dies when the scope of that local variable expires.
If it is held by some data object, it expires when that data object dies (and is freed).
It can also expire when a reference-bearing field or array element is mutated.

Some regions need to know when each reference dies.
This is accomplished by invoking the region's _dealias method, if defined.

When a single-owner reference dies, we know there are no more references.
Thus, the object can be finalized and freed:

    fn _dealias():
        self.drop()
		
When a ref-counted reference dies, we decrement the counter and
finalize/free the object if the counter drops to 0:

    fn _dealias():
        if --self.counter == 0:
            self.drop()

Tracing GC, arena and pool regions do not use (or define) _alias and _dealias methods.

### Object Death ###

For memory safety, an object must stay alive as any reference to it exists and is usable
for dereferencing by the program. Once the last such reference dies, so may the object
at any time thereafter. For single-owner and ref-counting references,
this happens immediately. For other regions, there may be some delay until
it can be determined that all such references are gone.

Regardless of the region, the death of an object goes through several stages:

1. De-alias any references held within the object's fields or elements.
   This *may* trigger the death of other objects recursively.
   
2. Implicitly run the object type's finalizer on the object.
   Most types do not need, and will not have, a finalizer.
   Finalizers are only required when the object can accumulate
   a dependency on other system objects that need to be
   untangled before the object goes away.
   
3. The region's _free method, if it exists, is invoked to actually
   reclaim the memory used by the object.

#### Explicit Single-owner Death ####

The single-owner region has a special power:
It desired, the program may decide when the object dies.
This is done by allowing the program to invoke the `drop` method
on its only reference. With any other region, this is not allowed.

This capability also means that parameters may be passed to
the drop method and values may be returned. These parameters are
passed to the finalizer, which also then responsible for returning
any expected value. Like other methods, `final` may be overloaded.
Note that implicit finalization will pass no parameters and
return no values.

#### Arena Deaths ####

Arena objects do not die individually. They all die at once when the arena dies.
Obviously, this can happen rather quickly and easily if none of the
objects has a finalizer or holds any references to a different region.

If an arena allows objects that require some finalization logic,
this is why we allowed the region's _allocate to obtain runtime type
information. This can be captured as part of a linked-list in the arena,
and then fired prior to the arena being freed.

## Array References ##

Array references are fat pointers that include a pointer to an array of same-sized elements
and an unsigned integer that specifies the number of elements.
Array references behave like regular references in most ways.
However, there are some differences:

- Cannot dereference. Must index (with bounds check)

- Borrow substructures are called slices

## Virtual References ##

Virtual references are fat pointers that include a pointer to some variant type object
and another pointer to a vtable that maps the location of the certain fields and methods in that object.
Virtual references behave like regular references in most ways.
However, there are some differences:

- Cannot dereference, but can access (indirectly) certain fields and methods.

- Borrow cannot substructure


## Static and Dynamic Regions ##

Most need static state and sometimes global functions.

### Tracing Garbage Collection ###

Collector that does the tracing to mark and then sweep.

Need tracing maps/functions, safepoints, root set, stack tracing, RTTI.

### First-class Regions ###

Integer (non-pointer) references used by first-class
arenas or pools, as their mechanisms work differently in important ways.
Lifetime generativity on return values.
