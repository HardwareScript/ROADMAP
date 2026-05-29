# System 6: Ecosystem & Tooling - Implementation Plan

**System**: Ecosystem & Tooling  
**Status**: ❌ NOT STARTED  
**Migration Strategy**: **BUILD NEW** (Unified .hw architecture)  
**Based On**: v0.1.4 Documentation  
**Prerequisites**: System 1 ✅ **COMPLETE**, Systems 2-5 ⏭️ **PENDING**  
**Last Updated**: March 24, 2026

---

## Overview

The ecosystem and tooling system provides the developer experience layer for Hardware Script. This includes the package manager (hpm), CLI tools (hwc), IDE integration, and the unified 3-file architecture.

**What's New in v0.1.4**:
- Unified `.hw` extension (not 10+ file types)
- Single parser for all `define` blocks
- Two-tier package system (`@std/` + HPM Registry)
- Native SI unit parsing (no YAML)
- Simplified IDE integration (one grammar)
- LLM integration via `.hw-llm-index.md`

**Architecture**:
- hpm: Package manager (install, publish, search, update)
- hwc: Compiler CLI (build, verify, test, generate)
- LSP: Language Server Protocol for IDE integration
- VS Code Extension: Syntax highlighting and IntelliSense
- hwsd: Documentation generator
- CI/CD: GitHub Actions integration

**v0.1.4 Integration Points**:
- ✅ System 1 provides unified parser for all `define` blocks
- ✅ Symbol Table enables go-to-definition and find-references
- ✅ Native SI unit parsing simplifies syntax highlighting
- ✅ 3-file system (`hw.toml`, `hw.lock`, `.hw`) reduces complexity

**What System 1 Provides**:
- `Parser::parse() -> Result<Program, ParseError>` - Unified parser
- `SymbolTable` - All definitions registered
- `Lexer::tokenize()` - Native SI unit tokens
- Error diagnostics with `miette`

**Reference**: See `SYSTEM-1-IMPLEMENTATION-PLAN.md` for parser architecture

---

## Migration Strategy: Build New

### ✅ **Keep from v0.1.3**:
- Core CLI command structure
- Package registry architecture
- Semantic versioning logic
- Cryptographic hash validation

### 🆕 **Build New for v0.1.4**:
- Unified `.hw` file handling (not 10+ parsers)
- LSP for all `define` blocks
- VS Code grammar for unified syntax
- `hw.toml` manifest parser
- `.hw-llm-index.md` generator

### ❌ **Remove from v0.1.3**:
- Separate file type parsers (`.hwmat`, `.hwp`, `.hwx`, etc.)
- YAML dependencies
- Multiple grammar files for IDE

---

## Phase 1: Project Structure & Manifest ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - The 3-File Architecture

**Current Status**: No project structure implementation  
**Target Status**: `hw.toml` and `hw.lock` handling complete

**Prerequisites**: ✅ System 1 complete - Parser available

### 1.1 hw.toml Manifest Parser ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "hw.toml (Project Manifest)"

**Manifest Structure**:
```toml
[project]
name = "robot-controller"
version = "1.0.0"
scale = "PCB"
author = "Your Name"
license = "MIT"

[dependencies]
"@power/5v-regulator" = "1.2.0"
"@sensors/imu-module" = "2.0.1"

[build]
default_target = "pcb"
optimization_level = 2

[features]
pro_model = ["extra_ram", "bluetooth"]
lite_model = []
default = ["lite_model"]
```

**Changes Required**:
- [ ] Create `hwc/crates/hwc-manifest` crate
- [ ] Implement TOML parser for `hw.toml`
- [ ] Define `ProjectManifest` struct
- [ ] Validate project metadata
- [ ] Parse dependency specifications
- [ ] Parse build configuration
- [ ] Parse feature flags
- [ ] Handle semantic versioning
- [ ] Create unit tests

**Estimated Impact**: 300+ lines new code

