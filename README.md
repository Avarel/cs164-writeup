# Chapter 6

## Overview
In this assignment, you will be extending your “Loop” language compiler from the last assignment to create the "Vector" language. This language will add the following features:
* Hetereogenous lists or tuples
* Garbage collection

## Textbook Coverage
This assignment is based on chapter 6 of Essentials of Compilation.

## New Language Features
The new language features are as follows:
* `has-type` type annotation constructs
* Expressions
  * `vector` (tuple insantiation)
  * `vector-length` (tuple length access)
  * `vector-ref` (tuple element access)
  * `vector-set!` (tuple element mutation)
* Garbage collection runtime

## Code To Write: The Compiler Passes

### A. Passes from Previous Assignment
Many passes are similar in structure to the previous chapter, with the 
exception of trivial handling of new forms and names of imported modules 
(e.g. `Cloop` changing to `Ctup`).
* `remove-unused`
* `build-interference`
* `remove-jumps`

Some passes have to be extended to handle new forms, but they should be
relatively straightforward.
* `shrink`
* `uniquify`
* `uncover-get`
* `remove-complex`
* `uncover-live`
* `patch-instructions`

One note of interest is that in the `ul` passes, the `deref` location should
consider only their relative registers, not the entire location. This did not
present a problem in previous passes.

| Pass | Changes                                                                                               |
| ---- | ----------------------------------------------------------------------------------------------------- |
| `sh` | Trivial addition of `vector`, `vector-length`, `vector-ref`, `vector-set!`, `has-type`.               |
| `un` | Trivial addition of `vector`, `vector-length`, `vector-ref`, `vector-set!`, `has-type`.               |
| `ea` | **New pass.** More details below.                                                                     |
| `ug` | Trivial addition of `collect`, `allocate`, `globalval`, `vector-length`, `vector-ref`, `vector-set!`. |
| `rc` | Trivial addition of `collect`, `allocate`, `globalval`, `vector-length`, `vector-ref`, `vector-set!`. |
| `ec` | **Nontrivial changes.** More details below.                                                           |
| `ru` | No significant changes.                                                                               |
| `si` | **Nontrivial changes.** More details below.                                                           |
| `ul` | Trivial addition of `andq`, `sarq`.                                                                   |
| `bi` | No significant changes.                                                                               |
| `ar` | **Nontrivial changes.** More details below.                                                           |
| `rj` | No significant changes.                                                                               |
| `pi` | Trivial addition of `andq`, `sarq`.                                                                   |
| `pc` | **Nontrivial changes.** More details below.                                                           |
| `pa` | Provided.                                                                                             |

### B. Expose Allocation
As discussed in the textbook, the `expose-allocation` pass lowers the `vector`
tuple creation form into conditional calls to the allocator. This pass generates
complex operands, so it is placed before the `remove-complex` pass.

The conditional calls to the alloctor take the forms `collect`, `allocate`, and
`globalval`.
* `collect n` runs the garbage collector, making sure that there are `n` bytes
  ready to be allocated.
* `allocate n type` obtains memory for `n` elements (and a 64-bit tag).
* `globalval` reads the value of a global variable, such as `free_ptr` or
  `fromspace_end`.

Type information is obtained from the type checking passes.

The main transform of interest in this pass is as follow:
```s
(HasType
  (Vec ((e0) .. (en)))
  (Vector (ty0 ... tyn)))
```

```s
(Let x0 e0                                                  # 1. prologue
  (...                                                      # 1. prologue
  (Let xn en                                                # 1. prologue
    (If                                                     # 2. cond collect call
      (Prim                                                 # 2. cond collect call
        Lt                                                  # 2. cond collect call
        ((Prim                                              # 2. cond collect call
           Add                                              # 2. cond collect call
           ((GlobalVal free_ptr) (Int <8 * (n + 1)>)))      # 2. cond collect call
         (GlobalVal fromspace_end)))                        # 2. cond collect call
      Void                                                  # 2. cond collect call
      (Collect <8 * (n + 1)>))                              # 2. cond collect call
    (Let                                                    # 3. allocate call
      $ea.1                                                 # 3. allocate call
      (Allocate n (Vector (ty0 ... tyn)))                   # 3. allocate call
      (Let                                                  # 4. tuple initialization
        _.n                                                 # 4. tuple initialization
        (VecSet (Var $ea.1) 0 (Int x0))                     # 4. tuple initialization
        (...                                                # 4. tuple initialization
        (Let                                                # 4. tuple initialization
          _.1                                               # 4. tuple initialization
          (VecSet (Var $ea.1) n (Int xn))                   # 4. tuple initialization
          (Var $ea.1))))))))                                # 4. tuple initialization
```
The transformation lowers the tuple forms into the following parts:
1. Prologue of temporary variable bindings for initializing expressions
2. Conditional calls to `collect`
3. Call to `allocate`
4. Epilogue of initializing the tuple elements

#### Prologue
The prologue calls prior to `allocate` are important, as they may trigger
garbage collection which may not happen while there is an uninitialized tuple
on the heap. Most expressions will not trigger this so it unnecessary to bind
them to another temporary variable in the prologue. The expressions that you
may want to bind in the prologue are:
* `has-type`
* `vector-ref`
* `prim`

