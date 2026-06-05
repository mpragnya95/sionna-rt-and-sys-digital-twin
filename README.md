# sionna-rt-and-sys-digital-twin
Radio Map and System Level Throughput Analysis in a Sionna-Based 5G NR Digital Twin

> **Capstone Project** · School of Electrical and Computer Engineering · University of Sydney · 2026  
> Simulation site: **Manado, Indonesia**

---

## Overview

This project implements a **Digital twin** for 5G New Radio (NR) networks by integrating three simulation pipelines:

- **Sionna Ray Tracing (RT)** — site-specific, per-slot propagation modeling  
- **Sionna System-Level (SYS)** — full 5G NR physical-layer abstraction pipeline  
- **SUMO (Simulation of Urban Mobility)** — realistic vehicular trajectory generation

The central research question is:

> *How large is the gap between a static radio-map-based coverage evaluation (propagation only) and a 5G NR system-level digital twin pipeline running per slot — and what causes that gap?*

The project is validated on a 3D scene of the Megamas district, Manado, with 3 Base Stations (BSs) and 3 vehicular User Equipments (UEs) following SUMO-generated routes.

---

## Key Features

| Feature | Details |
|---|---|
| Carrier frequency | 3.5 GHz (FR1) |
| Bandwidth | ~18.36 MHz (612 subcarriers × 30 kHz) |
| BS antenna array | 16-antenna planar array (2×4, cross-polarized, TR 38.901) |
| UE antenna array | 4-antenna planar array (1×2, cross-polarized, dipole) |
| Spatial multiplexing | 4 streams per UE |
| Precoding | Regularized Zero-Forcing (RZF) |
| Equalization | Linear Minimum Mean Square Error (LMMSE) |
| Scheduling | Proportional Fair (PF), multi-UE MIMO |
| Link adaptation | Outer Loop Link Adaptation (OLLA), target BLER = 10% |
| Handover | A3 hysteresis (best-server RSS) |
| Simulation length | 200 outer steps × 100 ms = 20 s |
| NR slot resolution | 0.5 ms (numerology µ=1, 30 kHz SCS) |
| Ray tracing | max_depth=5, LoS + specular + diffraction + refraction |

---

## Repository Structure

```
sionna-v2x-digital-twin/
│
├── capstone_maulana_sionna_final.ipynb   # Main simulation notebook
│
├── manado_scene/
│   └── megamas2.xml                      # Sionna 3D scene (Blender export)
│
├── sumo_projects/
│   └── tutorial1/
│       ├── net.net.xml                   # SUMO road network
│       └── random_routes.rou.xml         # Vehicle route definitions
│
├── figures/                              # Auto-generated output figures
│   └── ray_tracing_mid_route.png
│
├── cell13_results.npz                    # Persisted simulation results (auto-generated)
│
└── README.md
```

> **Note:** `cell13_results.npz` and all figures in `figures/` are generated outputs — they are not required to be committed to the repository.

---

## Dependencies

### Python packages

| Package | Tested version | Notes |
|---|---|---|
| `tensorflow` | ≥ 2.13 | GPU recommended |
| `sionna` | ≥ 0.19 | Includes `sionna.rt`, `sionna.phy`, `sionna.sys` |
| `sumolib` | any | SUMO Python tools |
| `traci` | any | SUMO TraCI interface |
| `numpy` | ≥ 1.24 | |
| `matplotlib` | ≥ 3.7 | |
| `imageio` | ≥ 2.31 | For GIF animation export |

Install with:

```bash
pip install sionna tensorflow numpy matplotlib imageio
pip install sumolib --break-system-packages
```

### External software

