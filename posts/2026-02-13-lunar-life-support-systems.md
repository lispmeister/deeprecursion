---
title: "Lunar Habitat Life Support Systems"
date: 2026-02-13
meta: "Atmospheric Control & CO₂ Scrubbing"
url: https://lispmeister.github.io/deeprecursion/posts/2026-02-13-lunar-life-support-systems.html
---

# Lunar Habitat Life Support Systems

*Atmospheric Control & CO₂ Scrubbing · February 2026*

> **Executive Summary:** The race to establish permanent human presence on the Moon is driving a generational shift in Environmental Control and Life Support Systems (ECLSS). While the ISS has served as the proving ground for current-generation CO₂ removal technology — primarily zeolite-based adsorption systems like the CDRA — the demands of lunar surface habitats are fundamentally different: higher crew counts, longer durations, no easy resupply, the need for gravity-compatible systems, and extreme thermal environments.

## Current State: ISS Heritage Systems

### Carbon Dioxide Removal Assembly (CDRA)

The CDRA, built by Honeywell Aerospace, has been the primary CO₂ removal system for the U.S. segment of the ISS since February 2001. It operates as a dual-bed, zeolite-based temperature-swing adsorption (TSA) system. Two sets of desiccant/adsorbent bed pairs alternate between adsorbing CO₂ from cabin air and desorbing it into space vacuum. The system supports a crew of up to six at approximately 0.4% CO₂ concentration.

The CDRA has suffered from persistent reliability issues — most notably zeolite pellet dust contaminating downstream components, degraded seals, and filter clogging — making it a high-maintenance system requiring regular crew intervention. ACE Clearwater Enterprises has manufactured replacement inner chambers, end caps, and specialized filters for the CDRA in partnership with Honeywell and NASA.

### The 4-Bed CO₂ Scrubber (4BCO₂)

The 4BCO₂ is NASA Marshall Space Flight Center's evolution of the CDRA, developed with collaboration from NASA Johnson. It incorporates lessons from CDRA's failure modes: cylindrical beds replace rectangular ones, redesigned heater cores eliminate void spaces, new dust filters capture sorbent particles, and improved valves extend operating lifetimes. The sorbent selection process evaluated BASF 13X and Grace MS544 C 13X zeolites, both proven robust under humid and dry conditions. The goal is continuous, failure-free operation for nearly 20,000 hours.

---

## Top Companies Working on Next-Generation Systems

### Honeywell Aerospace

*Industry leader and legacy provider of ISS CDRA systems; developer of next-generation CDRILS technology*

Honeywell is developing the **Carbon Dioxide Removal by Ionic Liquid Sorbent (CDRILS)** system, which represents a paradigm shift from solid sorbent to liquid absorbent CO₂ removal. CDRILS uses an ionic liquid mixture containing 1-ethyl-3-methylimidazolium acetate (EMIM[Ac]) with hollow fiber membrane contactors for continuous CO₂ absorption and desorption. A flight demonstration on the ISS is targeted for 2028.

**Key advantages over CDRA:**

- Continuous flow processing (no batch cycling between beds)
- Can maintain CO₂ partial pressure below 2 mmHg — half that of CDRA
- Eliminates the need for a CO₂ compressor and accumulator (saving 291.1 kg of equivalent system mass)
- Can also handle humidity control and significant trace contaminant removal
- Lower volume, mass, power, and cooling requirements

### Paragon Space Development Corporation

*ECLSS designer for NASA Lunar Gateway HALO module; specialist in life support, thermal control, and ISRU*

Paragon is responsible for the design and implementation of the HALO (Habitation and Logistics Outpost) ECLSS for the Lunar Gateway, under a ~$100M subcontract from Northrop Grumman (awarded May 2021). The HALO ECLSS maintains safe levels of oxygen, CO₂, humidity, and trace contaminants for visiting crew during stays of up to 90 days.

**Key technologies:**

- **Ionomer-membrane Water Processing (IWP)** — used on Boeing's CST-100 Starliner for humidity control with no moving parts
- **Brine Processor Assembly (BPA)** — launched to ISS in February 2021, increasing water loop closure from 93% to over 98%
- **Primary Urine Processor (PUP)** — designed to replace two ISS systems with one compact unit
- **Atmospheric Monitoring System (AMS)** — detects harmful gases with integrated microgravity smoke detection

### Collins Aerospace (RTX Corporation)

*ECLSS provider for commercial orbital habitats; legacy Apollo and ISS spacesuit life support*

Collins Aerospace was awarded a $2.6M contract in 2021 for ECLSS supporting a privately owned orbital outpost in LEO. Their offering includes air revitalization and pressure control (cabin fans, heat exchangers, CO₂ removal, trace contaminant control, valves, regulators, smoke detection) and active thermal control. Collins also provides the Sabatier CO₂ reduction system currently operational on the ISS.

### Precision Combustion, Inc. (PCI)

*Sabatier reactor specialist; catalytic systems for CO₂ conversion*

