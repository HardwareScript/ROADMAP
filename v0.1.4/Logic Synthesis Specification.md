Here is the complete, totally rewritten **Logic Synthesis Specification**. 

It strips away all the old Verilog and functional programming baggage. It officially locks in the **"Custom Rust"** philosophy (strict, bare-metal truth, explicit `let`, `Reg()`, `match`, `else:`), wrapped in a visually silent, punctuation-free syntax (no `;`, `{}`, `::`, `=>`, or `_`).

--- START OF FILE LOGIC-SYNTHESIS-SPECIFICATION.md ---

# Hardware Script Compiler - Logic Synthesis Specification (v0.4.0)

**Document Type:** Implementation Guide & Syntax Specification  
**Module:** Layer 2 (Comptime Unrolling & Behavioral Synthesis)  
**Status:** Finalized Core Syntax 

---

## 1. The Philosophy: Custom Rust (Bare-Metal Truth)

Hardware Script enforces a strict **Rule of Duality**:
1. **The Macro Level (Spaces, Routing, Modules):** Uses *Custom Ruby*. It is descriptive, spatial, and declarative.
2. **The Micro Level (The `logic:` block):** Uses *Custom Rust*. Logic is electricity and mathematics. It requires strict typing, explicit state, and bare-metal truth.

We do not use synthetic "magic" (like automatic state machines or hidden clock triggers). We give the user the ultimate programming primitives (`struct`, `enum`, `match`, `let`), and the "dumb" compiler translates them 1-to-1 into physical silicon gates, multiplexers, and flip-flops.

**The Visual Rule:** Zero visual garbage. No semicolons `;`, no braces `{}`, no double colons `::`, no fat arrows `=>`, and no cryptic underscores `_`. Indentation handles scope. Colons `:` handle blocks.

---

## 2. Data Structures: `enum` and `struct`

Hardware requires aggressive manipulation of wire bundles and binary states. Instead of using raw hex codes and manual bit-slicing (Assembly style), Hardware Script uses strict structures that compile into zero-cost routing abstractions.

### 2.1 `enum` (Named Binary Constants)
Enums assign human-readable names to binary states. The compiler automatically determines the required bit-width to store them. Use a single dot `.` to access them.

```hw
enum CpuState:
    Fetch, Decode, Execute

enum Opcode:
    Add = 0x1, Sub = 0x2, Branch = 0x3
```

### 2.2 `struct` (Wire Bundling)
Structs define the exact bit-layout of a wire bundle. When you cast a raw wire to a struct, the compiler automatically handles the physical wire routing/slicing.

```hw
struct Instruction:
    opcode[4]
    func[4]
    imm[8]
```

---

## 3. The `logic:` Block Core Primitives

Inside a `logic:` block, everything happens concurrently at the speed of light.

### 3.1 `let` (Physical Wires)
The `let` keyword creates a physical wire. 
The `=` operator means **Wiring** (connect output to input).
The `==` operator means **Comparator Gate** (instantiate a logic gate that outputs 1 or 0).

```hw
logic:
    # Cast a raw 16-bit wire into a structured wire bundle
    let instr = RawInstr as Instruction
    
    # Instantiate a Comparator Gate, and wire its output to 'is_add'
    let is_add = (instr.opcode == Opcode.Add)
```

### 3.2 `let mut` (Multiplexers)
The `let mut` keyword defines a wire whose source can change based on conditions. The compiler reads this and automatically instantiates a **Multiplexer (MUX)**.

```hw
logic:
    let mut result = 0
    let mut write_enable = false
```

### 3.3 `Reg()` (Physical Memory / Flip-Flops)
We do not use magical `@Clk` triggers. Memory is a physical component. You instantiate it explicitly using `Reg()`. 
* To read its current state, read the variable (`state`). 
* To wire into its next clock tick (the D-pin), write to `.next` (`state.next`).

```hw
logic:
    # Instantiates a Flip-Flop Register
    let state = Reg(clock: Clk, reset: Rst, init: CpuState.Fetch)
    
    # State Transition (Wiring into the D-pin)
    state.next = CpuState.Decode
```

---

## 4. Control Flow (Hardware Multiplexers)

Hardware cannot execute logic sequentially. Control flow in hardware is a Multiplexer. Hardware Script uses `if / else` and `match` to generate these MUXes.

**Syntax Rule:** All blocks and match arms end in a single colon `:`. The catch-all is the English word `else:`.

### 4.1 The `match` Expression (N-to-1 Multiplexer)
Replaces `switch`/`case`. Perfectly unrolls into an N-to-1 Multiplexer.

```hw
logic:
    result = match instr.func:
        0x0: DataA + DataB
        0x1: DataA - DataB
        0x2: DataA & DataB
        else: 0
```

### 4.2 The `if / else` Expression (2-to-1 Multiplexer)
For standard binary choices.

```hw
logic:
    state.next = if Enable == 1: CpuState.Decode else: CpuState.Fetch
    
    # Block format for multiple mutations
    if state == CpuState.Execute:
        write_enable = true
        result = DataA + DataB
```

---

## 5. Bus Manipulation: Slicing and Bundling

Hardware Script uses clean array syntax `[]` for physical wire manipulation.