**Files to Create**:
- `hwc/crates/hwc-manifest/src/lib.rs`
- `hwc/crates/hwc-manifest/src/manifest.rs`
- `hwc/crates/hwc-manifest/src/dependency.rs`
- `hwc/crates/hwc-manifest/Cargo.toml`

**Dependencies**:
- `toml` crate for TOML parsing
- `semver` crate for version handling

### 1.2 hw.lock Lockfile Handler ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "hw.lock (Lockfile)"

**Lockfile Structure**:
```toml
[[package]]
name = "@power/5v-regulator"
version = "1.2.0"
checksum = "sha256:abc123..."
dependencies = []

[[package]]
name = "@sensors/imu-module"
version = "2.0.1"
checksum = "sha256:def456..."
dependencies = ["@power/5v-regulator"]
```

**Changes Required**:
- [ ] Define `Lockfile` struct
- [ ] Implement lockfile generation
- [ ] Implement lockfile parsing
- [ ] Calculate package checksums (SHA-256)
- [ ] Resolve dependency tree
- [ ] Detect circular dependencies
- [ ] Handle lockfile updates
- [ ] Create unit tests

**Estimated Impact**: 200+ lines new code

**Files to Create**:
- `hwc/crates/hwc-manifest/src/lockfile.rs`

### 1.3 Project Initialization ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Project Organization"

**Changes Required**:
- [ ] Implement `hpm init` command
- [ ] Generate default `hw.toml`
- [ ] Generate `.hw-llm-index.md`
- [ ] Create default project structure
- [ ] Generate example `.hw` files
- [ ] Create `.gitignore` for Hardware Script
- [ ] Create unit tests

**Estimated Impact**: 150+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/init.rs`

---

## Phase 2: Package Manager (hpm) ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - Package Management

**Current Status**: No package manager implementation  
**Target Status**: hpm commands working with registry

**Prerequisites**: ✅ Phase 1 complete - Manifest handling

### 2.1 Package Registry Client ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Package Management (hpm)"

**Changes Required**:
- [ ] Create `hwc/crates/hwc-registry` crate
- [ ] Implement HTTP client for registry API
- [ ] Handle authentication tokens
- [ ] Implement package search
- [ ] Implement package download
- [ ] Implement package upload
- [ ] Handle self-hosted registries
- [ ] Implement caching layer
- [ ] Create unit tests

**Estimated Impact**: 400+ lines new code

**Files to Create**:
- `hwc/crates/hwc-registry/src/lib.rs`
- `hwc/crates/hwc-registry/src/client.rs`
- `hwc/crates/hwc-registry/src/auth.rs`
- `hwc/crates/hwc-registry/Cargo.toml`

**Dependencies**:
- `reqwest` for HTTP client
- `serde_json` for API responses

### 2.2 Standard Library (@std/) ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "The Two-Tier System"

**Changes Required**:
- [ ] Create standard library structure
- [ ] Define `@std/materials` (Copper, FR4, etc.)
- [ ] Define `@std/components` (Resistor, Capacitor, etc.)
- [ ] Define `@std/logic` (NAND, NOR, etc.)
- [ ] Bundle with compiler
- [ ] Implement offline resolution
- [ ] Create documentation
- [ ] Create unit tests

**Estimated Impact**: 500+ lines new code

**Files to Create**:
- `hwc/stdlib/materials.hw`
- `hwc/stdlib/components.hw`
- `hwc/stdlib/logic.hw`
- `hwc/crates/hwc-stdlib/src/lib.rs`

### 2.3 hpm Commands ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Commands"

**Commands to Implement**:
- [ ] `hpm init` - Initialize new project
- [ ] `hpm search` - Search registry
- [ ] `hpm install` - Install package
- [ ] `hpm update` - Update dependencies
- [ ] `hpm publish` - Publish package
- [ ] `hpm login` - Authenticate with registry
- [ ] `hpm logout` - Clear authentication
- [ ] Create integration tests

**Estimated Impact**: 600+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/search.rs`
- `hwc/crates/hwc-cli/src/commands/install.rs`
- `hwc/crates/hwc-cli/src/commands/update.rs`
- `hwc/crates/hwc-cli/src/commands/publish.rs`

