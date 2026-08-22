# DWARF Operation to Access Call Frame Entry Registers

## Problem Description

`DW_OP_entry_value` was added in DWARF5 to make it possible to debug
optimized code. The concept is fine but is difficult for certain kinds
of consumers to implement.

If you are an advanced consumer like systemtap and you know ahead of
time what variables you are going to inspect at which locations, you
can add an implicit breakpoint at the beginning of a function and
capture the value at the locations specified by `DW_OP_entry_value`
and then use it later when evaluating a location which has the the
operator DW_OP_entry_value. However, if you are a general purpose
debugger and the user is stepping through code, you may not know that
the variable the user was going to inspect had a location list entry
which included `DW_OP_entry_value` and so you may not have known to
set an implicit breakpoint at the beginning of the function to capture
the entry value that might be used later.

This problem makes generally implementing locations which have
`DW_OP_entry_value` practically impossible for many kinds of commonly
used consumers nearly impossible. There is one exception though, The
way that GDB currently gets around this limitation is if the location
specified with `DW_OP_entry_value` is only `DW_OP_regN`, `DW_OP_bregN`
or `DW_OP_regval_type`, then gdb can unwind the stack, recover a
usable value from the previous frame, and evaluate the location
expression that contains the `DW_OP_entry_value`. If the expression
following `DW_OP_entry_value` is anything but those very simple
expressions, a general purpose consumer without prior knowledge of
which variables may need to be inspected and often times when, cannot
implement expressions which include `DW_OP_entry_value`.

Because of that limitation GCC and other producers do not emit any
DWARF expression in `DW_OP_entry_value` expression block. More than
99.99% of the time, the expression block is simply
`DW_OP_regN`. Consumers have never been able to interpret a generic
DWARF expression in a `DW_OP_entry_value` expression block.

## Proposed Solution

We propose adding a new operator `DW_OP_call_frame_entry_reg`. This
would take a single ULEB128 inline parameter as a register
number. Then it would use CFI to push the current location in the
current frame of that register in the previous frame.

This operator is implementable in most consumers because it is finite
in scope. It only handlse situations which can be handled with CFI. It
does not expect any arbitrary DWARF expression to be evaluated.

### Keep DW_OP_entry_value

We are not proposing that `DW_OP_entry_value` be removed. It certainly
will continue to be needed to provide backward compatibility. However,
we think that because of the challenges of implementing it, it should
be reserved for exceptional cases where `DW_OP_call_frame_entry_reg`
is not sufficient.

### Saves space

In more than 99.99% of the current cases, this would reduce the size
of DWARF. While `DW_OP_entry_value` <block size> `DW_OP_regN` would
take 3 bytes. `DW_OP_call_frame_entry_reg` would only take 2
bytes. Many programs currently have 10s of thousands of
`DW_OP_entry_value` `DW_OP_regN` combinations.

Because we now have locations on the stack, the cases where the the
expression block is currently `DW_OP_regval_type` could be replaced
with `DW_OP_call_frame_entry_reg` `DW_OP_deref_type` also saving a
byte. This situation is currently much less common.

The other case which is currently served by an expression block that
is `DW_OP_bregN` could be replaced by `DW_OP_call_frame_entry_reg`
`DW_OP_deref` `DW_OP_plus_uconst`. It should be noted that in the
millions of cases where `DW_OP_entry_value` is used that I have
inspected there are only about 1000 cases where `DW_OP_bregN` is used
and in all cases it has been a positive number. However, if there is a
case where a negative offset is needed, then
`DW_OP_call_frame_entry_reg` `DW_OP_deref``DW_OP_consts` `DW_OP_plus`
could be used with just a one byte penalty.

### Potentially easier for producers.

For either the current uses of `DW_OP_entry_value` or
`DW_OP_call_frame_entry_reg` to work, they must have good CFI
information. The explicit use of `DW_OP_call_frame_entry_reg` in a
location informs the producer that the normally caller-saved register
must be included in .eh_frame or .debug_frame for the range of PCs
where the location includes `DW_OP_call_frame_entry_reg`.

### Why is this coming from the GPU group.

When a DWARF expression involving the set of SIMT lanes active on
entry to a subprogram is required, the SIMT active lane mask may be
held in a register that is modified as the subprogram
executes. However, when a function returns the active lane mask
register must be restored to the value it had on entry.

The Call Frame Information (CFI) already encodes such register saving,
so it is more efficient to provide an operation to return the location
of a saved register than have to generate a loclist to describe the
same information.

