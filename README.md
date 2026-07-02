# SRAM Optimization

**Project:** DRT Optimization for GC-eDRAM Memory  
**Authors:** Eyal Schiff, Yaniv Terner  
**Supervisor:** Roman Golman (Prof. Adam Teman's Research Group)

---

## 📌 The Challenge: The Volatility of Embedded Memories

In modern System-on-Chip (SoC) architectures, embedded memories are crucial. While standard SRAM is fast, it is area-intensive (requiring 6 transistors per cell). Gain-Cell embedded DRAM (GC-eDRAM) offers a much higher density alternative by storing data as an electrical charge on a capacitor.

However, GC-eDRAM suffers from inherent volatility. The stored charge dissipates over time due to unavoidable leakage currents in nanometer CMOS processes, primarily:

*   **Sub-threshold Leakage:** Current flowing even when the write transistor is "OFF".
*   **Gate Leakage:** Electrons tunneling through the thin gate oxide.
*   **Junction Leakage:** Leakage through reverse-biased PN junctions.

The time it takes for a cell's voltage to drop below a readable threshold is the **Data Retention Time (DRT)**.

### The "Refresh Penalty"

Due to manufacturing process variations (such as Random Dopant Fluctuation), the DRT is not uniform across the memory array. While most cells can hold data for a long time, a statistical "tail" of weak cells loses charge very quickly. To guarantee 100% data integrity, the *entire* memory macro must be continuously refreshed at a frequency dictated solely by these few weakest cells. This creates a massive "Refresh Penalty," leading to severe static power consumption and reduced memory bandwidth.

---

## 💡 The Solution: Smart Memory Controller

This project introduces a power-optimized architecture that breaks the dependency on the weakest cells. By isolating the "tail cells" and managing them separately, the vast majority of the GC-eDRAM array can operate at a significantly slower, highly power-efficient refresh rate.

We designed a hierarchical, smart memory controller in SystemVerilog that integrates a large GC-eDRAM array with a small, static SRAM and a Fully-Associative Look-Up Table (LUT).

### System Architecture & Block Diagram

The smart wrapper manages data routing, address multiplexing, and adaptive power-gating logic according to the following hardware block layout:

![Block Diagram](OPT_DRT_GC_DRAM_SCHEMATIC.png)

1. **OPT_GC_DRAM_4096 (Top-Level Controller):** The main control wrapper handling external CPU handshake signals.
2. **gc_edram_macro_4096:** A structural 4096-line eDRAM array, divided into 8 interleaved banks of 512 lines each.
3. **gc_lut_512:** A pure-combinational Look-Up Table that stores the exact addresses of the 512 worst-case DRT cells.
4. **gc_sram_512:** A standard 512x64 static RAM block that acts as stable, non-volatile storage for the payload of those weak addresses.

---

## ⚙️ Operational Logic & Implementation

To bridge physical circuit constraints with architectural design, we developed a digital behavioral Verilog model that accurately emulates the DRT decay process using dedicated counters and statistical threshold arrays. The smart controller operates transparently to the CPU using the following logic:

### 1. Parallel Write Operation
When a write operation is triggered, the target address is evaluated by the LUT.
*   To prevent critical-path delays, data is written simultaneously to both the GC-eDRAM and the SRAM.
*   Inside the eDRAM model, a decay counter is reset to 0, representing the full recharging of the physical Storage Node.

### 2. Multiplexed Read Routing
The read path utilizes zero-latency combinational logic:
*   **MISS (hit = 0):** The address belongs to a "strong" cell. The controller fetches the data directly from the GC-eDRAM array.
*   **HIT (hit = 1):** The address belongs to a "weak" cell. Even if the data inside the eDRAM has naturally decayed to an unknown state (1'bx), the controller toggles an output multiplexer to flawlessly fetch the intact data from the SRAM.

---

## 📊 Silicon Profiling & Statistical Distribution

To evaluate systemic performance, a global 32K Monte Carlo single-cell simulation array was aggregated into a physical 512 × 64-bit hardware grid. Because a single failing bit compromises an entire row, each row's retention capability is strictly bounded by its weakest component.

### Single-Cell Decay Dynamics
The waveform below illustrates the transient discharge profile of an unmanaged single memory node, highlighting the severe sub-threshold voltage drop over time:

![Single-Cell Decay](cell_DRT.png)

### Row-Level DRT Array Distribution
By profiling the rows based on their worst-case inner bits and importing the results back into Cadence Virtuoso Maestro, we extract the row-level Gaussian bell curve. This distribution clearly shows the statistical simulation spectrum and the decay profile on the left tail, which our architecture effectively neutralizes:

![Statistical Distribution](mc_DRT_16384.png)

---

## 🧪 Simulation & Verification Results

A robust verification environment was built to prove the architecture's reliability. We utilized Python to generate statistical DRT distributions, mimicking Monte Carlo variations, and injected these parameters hierarchically into the deep structural banks.

The testbenches actively monitor the volatility of the cells. By injecting strategic time delays (exceeding the known DRT thresholds of the weak cells), we proved that while the internal eDRAM data decays to a corrupted 'X' state, the Top-Level controller successfully masks this failure and outputs the correct data from the SRAM bypass.

### Successful Integration Waveform
Below is the SimVision waveform demonstrating the successful operation of the Top-Level controller, showcasing the decay of the internal eDRAM node and the intact data retrieval via the SRAM:

![Simulation Waveform](OPT_DRT_GC_DRAM_4096_waveform.png)