### 2.4 Dependency Resolution ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Registry Security & CI/CD"

**Changes Required**:
- [ ] Implement dependency graph builder
- [ ] Implement version resolution algorithm
- [ ] Handle version conflicts
- [ ] Detect circular dependencies
- [ ] Generate `hw.lock` from resolution
- [ ] Validate checksums
- [ ] Create unit tests

**Estimated Impact**: 300+ lines new code

**Files to Create**:
- `hwc/crates/hwc-manifest/src/resolver.rs`

---

## Phase 3: Compiler CLI (hwc) ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - Compilation Targets

**Current Status**: Basic CLI exists from v0.1.3  
**Target Status**: Full hwc command suite with unified `.hw` handling

**Prerequisites**: ✅ System 1 complete - Parser available

### 3.1 hwc build Command ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Compilation Targets"

**Changes Required**:
- [ ] Implement `hwc build` command
- [ ] Support `--target` flag (pcb, fpga, asic, spice, viz)
- [ ] Load `hw.toml` manifest
- [ ] Resolve dependencies from `hw.lock`
- [ ] Parse all `.hw` files in project
- [ ] Build Symbol Table from all definitions
- [ ] Compile to Hardware IR
- [ ] Generate target-specific outputs
- [ ] Create integration tests

**Estimated Impact**: 400+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/build.rs`

### 3.2 hwc verify Command ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "CI/CD Integration"

**Changes Required**:
- [ ] Implement `hwc verify` command
- [ ] Run physics validation
- [ ] Run DRC checks
- [ ] Run constraint validation
- [ ] Generate verification report
- [ ] Exit with appropriate status codes
- [ ] Create integration tests

**Estimated Impact**: 200+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/verify.rs`

### 3.3 hwc test Command ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "CI/CD Integration"
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "6. Test Bench Definition"

**Changes Required**:
- [ ] Implement `hwc test` command
- [ ] Parse `define test` blocks
- [ ] Execute test benches
- [ ] Validate assertions
- [ ] Generate test report
- [ ] Support test filtering
- [ ] Create integration tests

**Estimated Impact**: 300+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/test.rs`
- `hwc/crates/hwc-test/src/lib.rs`

### 3.4 hwc generate Command ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Generated Outputs"

**Changes Required**:
- [ ] Implement `hwc generate` command
- [ ] Support format selection (gerber, gdsii, obj, glb, bom, etc.)
- [ ] Generate manufacturing files
- [ ] Generate 3D models
- [ ] Generate BOM
- [ ] Generate documentation
- [ ] Create integration tests

**Estimated Impact**: 250+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/generate.rs`

---

## Phase 4: IDE Integration (LSP) ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - IDE Integration

**Current Status**: No LSP implementation  
**Target Status**: Full LSP with unified `.hw` support

**Prerequisites**: ✅ System 1 complete - Parser and Symbol Table available

### 4.1 Language Server Protocol ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Language Server Protocol (LSP)"

**LSP Features to Implement**:
- [ ] Create `hwc/crates/hwc-lsp` crate
- [ ] Implement LSP server
- [ ] Syntax validation (real-time)
- [ ] Semantic analysis
- [ ] Go-to-definition
- [ ] Find all references
- [ ] Hover information
- [ ] Code completion
- [ ] Rename symbol
- [ ] Document symbols
- [ ] Workspace symbols
- [ ] Code actions (quick fixes)
- [ ] Create integration tests

**Estimated Impact**: 800+ lines new code

**Files to Create**:
- `hwc/crates/hwc-lsp/src/lib.rs`
- `hwc/crates/hwc-lsp/src/server.rs`
- `hwc/crates/hwc-lsp/src/handlers.rs`
- `hwc/crates/hwc-lsp/Cargo.toml`