- **Eclipse SUMO** ≥ 1.26.0 — must be installed separately.  
  Set `SUMO_HOME` environment variable to your installation path.  
  Download: [https://sumo.dlr.de](https://sumo.dlr.de)

- **GPU** (strongly recommended) — Ray tracing and system-level simulation are computationally intensive.

---

## How to Run

Open `capstone_maulana_sionna_final.ipynb` in Jupyter and run cells **in order**:

### Phase 1 — Setup & Scene

| Cell | Description |
|---|---|
| **Cell 1** | Import all libraries, configure TensorFlow GPU, check SUMO environment |
| **Cell 2** | Load the Megamas 3D scene, add 3 vehicle objects as scatterers |
| **Cell 3** | Set simulation parameters, place 3 BSs, build Stream Management and Resource Grid configs |
| **Cell 4** | Parse SUMO net + routes, convert to Sionna coordinate system, generate per-vehicle waypoints |

### Phase 2 — Radio Map

| Cell | Description |
|---|---|
| **Cell 5** | Ray-path visualization at mid-route (static render + optional 20 s GIF animation) |
| **Cell 6** | Compute the static radio map over the full scene |
| **Cell 7** | Path gain heatmap (best-server) |
| **Cell 8** | RSS (Reference Signal Received Power) heatmap |
| **Cell 9** | SNR heatmap (best-server) |

### Phase 3 — System-Level Simulation

| Cell | Description |
|---|---|
| **Cell 10** | Per-slot Channel Frequency Response (CFR) generation via SUMO + ray tracing |
| **Cell 11** | Instantiate PHY Abstraction, OLLA, and PF Scheduler |
| **Cell 12** | Define per-slot, per-BS step function |
| **Cell 13** | **Main simulation loop** — runs 200 outer steps with dual-rate NR slot processing |
| **Cell 13-SAVE** | Persist all simulation arrays to `cell13_results.npz` *(run once after Cell 13)* |
| **Cell 13-LOAD** | Reload results from disk *(use instead of Cell 13 in subsequent sessions)* |

### Phase 4 — Analysis & Visualization

| Cell | Description |
|---|---|
| **Cell 14** | Per-UE throughput, MCS, and BLER time-series (triple-axis plot) |
| **Cell 15** | Radio-map vs. system-level capacity gap quantification |
| **Cell 16** | Coverage gap bar charts and per-UE throughput CDF overlays |

> **Tip:** After running Cell 13 once and saving with Cell 13-SAVE, all subsequent analysis sessions can skip directly to Cell 13-LOAD → Cell 14 onward.

---

## Outputs

| File | Description |
|---|---|
| `figures/ray_tracing_mid_route.png` | Overhead ray-path render at t = 10 s |
| `ray_tracing_20s_animation.gif` | Ray-tracing animation over full 20 s trajectory |
| `cell13_results.npz` | All simulation history arrays (throughput, MCS, BLER, handover, RSRP, etc.) |
| `throughput_mcs_bler_per_ue.png` | Time-series plot per UE |
| `gap_coverage_per_ue.png` | Coverage gap bar chart (radio map vs. system-level) |
| `gap_cdf_overlay.png` | Per-UE CDF overlay: radio-map estimate vs. system-level actual |

---

## Key Methodology Notes

**Apple-to-apple comparison (Cell 15)**

The radio-map estimate uses **path gain only** (not radio-map SINR) to compute a single-stream attenuated Shannon bound:

```
C_RM = α · B · log₂(1 + SNR)
```

where `α = 0.6` (Mogensen et al., 2007) and `SNR = path_gain_dB + Tx_power_dBm − noise_dBm`.


Samples with `SNR < −10 dB` are excluded as no-coverage outliers.

**Reference:** Mogensen, P. et al. (2007). "LTE Capacity Compared to the Shannon Bound." *IEEE 65th Vehicular Technology Conference (VTC2007-Spring)*, pp. 1234–1238.

---

## Coordinate System

SUMO and Sionna use different coordinate origins. The conversion applied throughout is:

```python
X_SHIFT = 547.205
Y_SHIFT = 667.03

def sumo_to_sionna(x_sumo, y_sumo, z=1.5):
    return [x_sumo - X_SHIFT, y_sumo - Y_SHIFT, z]
```

Vehicle height is fixed at `z = 1.5 m` (rooftop UE).


---

## Acknowledgements

This project uses:
- [NVIDIA Sionna](https://nvlabs.github.io/sionna/) — open-source library for link-level and system-level wireless simulation
- [Eclipse SUMO](https://sumo.dlr.de) — Simulation of Urban Mobility
- Scene geometry derived from OpenStreetMap data for the Megamas district, Manado, Indonesia
