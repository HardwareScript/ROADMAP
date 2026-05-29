# Context-Aware Semantic Parsing

**Date**: 2026-04-21  
**Status**: ARCHITECTURAL DECISION  
**Impact**: Language Design, Parser Architecture, Standard Library Compatibility

---

## The Problem

During implementation of GAP 7 (Progressive Alignment), we initially added pin direction keywords (`input`, `output`, `power`, `ground`, `inout`) as global lexer tokens. This caused immediate conflicts:

1. **Standard Library Breakage**: units.hw and other stdlib files failed to parse
2. **Keyword Pollution**: Common words like `input` and `output` became reserved everywhere
3. **Inflexibility**: Adding new pin roles would require lexer changes

## The Solution: Context-Aware Semantic Parsing

**Core Principle**: Keywords should be **context-specific**, not **globally reserved**.

### Implementation Strategy

**Lexer Level** (Thin):
- `input`, `output`, `power`, `ground`, `inout` are lexed as regular `Identifier` tokens
- NO special keyword tokens for these words
- The lexer stays thin and conflict-free

**Parser Level** (Smart):
- The module parser recognizes these identifiers as property names in context
- When parsing a module body, specific property names trigger specialized parsing logic
- Each block type (module, component, space, etc.) has domain-specific intelligence

### Code Pattern

```rust
// Inside parse_module body:
fn parse_module_body_statement(&mut self) -> Result<ModuleStatement, ParseError> {
    // Peek at the identifier to determine what kind of statement this is
    if let Some(Token::Identifier(name)) = self.current() {
        match name.as_str() {
            // Pin role declarations (context-aware)
            "input" => self.parse_pin_role(PinDirection::Input),
            "output" => self.parse_pin_role(PinDirection::Output),
            "power" => self.parse_pin_role(PinDirection::Power),
            "ground" => self.parse_pin_role(PinDirection::Ground),
            "inout" => self.parse_pin_role(PinDirection::Inout),
            
            // Legacy support
            "pins" => self.parse_pins_list(),
            
            // Other module statements
            "add" => self.parse_add_statement(),
            "route" => self.parse_route_statement(),
            "for" => self.parse_for_loop(),
            "if" => self.parse_if_conditional(),
            "logic" => self.parse_logic_block(),
            
            _ => Err(self.error("Unknown module statement"))
        }
    }
}
```

### Syntax Examples

**Property-Style (Recommended)**:
```hardware
module Inverter_Logic:
    # Context-aware: 'input' is a property name here
    input: VIN
    output: VOUT
    power: VDD
    ground: GND
    
    add NMOS named M1
    route M1.gate to VIN
```

**Legacy Bracket Style (Also Supported)**:
```hardware
module Counter:
    pins: [input Clk, output Q[8]]
    
    logic:
        # ... behavioral logic
```

**Elsewhere in the Language**:
```hardware
# These work fine because 'input' is just an identifier
unit Input:
    symbol: "in"
    dimension: custom

const input = 5  # Variable named 'input'
```

## Benefits

### 1. Zero Keyword Pollution
Words like `input`, `output`, `power`, `ground` can be used freely as:
- Unit names in stdlib
- Variable names in logic blocks
- Property names in other contexts
- Component names

### 2. Standard Library Compatibility
No conflicts with existing stdlib files:
- `units.hw` - Can define units with any name
- `components/*.hw` - Can use these words in metadata
- `materials/*.hw` - No restrictions on property names

### 3. Extensible Architecture
Adding new pin roles is trivial:
- No lexer changes required
- Just add a new case in the module parser
- No risk of breaking existing code

### 4. Parser Intelligence
Each block type has domain-specific knowledge:
- Module parser knows about pin roles
- Component parser knows about electrical properties
- Space parser knows about geometry
- Material parser knows about physical properties

### 5. Follows the Boundary Law
Pin roles are **properties** (declarative facts), not inline keywords:
```hardware
# Property-style (follows Boundary Law)
input: VIN          # Colon indicates a static fact

# NOT inline keywords
pins: [input VIN]   # 'input' would be a keyword here (rejected)
```

## Comparison with Other Languages

### Traditional Approach (Verilog, VHDL)
```verilog
module inverter(
    input wire VIN,
    output wire VOUT,
    inout wire DATA
);
```
- `input`, `output`, `inout` are **global keywords**
- Cannot be used as identifiers anywhere
- Causes namespace pollution

### Hardware Script Approach
```hardware
module Inverter:
    input: VIN
    output: VOUT
    inout: DATA
```
- `input`, `output`, `inout` are **property names** (identifiers)
- Can be used freely elsewhere
- Parser recognizes them in context

## Implementation Checklist

- [x] Remove global keyword tokens from lexer
- [x] Update module parser to recognize property-style pin declarations
- [x] Update AST to store pin directions
- [x] Update logical synthesizer to extract pin directions
- [x] Test with stdlib (units.hw, etc.)
- [x] Update documentation (GAP7, LANGUAGE-SPEC)
- [x] Implement context-aware parsing in module parser
- [x] Update test files to use property-style syntax
- [x] Verify end-to-end alignment validation

## Future Extensions

This pattern can be extended to other contexts:

**Component Blocks**:
```hardware
component Resistor:
    input: A
    output: B
    electrical:
        resistance: 10kΩ
```

**Interface Blocks**:
```hardware
interface SPI:
    input: MISO
    output: MOSI, SCK
    inout: CS
```

**Test Blocks**:
```hardware
test InverterTest:
    input: VIN = [0V, 5V]
    output: VOUT
    assert: VOUT = not VIN
```

## Conclusion

Context-Aware Semantic Parsing is a fundamental architectural principle for Hardware Script. It keeps the language thin at the lexer level while enabling smart, domain-specific parsing at the block level. This approach:

- Prevents keyword pollution
- Maintains stdlib compatibility
- Enables extensibility
- Follows the Boundary Law
- Provides better error messages (context-specific)

This is the "God-Tier" way to design a hardware description language.

---

**Related Documents**:
- [GAP7-PROGRESSIVE-LVS.md](GAP7-PROGRESSIVE-LVS.md) - Alignment validation implementation
- [LANGUAGE-SPEC.md](../../Docs/v0.1.6/LANGUAGE-SPEC.md) - v0.1.6 syntax specification
- [COMPILER-ARCHITECTURE-PHILOSOPHY.md](../../Docs/v0.1.2/COMPILER-ARCHITECTURE-PHILOSOPHY.md) - Compiler design principles
