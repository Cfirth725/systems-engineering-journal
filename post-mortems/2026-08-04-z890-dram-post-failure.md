# System Troubleshooting & Incident Post-Mortem: Z890 DRAM POST Failure

* **Status:** Resolved
* **Date:** 2026-08-04
* **Issue:** Initial POST Failure on Z890 Mini-ITX Build (DRAM LED Handshake Loop)
* **Resolution:** Swapped memory kit for a manufacturer QVL-certified DDR5 kit

---

### 1. Incident Overview

During the initial assembly and bring-up of a high-performance Mini-ITX platform, the system failed to complete the Power-On Self-Test (POST). The system hung indefinitely on the motherboard’s diagnostic DRAM LED, preventing system initialization and display output.

* **Target Hardware:** Gigabyte Z890I AORUS ULTRA (LGA 1851 / Mini-ITX)
* **Processor:** Intel Core Ultra Series 2 (Integrated iGPU)
* **Initial Memory Kit:** Patriot Viper DDR5 32GB Kit (Non-QVL)
* **Status:** Resolved via out-of-band firmware updates and memory profile isolation.

---

### 2. Diagnostic Steps & Systematic Isolation

#### Phase 1: Physical Assembly & Component Isolation
* Verified 24-pin ATX and 8-pin EPS power supply connections.
* Ensured memory modules were fully seated into DIMM slots until physical latches engaged.
* Tested single-stick configurations in isolation to rule out physical socket or memory channel degradation.
* Verified that display output relied on the CPU's integrated iGPU via rear I/O.

#### Phase 2: Firmware Microcode Update
* Utilized Gigabyte Q-Flash Plus to flash the motherboard’s BIOS to release F20 without requiring a successful POST or CPU/RAM boot state.
* Updated board microcode to ensure the memory controller possessed the latest Intel 800-series Memory Reference Code (MRC) training protocols and microcode fixes.

#### Phase 3: POST State & Behavior Analysis
* Observed diagnostic LED behavior during power delivery:
  1. System initiates power-on sequence (CPU LED illuminates briefly).
  2. Diagnostic LED transitions to DRAM as the memory controller attempts initial signal training.
  3. Memory controller fails to negotiate sub-timings, power cycles, and halts on a solid amber DRAM LED without advancing to VGA or BOOT checks.
* **Analysis:** The power reset confirmed physical CPU socket integrity and initial board execution. The halt at the DRAM stage isolated the failure to a memory controller training loop caused by incompatible SPD profile tables.

---

### 3. Root Cause Analysis (RCA)

* **Firmware Handshake Failure:** The Serial Presence Detect (SPD) chip on the non-QVL Patriot kit contained timing profiles that failed to complete memory training handshakes with the Z890 memory controller under the F20 BIOS.
* **Sub-Timing Tolerances:** Intel Z890 memory controllers enforce strict primary and secondary sub-timing parameters during initial boot. Non-validated profile tables cause permanent boot retries.
* **Resolution:** Swapped memory kit for a QVL-certified Corsair Vengeance DDR5 kit (`CMK32GX5M2B6400C36`), featuring native Intel XMP 3.0 profiles explicitly validated by Gigabyte for the Z890I AORUS ULTRA.

---

### 4. Key Takeaways & Engineering Insights

* **Out-of-Band Maintenance:** Demonstrated proficiency using low-level board recovery tools (Q-Flash Plus) to update system microcode independent of POST state.
* **DRAM Diagnostic Logic:** Applied knowledge of sequential POST stages (`CPU → DRAM → VGA → BOOT`) to isolate hardware bottlenecks without relying on display output.
* **QVL & Spec Validation:** Reaffirmed the critical necessity of validating manufacturer Qualified Vendor Lists (QVL) and exact SKU part numbers when working with sensitive, small-form-factor platforms.