#### Conditional Collect Call
The conditional collect call simply checks if there is enough space between the
`free_ptr` and `fromspace_end` for the tuple. If not, it collects the space. The
tuple occupies `8` bytes per element, plus another `8` bytes for the 64-bit tag.
Thus, it occupies `8 * (n + 1)` bytes.

#### Allocate Call
The `allocate` call will be furthered lowered in the `si` pass.

#### Epilogue
The epilogue initializes the tuple elements using a series of `let` statements.
The reason for doing so instead of using `begin` blocks is because optimizations
generally works better with chained `let` rather than `begin`.

### C. Explicate Control
#### `convert_atom` and `convert_exp`
* New forms need to be handled in `convert_atom` and `convert_exp`. This should be straightforward.

#### `explicate_assign`
* The new forms `allocate`, `globalval`, `vector-len`, `vector-ref`, and `vector-set!` forms need to be handled. Just like the `atm` and `prim` forms, they can be delegated to a assignment sequence using `convert_exp`.
* The `collect` form has side effects should be redirected to `explicate_effect`.

#### `explicate_pred`
* `vector-ref` can be used as an expression in the `if` form's condition position. The vector ref should be assigned
  into a temporary `vecref` variable, and then that variable can be handled in `explicate_pred` like a regular `atm` in the condition position.

#### `explicate_effect`
* `globalval`, `vector-len`, `vector-ref` has no effects, and should be handled like `atm` and `prim`.
* `collect` expression should be transformed into the `collect` statement.
* `vector-set!` has effects, and should be transformed into the `vector-set!` statement (using `VecSetS` constructor).
* `allocate` should not be in the effect position.

#### `explicate_tail`
* This function has to be extended to deal with the `allocate`, `globalval`, `vector-len`, `vector-ref`, `vector-set!`, and `collect` forms. This will involve calls to `convert_exp`, `explicate_assign` and/or `explicate_effect` as needed.

### D. Select Instructions

#### Computing Tuple Tags
All tuples have a 64-bit tag associated with them. This tag primarily determines the following:

To handle this, we designate the following bits for the tuple's tag:
```
             ┌────────┬─────────────────────────────────────┬──────────┬─────┐
             │        │         .......                1101 │   000010 │   1 │
             └────────┴─────────────────────────────────────┴──────────┴─────┘
             ┌─────── ┌──────────────────────────────────── ┌───────── ┌──────
Function     Unused   Pointer Mask                          Length     Forward
Bits         63-57    56-7                                  6-1        0
Length       7 bits   50 bits                               6 bits     1 bit
```

- 50 bits are used for the pointer mask.
  - 1 indicates a tuple, 0 indicates other data types.
  - This is so that the garbage collector correctly handles nested tuples.
  - LSB represents lower tuple indices, MSB represents higher tuple indices.
- 6 bits for the length of the tuple.
- 1 bit to signify if the tuple has been copied,
  - This is for the garbage collection runtime we are using.

#### Converting Expressions
Tuples are pointers, so to interact with them, we put the tuple pointers into
the `r11` register and dereference relative to that for simplicity. The `global-value`
argument should transform into a `global-arg` label argument, but that should be
relatively straightforward. The rest of the transformations are listed below.

##### `lhs = (vector-ref tup n);`
```
movq tup′, %r11
movq 8(n + 1)(%r11), lhs′
```

##### `lhs = (vector-set! tup n rhs);`
```
movq tup′, %r11
movq rhs′, 8(n + 1)(%r11)
movq $0, lhs′
```

##### `lhs = (vector-len tup);`
```
movq tup′, %rax
movq (%rax), %rax
andq $0b1111110, %rax
sarq $1, %rax
movq %rax, lhs'
```

##### `lhs = (allocate len (Vector type ... ));`
```
movq free_ptr(%rip), %r11
addq 8(len + 1), free_ptr(%rip)
movq $tag, 0(%r11)
movq %r11, lhs′
```

##### `(collect bytes)`
```
movq %r15, %rdi
movq $bytes, %rsi
callq collect
```

### Allocate Registers
The new thing about this pass besides the trivial addition of `andq` and `sarq`
conversion is that you must now keep track of the number of spills on the root
stack.
- Spills on the root stack become dereferences relative to `%r15`, compared to
  spills on the normal stack that becomes dereferences relative to `%rbp`.
- A variable spills on the root stack if it is a tuple typed variable. This 
  information is provided by the `vvs : VarSet.t` argument passed into
  `varmap_of_colormap`. `vvs` contains all of the tuple typed variable names,
  and is computed by `vector_vars`.
  
### Prelude Conclusion
The new changes in the `prelude_conclusion` pass besides the trivial addition of
`global-arg`, `andq`, and `sarq` assembly conversion is that it must now handle
the spilled arguments on the root stack.
- A call to the GC runtime should be made with `initialize(init_heap_size, init_heap_size)`.
- The global variable `rootstack_begin` should be moved to `%r15`.
- All spills on the root stack should be zero'd out.
- Reserves space for all the root stack spills (a multiple of 16) by modifying the stack pointer `%r15`.