Adding this operator allows us to address this problem in many
consumers without having to address the implementability problems
inherent in `DW_OP_entry_value`.

## PROPOSAL

In Section 3.6 Conxtext Query Operations add the following operation
after `DW_OP_push_lane`:

> 5.  `DW_OP_call_frame_entry_reg`([ULEB] R)
>
>     → <[location] register's location>
>
>     `DW_OP_call_frame_entry_reg` has a single ULEB128 integer
>     operand that represents a target architecture register number R.
>
>     It pushes a location description that holds the value of
>     register R on entry to the current subprogram as defined by the
>     call frame information (see Section 7.4 "Call Frame
>     Information").
>
>     If there is no call frame information defined, then the default
>     rules for the target architecture are used. If the register rule
>     is <i>undefined</i>, then the undefined location description is
>     pushed.  If the register rule is <i>same value</i>, then a
>     register location description for R is pushed.
>
>     *Producers are reminded that there must be a corresponding CFI
>     entry for the specified register spanning the range of addresses
>     for which the location containing `DW_OP_call_frame_entry_reg`
>     applies. Therefore while the .debug_frame and the .eh_frame have
>     historically mostly preserved the location of callee-saved
>     registers, this operation often requires register rules for
>     caller-saved registers as well. (See Section 7.4.2.3)*

In Section 3.17 Special Operations when describing `DW_OP_entry_value`
replace the paragraph that says:

>     *The register location expression provides a more compact form
>     for the case where the value was in a register on entry to the
>     subprogram.*

With:

>     *In cases where the value of a register in the previous frame
>     can provide the entry value of the
>     variable. `DW_OP_call_frame_entry_reg` should be preferred. It
>     is more compact and easier for consumers to
>     implement. `DW_OP_entry_value` should be reserved for situations
>     where referring to the previous frame is not sufficient. The
>     DWARF5 expression `DW_OP_entry_value` [`DW_OP_regN`] can be
>     replaced more succinctly by `DW_OP_call_frame_entry_reg` N.
>     Also `DW_OP_entry_value` [`DW_OP_regval_type` reg type] can be
>     replaced more succinctly with `DW_OP_call_frame_entry_reg` N
>     `DW_OP_deref_type`. And finally, `DW_OP_entry_value`
>     [`DW_OP_bregN`] can be replaced with
>     `DW_OP_call_frame_entry_reg` N `DW_OP_deref` `DW_OP_plus_uconst`
>     in almost all cases.*

Then edit the following paragraph as follows:

>    *The values needed to evaluate DW_OP_entry_value could be
>    obtained in several ways. The consumer could suspend execution on
>    entry to the subprogram, record values needed by
>    `DW_OP_entry_value` expressions within the subprogram, and then
>    continue; when evaluating `DW_OP_entry_value`, the consumer would
>    use these recorded values rather than the current values. Or,
>    when evaluating <ins>legacy uses of</ins> `DW_OP_entry_value`,
>    the consumer could virtually unwind using the Call Frame
>    Information (see Section 7.4 on page 200) to recover register
>    values that might have been clobbered since the subprogram entry
>    point.*

In Section 7.4.2 Call Frame Instructions, Change the following
sentence to "The DWARF operations that can be used in E have the
following restrictions:".

Add the following bullet to the end of the list:

> * `DW_OP_call_frame_entry_reg` is not allowed if evaluating E causes a
>   circular dependency between `DW_OP_call_frame_entry_reg` operations.
>
>    *For example, if a register R1 has a `DW_CFA_def_cfa_expression`
>    instruction that evaluates a `DW_OP_call_frame_entry_reg`
>    operation that specifies register R2, and register R2 has a
>    `DW_CFA_def_cfa_expression` instruction that that evaluates a
>    `DW_OP_call_frame_entry_reg` operation that specifies register R1.*

In Section 8.7.1 "Operation Expressions" of [Allow location
description on the DWARF evaluation stack], add the following row to
Table 8.9 "DWARF Operation Encodings":

    ---------------------------------------------------------------------------

    Table 8.9: DWARF Operation Encodings
    ================================== ===== ======== =========================
    Operation                          Code  Number   Notes
                                             of
                                             Operands
    ================================== ===== ======== =========================
    DW_OP_call_frame_entry_reg         TBA      1     ULEB128 register number
    ================================== ===== ======== =========================

    ---------------------------------------------------------------------------