```hw
logic:
    # Bit Slicing (Extracting Wires)
    let top_byte = AddressBus[15..8]
    let sign_bit = Value[31]
    
    # Bundling (Combining Wires from MSB to LSB)
    let full_address[16] = [top_byte, LowByte[8]]
    
    # Duplication / Padding
    let padded[16] = [(0 * 12), SmallNum[4]]
```

---

## 6. Implementation Guide (For Compiler Developers)

This section provides the exact specifications needed for the Rust parser and AST expansion passes.

### 6.1 Lexer Tokens (Updates to `token.rs`)

**Remove these tokens:**
`Behavior`, `Always`, `RisingEdge`, `FallingEdge`, `Do`, `End`, `Then`, `Case`, `When`, `Default`, `AtTrigger` (`@`), `Arrow` (`->`, `=>`), `Underscore` (`_`), `DoubleColon` (`::`).

**Add/Ensure these tokens:**
```rust
#[token("logic")]
Logic,

#[token("enum")]
Enum,

#[token("struct")]
Struct,

#[token("let")]
Let,

#[token("mut")]
Mut,

#[token("match")]
Match,

#[token("else")]
Else,

#[token("pass")]
Pass,

#[token("Reg")]
RegisterInit,

#[token("as")]
CastAs,
```

### 6.2 EBNF Grammar

```ebnf
enum_def = "enum" identifier ":" INDENT { identifier [ "=" number ] ","? } DEDENT
struct_def = "struct" identifier ":" INDENT { identifier "[" number "]" } DEDENT

logic_block = "logic" ":" INDENT { logic_statement } DEDENT

logic_statement = 
    | let_decl
    | assignment
    | if_stmt

let_decl = "let" [ "mut" ] identifier [ "[" number "]" ] "=" expression
assignment = identifier [ "." "next" | "[" range "]" ] "=" expression

if_stmt = "if" expression ":" block_or_expr [ "else" ":" block_or_expr ]

expression = 
    | binary_op
    | match_expr
    | bundle_expr
    | reg_init
    | identifier [ "." identifier | "[" range "]" ]
    | number | "true" | "false"
    | expression "as" identifier

match_expr = "match" expression ":" INDENT { match_arm } DEDENT
match_arm = ( expression | "else" ) ":" block_or_expr

block_or_expr = 
    | expression
    | "pass"
    | INDENT { logic_statement } DEDENT

reg_init = "Reg" "(" "clock" ":" expression "," "reset" ":" expression "," "init" ":" expression ")"
```

### 6.3 AST Unrolling Mapping (Pass 2)

When Pass 2 encounters the `logic:` block AST, it performs text-substitution to generate the Physical IR.

**Mapping 1: MUX Generation (`let mut` + `match`)**
*   **Compiler Action:**
    1. Look at all assignments to `result`.
    2. Instantiate a Multiplexer.
    3. Route the `match` condition wire to the Selector pins.
    4. Route the right-hand values to the Data pins.
    5. Route the MUX output to `result`.

**Mapping 2: Register Generation (`Reg()`)**
*   **Compiler Action:**
    1. Detect `Reg(...)`. Look up `D_FlipFlop` in stdlib.
    2. Route the `clock` argument to the `Clk` pin.
    3. Route the `reset` argument to the `Rst` pin.
    4. When `var.next = X` is found, route `X` to the `D` pin.
    5. When `var` is read, read from the `Q` pin.

---

## 7. Complete End-to-End Example (16-Bit CPU Control Unit)

This example proves that nobody needs to touch Verilog again. It compiles 1-to-1 into gates and flip-flops, but reads like perfectly clean Python/Rust software.

```hw
enum CpuState:
    Fetch, Decode, Execute

struct Instruction:
    opcode[4]
    func[4]
    imm[8]

define module "CPU_Control_Unit":
    pins: Clk, Rst, RawInstr[16], DataA[16], DataB[16]
    
    logic:
        # Cast the raw wire to our structural format
        let instr = RawInstr as Instruction
        
        # Instantiate the State Register (Flip-Flops)
        let state = Reg(clock: Clk, reset: Rst, init: CpuState.Fetch)
        
        # Declare mutable wires (Multiplexer outputs)
        let mut result = 0
        let mut write_enable = false
        let mut branch = false
        
        # Core CPU Logic
        if state == CpuState.Execute:
            match instr.opcode:
                0x1: 
                    write_enable = true
                    result = match instr.func:
                        0x0: DataA + DataB
                        0x1: DataA - DataB
                        0x2: DataA & DataB
                        0x3: DataA | DataB
                        0x4: DataA ^ DataB
                        else: 0
                        
                0x2:
                    write_enable = true
                    result = DataA + instr.imm
                    
                0x3:
                    branch = (DataA == DataB)
                    
                else: 
                    pass
                    
        # State Machine Transition (Wired to Register D-Pin)
        state.next = match state:
            CpuState.Fetch:   CpuState.Decode
            CpuState.Decode:  CpuState.Execute
            CpuState.Execute: CpuState.Fetch
```

**Conclusion:** The updated `logic:` block perfectly unifies software readability with strict physical hardware mapping. By stripping away visual clutter and adopting uncompromised bare-metal primitives, it eliminates the need for fragmented logic synthesis tools.

--- END OF FILE LOGIC-SYNTHESIS-SPECIFICATION.md ---