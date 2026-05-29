# Hardware Script - IDE Integration Architecture

**Document Type**: Engineering & Ecosystem Architecture  
**Status**: Official Specification  
**Last Updated**: April 2026  

---

## The Philosophy: Respecting the User's RAM

Modern Language Servers (like `rust-analyzer` or `elixir-ls`) are notoriously heavy, often consuming gigabytes of RAM to provide real-time autocomplete and error checking. They do this by constantly running complex borrow-checkers, expanding macros, or spinning up virtual machines on every keystroke.

Hardware Script takes a different approach, modeled after the blazing-fast performance of **TypeScript**. 

To provide the ultimate developer experience while keeping resource usage under 50MB of RAM, the Hardware Script IDE integration is strictly divided into **Two Modular Tiers**. Users have the absolute freedom to install exactly what they need.

---

## Tier 1: The Syntax & Formatting Extension (Zero-Cost)

The first tier is a completely "dumb," zero-computation extension. It requires no background binaries, consumes zero background RAM, and runs natively inside the text editor's (e.g., VS Code) core engine.

### What it provides:
1. **Semantic Highlighting:** Beautiful color-coding for all Hardware Script keywords (`define`, `route`, `module`, `space`).
2. **Tri-Fold Case Sensitivity Rules:** Highlights SI units (`mV`, `kΩ`) differently from user identifiers (`ESP32`) and keywords.
3. **Bracket Matching & Folding:** Allows users to collapse `logic:` or `layout:` blocks easily.
4. **Code Snippets:** Typing `defcomp` and hitting Tab instantly generates the boilerplate for a `define component` block.

### How it works under the hood:
It does not parse the actual code structure. It uses **TextMate Grammars** (a series of highly optimized Regular Expressions stored in a `.tmLanguage.json` file). 

**Why separate this?**
If a user is on an underpowered laptop, or just wants to quickly read a `.hw` file on the web (like in GitHub.dev), they only need this extension. It makes the code readable instantly without downloading or running the Hardware Script compiler.

---

## Tier 2: The Language Server Protocol (LSP) (Lightweight Intelligence)

For users who want the full "compiler-as-you-type" experience, they can install the **Hardware Script LSP**. This is a standalone Rust binary (`hwc-lsp`) that runs in the background and communicates with the IDE via standard JSON-RPC.

### The "Short-Circuit" Boundary (The Secret to Speed)
The Hardware Script Compiler has a 5-Layer pipeline. If the LSP ran the 3D Voxel Engine (Layer 3) and the Physics DRC (Layer 4) on every keystroke, the computer would freeze. 

To achieve sub-50ms response times, the `hwc-lsp` implements a **Strict Short-Circuit Boundary**:
* **It ONLY runs Pass 1 (Lexing/Parsing) and Pass 2 (Symbol Resolution & Logic Synthesis).**
* It checks if your syntax is valid.
* It checks if the wires in your `logic:` block exist (Electrical Borrow Checker).
* It checks if imported components (`Resistor_0805`) actually exist in your Symbol Table.
* **It NEVER runs the geometry router or physics engine.** Those only run when the user explicitly triggers a full build (`hwc build`).

### Key Features of the `hwc-lsp`:

#### 1. Instant Autocomplete (O(1) Symbol Table Lookup)
Because Pass 1 builds a highly optimized Rust `HashMap` (the Symbol Table), autocomplete is instant. When a user types `add `, the LSP simply returns the keys of the Symbol Table. It knows exactly what components, modules, and materials are available in the current scope.

#### 2. Reusing `miette` Diagnostics
We do not write separate error messages for the IDE. Because the core compiler uses the `miette` crate, every error already contains exact byte-spans, line numbers, column numbers, and human-readable hints. 
When the user makes a mistake (e.g., `L01: Unbound Wire`), the LSP extracts the `miette` span and tells VS Code exactly where to draw the red squiggle. Hovering over the squiggle displays the exact compiler hint.

#### 3. The 300ms Debouncer
The LSP does not parse the file on every single keystroke. It uses a **Debouncer**. It waits until the user stops typing for 300 milliseconds before running the parser. This single architectural decision reduces CPU usage by over 90%.

### How to Build it (For Contributors)
The LSP is built using the industry-standard **`tower-lsp`** Rust crate. It acts as a lightweight wrapper around the existing `hwc-parser` and `hwc-compiler` crates.

---

## How They Work Together

A standard professional setup uses both:
1. The user opens `main.hw`. The **Tier 1 Syntax Extension** colors the text instantly.
2. The user types `route MainPower to `.
3. After 300ms of no typing, the **Tier 2 LSP** parses the file, checks the Symbol Table, and pops up an autocomplete box showing all valid pins (e.g., `ESP32.VIN`, `LED1.Anode`).
4. If the user types a non-existent pin, the LSP catches the `C12: Pin does not exist` error, extracts the coordinates, and underlines the mistake in red.

---

## Future Roadmap: Hardware Script Studio (Native IDE)

While VS Code extensions serve the immediate developer ecosystem, the ultimate goal of the project is to provide a fully integrated, zero-configuration Native IDE: **Hardware Script Studio**.

### The "Build Once, Run Anywhere" Strategy
Hardware Script Studio will not be a web-based SaaS requiring servers, logins, or cloud hosting. It will be a native desktop application compiled from a single Rust codebase.

* **The Tech Stack:** Built using Rust-native UI frameworks (like `egui` + `wgpu` or Tauri). 
* **The Viewport:** Because the UI is built in Rust, it can natively integrate the 3D Voxel Engine to render the `.hwx` binary in real-time, completely offline.
* **Security & Performance:** Runs entirely on the host machine. Ideal for defense contractors, offline development, and maximum privacy. No cloud dependencies.

---

## ⚠️ Commercial Integration Policy (AGPLv3 Notice)

Hardware Script is open-source infrastructure. However, the intelligence of the Language Server is protected intellectual property.

**The Command-Line / LSP Boundary Rule:**
If a proprietary, closed-source company wishes to build their own GUI or IDE for Hardware Script:
1. Utilizing the Tier 1 Syntax Extension (TextMate grammars) is unrestricted.
2. **Connecting to, querying, or piping data to/from the Hardware Script Language Server (`hwc-lsp`) constitutes an intimate integration and forms a combined derivative work.** 
3. Parsing, reading, or visualizing the proprietary `.hwx` simulation binary or internal AST memory structures also constitutes a derivative work.

Proprietary integrations that rely on the `hwc-lsp` or internal representations require a **Commercial Enterprise License**. See `COMMERCIAL-LICENSE.md` for details on how to legally integrate Hardware Script into closed-source commercial EDA software.

---