PCI has developed a compact 4-crew Sabatier reactor that integrates directly with Honeywell's CDRILS system, converting captured CO₂ and hydrogen into water and methane. The reactor's waste heat is repurposed to heat the ionic liquid in the CDRILS system, saving approximately 72.5–145 W. PCI is a key member of the Honeywell-led Commercial ECLSS team.

### Thales Alenia Space

*Prime contractor for ESA's Lunar I-Hab module and Italy's Lunar Multi-Purpose Habitat*

Thales Alenia Space in Turin is building the primary structure for the Lunar I-Hab module (launching on Artemis IV circa 2028), with JAXA providing the ECLSS, batteries, and thermal control. The I-Hab provides approximately 10 m³ of habitable volume for up to four astronauts for 30–90 day stays.

### Airbus Defence and Space

*ECLSS design for Starlab commercial space station; ISS Columbus module heritage*

Airbus is providing technical design and engineering for the Starlab station's ECLSS, which leverages over 30 years of NASA and ESA investment in life support and thermal control. The station targets a loop closure greater than 90%, with a continuous crew of four (up to eight temporarily) in a 450 m³ pressurized volume. Launch planned on SpaceX's Starship in 2028–2029.

### Vast (Haven Stations)

*In-house ECLSS development for commercial space stations with iterative approach*

Vast is developing its own life support systems in-house at its Long Beach headquarters. Haven-1 (expected Q1 2027 launch) uses a simple open-loop ECLSS architecture suited to short-duration crewed missions. However, Vast is flying closed-loop ECLSS experiments on every Haven-1 mission to enable rapid iteration. The company expects to be on a 5th-generation ECLSS system by the time Haven-2 launches to support continuous human presence.

---

## CO₂ Capture Technologies: Comparative Analysis

| Technology | TRL | Crew Scale | Power (4-crew) | Consumables | Lunar Suitability |
|---|---|---|---|---|---|
| **CDRA (Zeolite TSA)** | 9 (operational) | 4–6 | ~500W | Zeolite replacement | Moderate — proven but high maintenance |
| **4BCO₂ (Improved Zeolite)** | 7–8 | 4–6 | ~400W | Improved zeolites | Moderate — addresses CDRA failure modes |
| **CDRILS (Ionic Liquid)** | 5–6 | 4 (prototype) | ~300W est. | IL replenishment (~2yr) | High — continuous flow, low maintenance |
| **CDep (Cryogenic)** | 4–5 | 4 (full-scale ground) | ~130W (deposition) | None | Very High — leverages lunar cold |
| **Vozdukh (Amine TSA)** | 9 (operational) | 6 | ~400W | Amine replacement | Moderate — degradation concerns |
| **BLSS (Bioregenerative)** | 3–4 | 4 (ground demo) | High (lighting) | Biological inputs | Future — 2040s for operational systems |

---

## China's Yuegong-1: The Large-Scale Demonstrator

The most advanced large-scale life support demonstration belongs to China. The **Yuegong-1 (Lunar Palace 1)** at Beihang University in Beijing is a 500 m³, 160 m² bioregenerative life support system (BLSS) that successfully sustained a crew of four for 370 days in 2017–2018 — the world record for the longest stay in a self-contained laboratory.

Yuegong-1 demonstrated 100% atmospheric regeneration through plants and microalgae, with 55% food self-sufficiency. The facility includes two plant cultivation cabins (58 m²), a living area (42 m²) with bedrooms, dining room, bathroom, and waste disposal. A critical August 2025 paper in *npj Microgravity* warns that NASA has fallen behind China in bioregenerative life support capabilities.

---

## Key Observations

> **No one is building a lunar-base-scale ECLSS today.** All current programs — Gateway HALO, Lunar I-Hab, Artemis Surface Habitat — are designed for 4 crew for 30–90 day stays. True settlement-scale life support (8–12+ crew, continuous habitation for months or years) remains a paper exercise outside of China's Yuegong program.

> **CDep is the most promising breakthrough for lunar surface application.** The ability to leverage the Moon's cold environment (permanently shadowed craters near the south pole reach approximately 40K) for radiative cooling could dramatically reduce the power requirements of cryogenic CO₂ capture, potentially making CDep more efficient on the lunar surface than in LEO.

> **The CDRILS ISS flight demo in 2028 is a pivotal milestone.** If it succeeds, it could become the baseline air revitalization technology for all post-ISS crewed habitats, displacing zeolite-based systems.

> **The commercial space station race is driving ECLSS innovation faster than government programs.** Vast, Starlab, Orbital Reef, and Axiom are all investing in ECLSS as a competitive differentiator, with NASA's 2030 ISS retirement deadline creating urgency. These LEO systems will inform the technology baseline for lunar surface habitats.

*Sources: NASA Technical Reports Server, International Conference on Environmental Systems (ICES) proceedings 2018–2025, Honeywell Aerospace, Paragon Space Development Corporation, ESA, npj Microgravity (August 2025), Tech Briefs, SpaceNews, Aerospace America (July 2025).*