**Dependencies**:
- `tower-lsp` for LSP protocol
- `tokio` for async runtime

### 4.2 Semantic Tokens ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "VS Code Extension"

**Changes Required**:
- [ ] Implement semantic token provider
- [ ] Token types (keyword, type, variable, function, etc.)
- [ ] Token modifiers (definition, readonly, etc.)
- [ ] Highlight all `define` blocks
- [ ] Highlight SI units
- [ ] Highlight coordinates
- [ ] Create unit tests

**Estimated Impact**: 200+ lines new code

**Files to Create**:
- `hwc/crates/hwc-lsp/src/semantic_tokens.rs`

### 4.3 Diagnostics ❌ NOT STARTED

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) Section: "Error Handling"

**Changes Required**:
- [ ] Implement diagnostic provider
- [ ] Convert parser errors to LSP diagnostics
- [ ] Convert compiler errors to LSP diagnostics
- [ ] Convert physics errors to LSP diagnostics
- [ ] Support error codes (S-codes, C-codes, R-codes, P-codes, M-codes)
- [ ] Provide quick fixes
- [ ] Create unit tests

**Estimated Impact**: 250+ lines new code

**Files to Create**:
- `hwc/crates/hwc-lsp/src/diagnostics.rs`

---

## Phase 5: VS Code Extension ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - VS Code Extension

**Current Status**: No VS Code extension  
**Target Status**: Full extension with syntax highlighting and LSP

**Prerequisites**: ✅ Phase 4 complete - LSP server available

### 5.1 Extension Scaffold ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "VS Code Extension"

**Changes Required**:
- [ ] Create VS Code extension project
- [ ] Set up TypeScript build
- [ ] Configure extension manifest
- [ ] Implement extension activation
- [ ] Connect to LSP server
- [ ] Create extension tests

**Estimated Impact**: 300+ lines new code (TypeScript)

**Files to Create**:
- `vscode-extension/package.json`
- `vscode-extension/src/extension.ts`
- `vscode-extension/src/client.ts`

### 5.2 Syntax Highlighting ❌ NOT STARTED

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) - Complete syntax

**Changes Required**:
- [ ] Create TextMate grammar for `.hw` files
- [ ] Highlight all `define` blocks
- [ ] Highlight keywords (lowercase only)
- [ ] Highlight SI units (case-sensitive)
- [ ] Highlight strings and comments
- [ ] Highlight coordinates
- [ ] Highlight operators
- [ ] Create grammar tests

**Estimated Impact**: 400+ lines (JSON grammar)

**Files to Create**:
- `vscode-extension/syntaxes/hardware-script.tmLanguage.json`

### 5.3 IntelliSense ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "VS Code Extension"

**Changes Required**:
- [ ] Implement completion provider
- [ ] Complete `define` keywords
- [ ] Complete component names
- [ ] Complete material names
- [ ] Complete pin names
- [ ] Complete SI units
- [ ] Snippet support
- [ ] Create tests

**Estimated Impact**: 200+ lines (TypeScript)

**Files to Create**:
- `vscode-extension/src/completion.ts`
- `vscode-extension/snippets/hardware-script.json`

### 5.4 3D Preview Panel ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "VS Code Extension"

**Changes Required**:
- [ ] Implement webview panel
- [ ] Integrate Three.js for 3D rendering
- [ ] Load GLB models from build output
- [ ] Support camera controls
- [ ] Support layer visibility
- [ ] Auto-refresh on build
- [ ] Create tests

**Estimated Impact**: 500+ lines (TypeScript + HTML)

**Files to Create**:
- `vscode-extension/src/preview.ts`
- `vscode-extension/media/preview.html`

---

## Phase 6: Documentation System (hwsd) ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - Documentation system

**Current Status**: No documentation generator  
**Target Status**: hwsd generates HTML docs from `##` comments

