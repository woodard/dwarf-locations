# Operator handling of overlapping locations

## Introduction

According to Section 2.5 Location Lists have long been able to have
overlapping PCs. Since multiple location was only the end result of
evaluating a location list and no further action could be taken on the
result, it was thought that the question about what to do when a
location list yielded overlapping PC could be avoided by saying that
the value stored is the same in all locations.

There are a handful of cases where this turns out to not be true. 

The first one `DW_OP_push_object`_location never specified how to
handle a case when an object's location list evaluated to multiple
overlapping PCs. Another case is when `DW_OP_call` referenced a DIE
that had a location list attribute.`

Furthermore, debugger information entry attributes such as
`DW_AT_data_member_location`, `DW_AT_use_location`, and
`DW_AT_vtable_elem_location` are defined as pushing a location
description on the expression stack before evaluating the expression.

However now that locations can be on the DWARF expression stack, the
question about how DWARF operators behave when applied to multiple
locations must be confronted.

The following proposal defines how operators behave when dealing with
multiple locations such as when a location list has overlapping
PCs.

## Discussion

Fortunately relatively few operators need to be modified to
handle this situation. For example the arithmatic operators and the
logical operators operate on values rather than operating on
locations so they are not affected.

Similarly, most operators that create locations need not worry about
the potential for handling multiple locations because they have been
designed to only create locations that reference a single
location. One exception is `DW_OP_push_object`_location which can
create a location that contains multiple locations by referencing an
object whose location is a location list with overlapping PC
ranges. Similarly, the `DW_OP_call*` operators can also reference a DIE
which has a location list attribute.

The operators which potentially need to be modified are those which
consume a location off of the stack. However, as they were specified
in DWARF5 the derefencing operators currently found in section 3.13 do
not need to be modified because of the guarentee that the value stored is
the same in all locations.

This leaves the new operators which were designed to operate with
locations on the stack: `DW_OP_offset`, `DW_OP_bit_offset` and
`DW_OP_overlay`.

### The problem with avoiding overlapping PC ranges.

Since the questions about what to do with overlapping PC ranges have
not been settled. Both producers and consumers have often avoided the
problems by not emitting overlapping PC ranges. Producers frequently
only track the most local location for a variable and consumers often
only look for one location and operate on that.

The problem can happen if an optimization pass copies a variable from
a memory location to a register solely for performance reasons and
does not provide overlapping PC ranges, then it can create problems
for consumers.

If the only location that the producer provides is the memory
location, then if a debugger stops within the PC range in which the
variable is in a register, then if a user instructs the consumer to
change the value of the variable, then prints it out, the value will
have changed but the code will not be referencing the changed
value.

If on the other hand the producer provides only the register copy of
the variable but that range of instructions doesn't modify the
variable, it may not write the variable back to memory. Thus a user
may chage the variable and the consumer writes it to the register but
since the register is not written back to memory, the value reverts to
its original value.

**Insert graphical example**

This is known as the liveness problem.

### The problem with implicit conversion

While a single memory location in the default address space can be
implicitly converted into a value. When a location contains multiple
locations, implict conversion to a single value is impossible and so
implicit conversion is not done. This is one of the reasons why
`DW_OP_offset` was added rather than overloading `DW_OP_plus` and
`DW_OP_minus` to work on locations.

## Proposal

**Implicit Conversion**
**Offset**
**push_object location**
**call**
**overlay**
