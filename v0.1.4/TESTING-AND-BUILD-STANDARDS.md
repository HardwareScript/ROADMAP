# Testing & Build Standards

## Quick Reference

**During development (fast iteration):**
```bash
.\test.bat -p <package-name>    # Test single crate only
```

**Before committing (mandatory):**
```bash
.\test.bat --full --workspace   # Test entire workspace
```

---

## Development Workflow

1. **While building:** Test only the crate you're working on
   ```bash
   .\test.bat -p hwc-parser
   .\test.bat -p hwc-engine
   ```

2. **Before committing:** Test the entire workspace to catch integration issues
   ```bash
   .\test.bat --full --workspace
   ```

**Why test the whole workspace?**
- Changes in one crate can break others
- Integration issues only appear when testing all crates together
- Ensures the entire system works as a cohesive unit

---

## Strict Clippy Configuration

All lints configured in `hwc/Cargo.toml`:

```toml
[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "warn", priority = -1 }
nursery = { level = "warn", priority = -1 }
cargo = { level = "warn", priority = -1 }

# Critical for compiler reliability
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
```

**Key enforcements:**
- No `unwrap()`, `expect()`, or `panic()` - use proper error handling
- All warnings treated as errors with `-D warnings`
- Checks library code, tests, benchmarks, and examples with `--all-targets`

---

## Common Issues & Fixes

**Unwrap/Expect:**
```rust
// ❌ Bad
let value = some_option.unwrap();

// ✅ Good
let value = some_option.ok_or_else(|| Error::MissingValue)?;
```

**Unnecessary Clones:**
```rust
// ❌ Bad
fn process(data: String) { }
process(my_string.clone());

// ✅ Good
fn process(data: &str) { }
process(&my_string);
```

---

## Test Writing Best Practices

### Always Parse Real Source Code

**Standard**: Parse real Hardware Script source in tests, don't manually construct AST.

```rust
// ✅ Good
use hwc_parser::{Lexer, Parser};

fn parse(source: &str) -> Result<hwc_parser::Program, String> {
    let lexer = Lexer::new(source);
    let tokens = lexer.tokenize().map_err(|e| format!("Lex error: {:?}", e))?;
    let mut parser = Parser::new(tokens);
    parser.parse().map_err(|e| format!("Parse error: {:?}", e))
}

#[test]
fn test_compile_space() {
    let source = r#"define space "TestBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
"#;
    
    let program = parse(source).expect("Failed to parse");
    let ir = compile_to_ir(&program, source.to_string()).unwrap();
    assert_eq!(ir.space_name, "TestBoard");
}
```

**Why?**
- Tests the full pipeline (lexer → parser → compiler)
- Automatically fills in span fields for error reporting
- More readable and maintainable
- Validates real user-facing syntax

---

## Why These Standards Matter

**For Compiler Development:**
- Catches bugs before they reach hardware
- Prevents costly manufacturing errors
- Ensures deterministic, reliable compilation
- Maintains code quality as the project scales

**Your compiler translates code to physical hardware. Bugs aren't just inconvenient - they're expensive and potentially dangerous.**

---