**Prerequisites**: ✅ System 1 complete - Parser available

### 6.1 Documentation Parser ❌ NOT STARTED

**Documentation Reference**:
- [LANGUAGE-SPEC.md](../../Docs/v0.1.4/LANGUAGE-SPEC.md) - Comment syntax

**Changes Required**:
- [ ] Create `hwc/crates/hwc-doc` crate
- [ ] Parse `##` documentation comments
- [ ] Extract component documentation
- [ ] Extract module documentation
- [ ] Extract material documentation
- [ ] Build documentation tree
- [ ] Create unit tests

**Estimated Impact**: 300+ lines new code

**Files to Create**:
- `hwc/crates/hwc-doc/src/lib.rs`
- `hwc/crates/hwc-doc/src/parser.rs`

### 6.2 HTML Generator ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Documentation system (hwsd)"

**Changes Required**:
- [ ] Implement HTML template engine
- [ ] Generate component pages
- [ ] Generate module pages
- [ ] Generate material pages
- [ ] Generate index page
- [ ] Support search functionality
- [ ] Create integration tests

**Estimated Impact**: 400+ lines new code

**Files to Create**:
- `hwc/crates/hwc-doc/src/html.rs`
- `hwc/crates/hwc-doc/templates/`

### 6.3 hwsd CLI ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Documentation system (hwsd)"

**Changes Required**:
- [ ] Implement `hwsd generate` command
- [ ] Support output directory configuration
- [ ] Support theme customization
- [ ] Generate static site
- [ ] Create integration tests

**Estimated Impact**: 150+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/doc.rs`

---

## Phase 7: LLM Integration ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - LLM Integration

**Current Status**: No LLM integration  
**Target Status**: `.hw-llm-index.md` generation and CLI query support

**Prerequisites**: ✅ System 1 complete - Parser available

### 7.1 LLM Index Generator ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "LLM Integration"

**Changes Required**:
- [ ] Generate `.hw-llm-index.md` on `hpm init`
- [ ] Include minimal syntax rules
- [ ] Include tri-fold case sensitivity rules
- [ ] Include CLI command references
- [ ] Include project structure overview
- [ ] Create unit tests

**Estimated Impact**: 200+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/llm_index.rs`

### 7.2 Documentation Query CLI ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "LLM Integration"

**Changes Required**:
- [ ] Implement `hpm doc read <name>` command
- [ ] Query documentation for components
- [ ] Query documentation for materials
- [ ] Query documentation for modules
- [ ] Format output for LLM consumption
- [ ] Create integration tests

