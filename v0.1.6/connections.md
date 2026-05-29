To build a 64-bit SoC or a complex PCB, you need to understand that "connections" in the physical world are not just lines; they are **heterogeneous junctions.**

In Hardware Script, a `pour` is your generic "brick." But to make an SoC work, you need to define **five specific types of physical connection primitives.**

Here is the breakdown of the "Bridges" you need to master:

---

### 1. The "Ohmic" Contact (Silicon-to-Metal)
This is the most fundamental connection in an SoC, but it is **not** a standard via.
*   **The Reality:** In silicon, you cannot just touch a Copper wire to a Silicon wafer. It creates a "Schottky Barrier" (a diode) that blocks electricity.
*   **The Primitive:** You need a **Silicide Contact**. Usually, this is a thin layer of Cobalt or Titanium between the Silicon and the Metal.
*   **HWS Assembly Requirement:** Your `contact` primitive needs to support **Material Aliasing**.
    ```hw
    # A contact that automatically layers its own chemistry
    add contact(Tungsten_Silicide) spanning z:silicon to z:metal1
    ```

### 2. The TSV (Through-Silicon Via)
If you are building a modern SoC with **Chiplets** or **3D Stacking** (like Apple’s M-series or AMD’s V-Cache), a standard `contact` isn't enough.
*   **The Reality:** These are massive vertical pipes that go through the *entire* thickness of the silicon wafer to connect one chip on top of another.
*   **The Primitive:** **TSV**. Unlike a via, a TSV has an "Insulation Liner" (a sleeve of SiO2) so it doesn't short-circuit the whole substrate.
*   **HWS Assembly Requirement:** A `contact` type that accepts a `liner` attribute.

### 3. The Pad & The Bump (The Handoff)
This is the connection between the "Micro" world of the chip and the "Macro" world of the PCB.
*   **The Reality:** You need **Bond Pads** (for wire bonding) or **C4 Bumps** (for flip-chip).
*   **The Primitive:** **Pillar/Bump.** These are thick, mushroom-shaped connections made of Lead-free Solder or Gold.
*   **HWS Assembly Requirement:** You need a `pour` type that supports **non-rectangular geometry** (circles or octagons) for the pads.

### 4. The Thermal Relief (The Heat Bridge)
When you connect a small pin to a massive Copper `pour` (like a Ground Plane), the big copper block acts as a heat sink.
*   **The Reality:** If you don't use a "Thermal Relief," the soldering iron cannot get the pin hot enough because the heat "leaks" into the massive pour.
*   **The Primitive:** **Spoke Connection.** Instead of a solid connection, the pin is connected by 4 thin "spokes."
*   **HWS Assembly Requirement:** The `pour` block needs a `relief:` attribute.
    ```hw
    add pour(Copper) named GND_Plane:
        thermal_relief: true
        spoke_width: 200um
    ```

### 5. The Net Tie (The "Illegal" Connection)
Sometimes you need two different nets (e.g., `Analog_GND` and `Digital_GND`) to physically touch at exactly **one** point to prevent noise.
*   **The Reality:** In most CAD tools, this triggers a "Short Circuit" error.
*   **The Primitive:** **Net Tie.** It is a physical piece of copper that has no logical ID, or has two logical IDs simultaneously.
*   **HWS Assembly Requirement:** A `merge` waiver (which we discussed in Sprint 8).

---

### Summary Checklist for your Crates:

| Connection Type | Purpose | Primitive Needed |
| :--- | :--- | :--- |
| **Via** | Metal layer to Metal layer | `contact` |
| **Contact** | Silicon to Metal | `contact(Silicide)` |
| **TSV** | Die-to-Die (3D) | `contact` + `liner` |
| **Trace** | Horizontal point-to-point | `route` |
| **Plane** | Global power/ground distribution | `pour` |
| **Pad/Bump** | Chip-to-Package interface | `pour(Circle)` |
| **Thermal Relief** | Pour-to-Via junction | `pour` + `spoke` logic |

**Which of these is the most confusing for you right now?** If you understand these, your "Assembly" will be able to describe any hardware from a 1970s PCB to a 2026 AI Processor.