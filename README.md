# Sounding Analysis Parameters Reference (v2.1)

Complete reference for atmospheric sounding thermodynamic and kinematic calculations used in severe weather forecasting.

---

## Table of Contents

- [Overview](#overview)
- [Constants](#constants)
- [Thermodynamic Calculations](#thermodynamic-calculations)
- [Parcel Theory](#parcel-theory)
- [Kinematic Calculations](#kinematic-calculations)
- [Effective Layer Parameters](#effective-layer-parameters)
- [Composite Parameters](#composite-parameters)
- [Stability Indices](#stability-indices)
- [Downdraft Parameters](#downdraft-parameters)
- [Entraining CAPE](#entraining-cape)
- [Temperature Parameters](#temperature-parameters)
- [Lapse Rates](#lapse-rates)
- [Moisture Parameters](#moisture-parameters)
- [Winter Weather Parameters](#winter-weather-parameters)
- [Fire Weather Indices](#fire-weather-indices)
- [Shear Layers Reference](#shear-layers-reference)
- [Interpolation Methods](#interpolation-methods)
- [Unit Conversions](#unit-conversions)

---

## Overview

This JSON schema defines all parameters, formulas, thresholds, and calculation methods used in a sounding analysis engine. It covers five major categories:

- **Thermodynamic** — parcel lifting, buoyancy, moisture
- **Kinematic** — shear, helicity, storm motion
- **Composite** — multi-parameter indices (SCP, STP, SHIP, etc.)
- **Winter Weather** — precip type, DGZ, snow squall potential
- **Fire Weather** — FFWI, HDWI, RFTI

---

## Constants

Physical constants used throughout all calculations.

| Symbol | Value | Units | Description |
|--------|-------|-------|-------------|
| `g` | 9.81 | m/s² | Gravitational acceleration |
| `Rd` | 287.04 | J/kg/K | Dry air gas constant |
| `Rv` | 461.5 | J/kg/K | Water vapor gas constant |
| `Cp_d` | 1005 | J/kg/K | Specific heat of dry air |
| `Cp_v` | 1870 | J/kg/K | Specific heat of water vapor |
| `Cp_l` | 4190 | J/kg/K | Specific heat of liquid water |
| `Cp_i` | 2106 | J/kg/K | Specific heat of ice |
| `Lv` | 2,501,000 | J/kg | Latent heat of vaporization |
| `Lf` | 333,000 | J/kg | Latent heat of fusion |
| `Ls` | 2,834,000 | J/kg | Latent heat of sublimation |
| `T_trip` | 273.15 | K | Triple point temperature |
| `ε` | 0.62197 | — | Rd/Rv ratio |
| `κ` | 0.28571 | — | Rd/Cp (Poisson constant) |

---

## Thermodynamic Calculations

### Saturation Vapor Pressure (`es`)
- **Method:** Bolton (1980) polynomial approximation
- **Formula:** `es = 6.112 * exp(17.67 * T / (T + 243.5))`
- **Input:** Temperature (°C) → **Output:** hPa

### Mixing Ratio (`w`)
- **Formula:** `w = ε * e / (p - e)` where `e = es(Td)`
- **Input:** Pressure (hPa), Dewpoint (°C) → **Output:** g/kg

### Virtual Temperature (`Tv`)
- **Formula:** `Tv = T * (1 + w/ε) / (1 + w)`
- Note: `Tv ≥ T` always; accounts for density reduction from water vapor

### Potential Temperature (`θ`)
- **Formula:** `θ = T * (p₀/p)^κ` where `κ ≈ 0.286`
- Reference pressure `p₀` defaults to 1000 hPa

### Equivalent Potential Temperature (`θe`)
- **Method:** Lift parcel to LCL, then follow moist adiabat to reference level
- **Formula:** `θe = θ_dry * exp(Lv * ws / (Cp * T_LCL))`

### Wet Bulb Temperature (`Tw`)
- **Method:** Normand's rule (iterative) — find LCL, descend moist adiabatically to original pressure

### Lifted Condensation Level (LCL)
- **Approximation:** `LCL_height ≈ 125 * (T - Td)` meters
- **Method:** Intersection of dry adiabat from T with saturation mixing ratio line from Td
- **Output:** Height (m AGL), Pressure (hPa), Temperature (°C)

### Moist Adiabatic Lapse Rate (`Γm`)
- **Formula:** `Γm = g * (1 + Lv*ws/(Rd*T)) / (Cp + Lv²*ws/(Rv*T²))`
- **Typical range:** 4–7 °C/km depending on temperature and moisture

---

## Parcel Theory

### Parcel Types

| Type | Abbreviation | Starting Level |
|------|-------------|----------------|
| Surface-Based | SB | Surface (lowest observation) |
| Mixed-Layer | ML | Mean θ and mixing ratio of lowest 100 hPa |
| Most-Unstable | MU | Level of max θe in lowest 300 hPa |
| Effective | EFF | Layer where CAPE ≥ 100 J/kg and CIN ≥ −250 J/kg |

### CAPE (Convective Available Potential Energy)
- **Formula:** `CAPE = g * ∫[LFC→EL] (Tv_parcel - Tv_env) / Tv_env dz`
- **Method:** Trapezoidal integration from LFC to EL

| Variant | Description |
|---------|-------------|
| SBCAPE | Surface-based |
| MLCAPE | Mixed-layer (100 hPa mean parcel) |
| MUCAPE | Most-unstable |
| CAPE3 | 0–3 km layer CAPE |
| HGZCAPE | Hail Growth Zone CAPE (−10°C to −30°C) |

**Thresholds:** Marginal <1000 / Moderate 1000–2500 / High 2500–4000 / Extreme >4000 J/kg

### CIN (Convective Inhibition)
- **Formula:** `CIN = g * ∫[sfc→LFC] (Tv_env - Tv_parcel) / Tv_env dz`
- Expressed as negative values; represents energy barrier to convection
- **Thresholds:** Weak >−50 / Moderate −50 to −200 / Strong <−200 J/kg

### NCAPE (Normalized CAPE)
- **Formula:** `NCAPE = CAPE / (EL - LFC)` (m/s²)
- Represents average buoyancy across the cloud-bearing layer

### Key Levels
- **LFC:** Level of Free Convection — where parcel first becomes warmer than environment
- **EL:** Equilibrium Level — where parcel temp equals environment above LFC
- **MPL:** Maximum Parcel Level — maximum height parcel reaches using all CAPE

---

## Kinematic Calculations

### Storm Motion — Bunkers Internal Dynamics Method
**Reference:** Bunkers et al. (2000)

1. Calculate pressure-weighted mean wind, surface to 6 km
2. Calculate 0–6 km shear vector
3. Apply 7.5 m/s deviation perpendicular to shear

```
RM = V_mean + (7.5 / |shear|) * (shear_y, -shear_x)
LM = V_mean - (7.5 / |shear|) * (shear_y, -shear_x)
```

**Deviant Tornado Motion:**
- `DTMR = (V_0-500m + V_RM) / 2`
- `DTML = (V_0-500m + V_LM) / 2`

### Storm Relative Helicity (SRH)
- **Formula:** `SRH = Σ[(u₁-Cx)(v₂-Cy) - (u₂-Cx)(v₁-Cy)]`
- **Unit:** m²/s²

| Layer | Usage |
|-------|-------|
| SRH 0–500 m | Low-level tornado parameter |
| SRH 0–1 km | Most common fixed-layer |
| SRH 0–3 km | Deep inflow layer |
| ESRH | Effective inflow layer |

**Thresholds:** Weak <100 / Moderate 100–300 / Strong 300–500 / Extreme >500 m²/s²

### Bulk Wind Difference (BWD / Shear)
- **Formula:** `BWD = V(top) - V(bottom)`
- **Unit:** knots or m/s

| Layer | Description |
|-------|-------------|
| BWD 0–6 km | Deep-layer shear (supercell/tornado primary) |
| BWD 0–1 km | Low-level shear |
| EBWD | Effective layer shear |

**0–6 km thresholds:** Weak <20 kt / Moderate 20–40 kt / Strong 40–60 kt / Extreme >60 kt

---

## Effective Layer Parameters

The **effective inflow layer** is the layer where parcels have sufficient buoyancy (CAPE ≥ 100 J/kg, CIN ≥ −250 J/kg) to sustain convection.

- **EBWD:** Shear vector across effective inflow layer; falls back to 0–6 km shear if no effective layer
- **ESRH:** SRH integrated through effective inflow layer

---

## Composite Parameters

### Supercell Composite Parameter (SCP)
```
SCP = (MUCAPE/1000) × (ESRH/50) × (EBWD/20)
```
EBWD clamped to [10, 20] m/s. **Thresholds:** >1 conditional / >4 favorable / >10 strongly favorable

### Significant Tornado Parameter — Fixed Layer (STP)
```
STP = (SBCAPE/1500) × (SRH_0-1km/150) × (EBWD/20) × LCL_term
```
- `LCL_term`: 1.0 if LCL <1000 m; 0.0 if LCL >2000 m; linear between
- **Thresholds:** >1 conditional / >3 significant / >5 high

### Significant Tornado Parameter — Effective Layer (STP_eff)
```
STP_eff = (MLCAPE/1500) × (ESRH/150) × (EBWD/20) × LCL_term × CIN_term
```
- `CIN_term`: 1.0 if MLCIN >−50; 0.0 if MLCIN <−200; linear between
- EBWD: <12.5 → 0; >30 → 1.5

### Significant Hail Parameter (SHIP)
```
SHIP = (MUCAPE × LR_700-500 × -T500 × MixRat × EBWD) / 42,000,000
```
Conditional multipliers applied if MUCAPE <1300 J/kg, LR_700-500 <5.8 °C/km, or freezing level <2400 m.
**Thresholds:** >0.5 conditional / >1.0 favorable / >1.5 significant

### Wind Damage Parameter (WNDG)
```
WNDG = (LR_0-3km/9) × ((50+MLCIN)/40) × (MLCAPE/2000) × (MeanWind_1-3.5km/15)
```
LR_0-3km set to 0 if <7 °C/km; MLCIN capped at −50 J/kg.

### Derecho Composite Parameter (DCP)
```
DCP = (DCAPE/980) × (MUCAPE/2000) × (EBWD/20) × (MeanWind_0-6km/16)
```

### Energy Helicity Index (EHI)
```
EHI = (CAPE × SRH) / 160,000
```
Available for 0–500 m, 0–1 km, 0–3 km, and effective layers.
**Thresholds:** >1 conditional / >2 favorable / >3 significant

### TEHI (Tornadic EHI)
```
TEHI = (SRH_0-1km × MLCAPE / 160,000) × (CAPE3/200) × (BWD_0-6km/20)
```
Set to 0 if MLCIN <−100, SBCIN <−200, or MLLCL >1700 m.

### TTS (Tornado Tilting and Stretching)
For low-CAPE, high-shear environments.
```
TTS = (SRH_0-1km × CAPE3 / 6500) × CAPE_term × (BWD_0-6km/20)
```
CAPE3 clamped to [0, 150] J/kg. Same zero-out conditions as TEHI.

### Significant Severe (SigSvr)
```
SigSvr = MLCAPE × EBWD   (m³/s³)
```
Threshold: >30,000 m³/s³

---

## Stability Indices

### K-Index (KI)
```
KI = (T850 - T500) + Td850 - (T700 - Td700)
```
<20 unlikely / 20–25 isolated / 26–30 widely scattered / 31–35 scattered / >35 numerous

### Total Totals (TT)
```
TT = (T850 - T500) + (Td850 - T500)
```
<44 unlikely / 44–50 possible / 51–52 isolated severe / 53–56 scattered severe / >56 significant

### Lifted Index (LI)
```
LI = T_env(500 mb) - T_parcel(500 mb)
```
>0 stable / 0 to −3 marginal / −3 to −6 moderate / −6 to −9 very / <−9 extreme

### Bulk Richardson Number (BRN)
```
BRN = CAPE / (0.5 × Shear²)
```
Shear = mean wind 0–6 km minus mean wind 0–500 m.
<10 supercells likely / 10–45 favorable balance / >45 multicells likely

---

## Downdraft Parameters

### DCAPE (Downdraft CAPE)
1. Find minimum θe in lowest 400 hPa
2. Start parcel at that level's wet-bulb temperature
3. Descend moist adiabatically to surface, integrate negative buoyancy

**Thresholds:** Weak <500 / Moderate 500–1000 / Strong >1000 J/kg

### Downdraft Temperature (DownT)
Estimated surface temperature of downdraft air in °F. Lower values → stronger cold pool potential.

---

## Entraining CAPE

### ECAPE (Peters et al. 2020)
CAPE accounting for entrainment of environmental air into the updraft.

| Parameter | Value | Description |
|-----------|-------|-------------|
| σ | 1.6 | Updraft width |
| α | 0.8 | Entrainment coefficient |
| L_mix | 120 m | Mixing length |
| Pr | 0.333 | Turbulent Prandtl number |

```
ε = 0.18 × α² × π² × L_mix / (Pr × σ² × EL)
```

**SR Wind Integral:** Storm-relative wind speed integrated from surface to EL (m²/s) — drives entrainment suppression.

---

## Temperature Parameters

| Parameter | Symbol | Unit | Description |
|-----------|--------|------|-------------|
| Convective Temperature | ConvT | °F | Surface temp needed to initiate surface-based convection |
| Maximum Temperature | MaxT | °F | Forecast max temp via surface parcel lifted 150 hPa |
| Maximum Wet Bulb | MaxWB | °C | Max wet-bulb in lowest 400 hPa |
| Freezing Level | FrzLvl | m AGL | Height of 0°C isotherm |
| LCL Temperature | — | °F | Temperature at the LCL |

---

## Lapse Rates

**Height-based:** `Γ = -(T_top - T_bottom) / (z_top - z_bottom)` (°C/km)

**Pressure-based:** `Γ = (T_lower - T_upper) / (z_upper - z_lower)` (°C/km)

Standard layers: LR 0–3 km, LR 3–6 km, LR 0–6 km, LR 700–500 hPa, LR 850–500 hPa

| Regime | Value |
|--------|-------|
| Dry adiabatic | 9.8 °C/km |
| Moist adiabatic | 4–7 °C/km |
| Absolutely stable | <6 °C/km |
| Conditionally unstable | 6–9.8 °C/km |
| Absolutely unstable | >9.8 °C/km |

---

## Moisture Parameters

### Precipitable Water (PWAT)
- **Formula:** `PWAT = (1/ρg) * ∫ w dp` (surface to 400 hPa)
- **Unit:** inches — total column water vapor if condensed

### θe Difference
- **Layer:** Surface to 3 km
- **Formula:** `Δθe = θe(surface) - θe(3km)`
- Large positive values indicate potential instability

---

## Winter Weather Parameters

### Precipitation Type Algorithm
1. Identify warm (T >0°C) and cold (T ≤0°C) layers
2. Calculate warm and cold area (temperature-depth integrals)
3. Evaluate layer depths and temperature maxima
4. Apply decision logic:

| Surface | Condition | Type |
|---------|-----------|------|
| Above freezing | Max warm >4°C | Rain |
| Above freezing | Max warm <1.5°C | Snow |
| Below freezing | No warm layer | Snow |
| Below freezing | Cold depth <300 m or warm/cold ratio >3 | Freezing Rain |
| Below freezing | Cold depth >600 m and max warm >3°C | Ice Pellets |
| Below freezing | Cold depth >400 m | Ice Pellets |
| Below freezing | Default | Freezing Rain |

### Dendritic Growth Zone (DGZ)
Layer from −12°C to −18°C where dendritic crystal growth is maximized.

| Parameter | Formula/Source | Unit |
|-----------|---------------|------|
| Depth | \|H(−12°C) − H(−18°C)\| | m |
| Omega (ω) | Mean vertical velocity in DGZ | Pa/s |
| PWAT | Mean precipitable water in DGZ | inches |
| RH | Mean relative humidity in DGZ | % |
| OPRH | ω × PWAT × RH (negative = upward motion, favors snow) | — |

### Snow Squall Parameter (SNSP)
```
SNSP = RH_term × θe_term × Wind_term
```
- `RH_term`: 0 if RH <60%; else `(RH−60)/15`
- `θe_term`: `(4 − Δθe) / 4`
- `Wind_term`: `MeanWind / 9 m/s`

---

## Fire Weather Indices

### Fosberg Fire Weather Index (FFWI)
```
FFWI = η × √(1 + U²) / 0.3002
```
`η` is a function of equilibrium moisture content derived from RH and temperature. Range: 0–100.

| Range | Danger |
|-------|--------|
| <25 | Low |
| 25–50 | Moderate |
| 50–75 | High |
| >75 | Extreme |

### Hot-Dry-Windy Index (HDWI)
```
HDWI = max(VPD_0-500m) × max(Wind_0-500m)
```
VPD = `es(T) − e(Td)` in lowest 500 m.

### Red Flag Threat Index (RFTI)
```
RFTI = Wind(mph) / RH(%)
```

---

## Shear Layers Reference

### Height-Based Layers
`0–500 m`, `0–1 km`, `0–3 km`, `0–6 km`, `1–3 km`, `3–6 km`, `0–9 km`, `0–12 km`, `9–12 km`

### Pressure-Based Layers
`Sfc–300 mb`, `Sfc–500 mb`, `Sfc–700 mb`

### Temperature-Based Layers
`DGZ` (−12°C to −18°C), `Sfc to −18°C`, `HGZ` (−10°C to −30°C)

Parameters computed per layer: SRH, BWD, EHI, Mean Wind

---

## Interpolation Methods

| Method | Formula |
|--------|---------|
| Pressure level | `f(p) = f(p1) + (f(p2)−f(p1)) × (p−p1) / (p2−p1)` |
| Height level | `f(z) = f(z1) + (f(z2)−f(z1)) × (z−z1) / (z2−z1)` |
| Temperature level | Linear interpolation between bounding levels |
| Wind | Interpolate u/v components separately; convert back to speed/direction |

---

## Unit Conversions

| Conversion | Factor |
|-----------|--------|
| kt → m/s | × 0.514444 |
| m/s → kt | × 1.94384 |
| mph → m/s | × 0.44704 |
| m/s → mph | × 2.23694 |
| ft → m | × 0.3048 |
| m → ft | × 3.28084 |
| inHg → hPa | × 33.8639 |
| °C → °F | `F = C × 9/5 + 32` |
| °C → K | `K = C + 273.15` |

---

*Schema version 2.1 — covers Thermodynamic, Kinematic, Composite, Winter Weather, and Fire Weather categories.*