**Estimated Impact**: 150+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/doc_read.rs`

---

## Phase 8: CI/CD Integration ❌ NOT STARTED

**Reference**: [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) - CI/CD Integration

**Current Status**: No CI/CD templates  
**Target Status**: GitHub Actions templates and examples

**Prerequisites**: ✅ Phase 3 complete - hwc CLI available

### 8.1 GitHub Actions Templates ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "CI/CD Integration"

**Changes Required**:
- [ ] Create GitHub Actions workflow template
- [ ] Support `hwc verify` in CI
- [ ] Support `hwc test` in CI
- [ ] Support `hwc build` in CI
- [ ] Generate artifacts
- [ ] Create example workflows

**Estimated Impact**: 100+ lines (YAML)

**Files to Create**:
- `.github/workflows/hardware-verify.yml` (template)
- `docs/ci-cd-examples.md`

### 8.2 Manufacturing Integration ❌ NOT STARTED

**Documentation Reference**:
- [ECOSYSTEM.md](../../Docs/v0.1.4/ECOSYSTEM.md) Section: "Manufacturing Integration"

**Changes Required**:
- [ ] Implement `hwc quote` command
- [ ] Integrate with JLCPCB API
- [ ] Integrate with PCBWay API
- [ ] Calculate fabrication costs
- [ ] Implement `hwc order` command
- [ ] Handle API authentication
- [ ] Create integration tests

**Estimated Impact**: 400+ lines new code

**Files to Create**:
- `hwc/crates/hwc-cli/src/commands/quote.rs`
- `hwc/crates/hwc-cli/src/commands/order.rs`
- `hwc/crates/hwc-fab/src/lib.rs`

---

## Phase 9: Testing & Verification ❌ NOT STARTED

### 9.1 Unit Tests ❌ NOT STARTED

**Test Categories**:
- [ ] Manifest parsing tests
- [ ] Lockfile generation tests
- [ ] Dependency resolution tests
- [ ] Package registry tests
- [ ] CLI command tests
- [ ] LSP handler tests
- [ ] Documentation generator tests

### 9.2 Integration Tests ❌ NOT STARTED

**Integration Tests**:
- [ ] End-to-end project initialization
- [ ] End-to-end package installation
- [ ] End-to-end build pipeline
- [ ] End-to-end verification
- [ ] LSP integration with VS Code
- [ ] CI/CD workflow execution

### 9.3 Code Quality ❌ NOT STARTED

**Quality Checks**:
- [ ] No clippy warnings
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Examples working

---

## Dependencies

**Requires**:
- ✅ System 1 (Parser) - **COMPLETE**
  - Provides: Unified parser for all `define` blocks
  - Provides: Symbol Table for LSP features
  - Provides: Native SI unit tokens
  - Provides: Error diagnostics with `miette`
- ⏭️ System 2 (Voxel Engine) - **PENDING**
  - Provides: Spatial operations for verification
- ⏭️ System 3 (Routing) - **PENDING**
  - Provides: Routing validation
- ⏭️ System 4 (Physics) - **PENDING**
  - Provides: Physics validation
- ⏭️ System 5 (Export) - **PENDING**
  - Provides: Manufacturing file generation

**Provides**:
- Package management (hpm)
- Compiler CLI (hwc)
- IDE integration (LSP + VS Code)
- Documentation generation (hwsd)
- CI/CD templates
- Developer experience

---

## Key Architectural Changes: v0.1.3 → v0.1.4

| Aspect | v0.1.3 | v0.1.4 |
|--------|--------|--------|
| **File Extensions** | 10+ types | 3 types (`hw.toml`, `hw.lock`, `.hw`) |
| **Parser** | Separate parsers per file type | Unified parser for all `define` blocks |
| **IDE Grammar** | 10+ grammar files | 1 grammar file |
| **LSP Complexity** | Multiple file handlers | Single `.hw` handler |
| **Package System** | Single-tier | Two-tier (`@std/` + HPM) |
| **Unit Parsing** | YAML values | Native SI units |
| **LLM Integration** | None | `.hw-llm-index.md` |

---

## Estimated Timeline

**Phase 1**: Project Structure & Manifest - 1 week  
**Phase 2**: Package Manager (hpm) - 2 weeks  
**Phase 3**: Compiler CLI (hwc) - 2 weeks  
**Phase 4**: IDE Integration (LSP) - 2 weeks  
**Phase 5**: VS Code Extension - 2 weeks  
**Phase 6**: Documentation System - 1 week  
**Phase 7**: LLM Integration - 1 week  
**Phase 8**: CI/CD Integration - 1 week  
**Phase 9**: Testing & Verification - 1 week  

**Total**: 13 weeks

---

## Notes

**Unified Architecture**: The 3-file system (`hw.toml`, `hw.lock`, `.hw`) dramatically simplifies the ecosystem compared to v0.1.3's 10+ file types.

**LSP Benefits**: Single grammar and parser means LSP implementation is straightforward - no context switching between file types.

**Two-Tier Packages**: Standard library ships with compiler (offline), HPM registry provides community packages (online).

**LLM-First**: `.hw-llm-index.md` enables zero-shot code generation by AI assistants without pre-training data.

**Manufacturing Integration**: Direct API integration with PCB fabs enables automated ordering from CLI.
