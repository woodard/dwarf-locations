# Part 14: DWARF Operation to Access Call Frame Entry Registers

## PROBLEM DESCRIPTION

As described in [DWARF Operations to Create Vector Composite Location Descriptions], a DWARF expression involving the set of SIMT lanes active on entry to a subprogram is required. The SIMT active lane mask may be held in a register that is modified as the subprogram executes. However, when a function returns the active lane mask register must be restored to the value it had on entry.

The Call Frame Information (CFI) already encodes such register saving, so it is more efficient to provide an operation to return the location of a saved register than have to generate a loclist to describe the same information. This is now possible since [Allow location description on the DWARF evaluation stack] allows location descriptions on the stack.

## PROPOSAL

In Section 2.5.4.4.1 "General Location Description Operations" of [Allow location description on the DWARF evaluation stack], add the following operation after `DW_OP_push_object_address`:

