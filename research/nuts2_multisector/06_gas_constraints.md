# Natural Gas documentation 4/5: constraint catalogue

Part 4 of the series. Commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8`. Paths
and line numbers are relative to `src/`. Nothing was run.

Labels: **[V]** verified, **[I]** interpretation, **[U]** unverified.

Notation follows `05_gas_decision_variables.md` and
`04_gas_parameters_and_lineage.md`. $p(t)$ is the strategic-period index of
operational period $t$ (`period_context[t].strategic`), $\mu$ = `mwhPerTon`,
$\rho$ = `NATURAL_GAS_ROW_SCALE` = 1e-3.

---

## 1. Where constraints are created

`create_natural_gas_constraints!` (`natural_gas.jl:252-404`) is called from
`create_constraints` (`model_definition.jl:292`) after the power constraints and
the emission constraints, and before Hydrogen and Industry. Before it runs, the
power `flow_balance` has already been built with the pipeline electricity
expression (`model_definition.jl:252-255`). Reserve rows are built by
`_create_natural_gas_reserve_constraints!` (`:183-242`). [V]

All constraints are linear. All are built for every gas node, arc, terminal or
generator, and every operational period, unless stated otherwise. [V]

## 2. Constraint catalogue

### C1. `natural_gas_for_power[(n,g), t]` (`:271-280`)

$$\mu\; q_{n,g,t} \;=\; \frac{P_{n,g,t}}{\eta^{gen}_{g,p(t)}}$$

- $P$ = `genOperational[n,g,t]` (MW, power module); $q$ = `ngForPower`.
- Units: MWh/h on both sides ($\mu$ in MWh/ton times ton/h).
- Index set: `_natural_gas_node_generators(sets)`, i.e. (node, generator) pairs
  whose generator contains "gas".
- Active whenever the gas module is on. [V]
- Interpretation: gas consumption is exactly the fuel-equivalent of the
  electricity produced at that node. There is no investment variable in the gas
  module; the capacity that sets $P$ is a power-module decision. [I]

### C2. `natural_gas_terminal_capacity_limit[(n,k), t]` (`:281-294`)

$$x_{n,k,t} \;\le\; C^{term}_{n,k,p(t)}$$

- Missing keys would give 0, but validation makes all keys mandatory. [V]
- 72 of the 308 capacity rows are zero in the dataset, and the Russian-gas
  capacity at several nodes falls to zero by period 7. [V]

### C3. `natural_gas_max_reserves[n,k,w,gs]` (`:183-242`)

For each (node, terminal) with a finite-reserve terminal type
(`domesticproduction` or `pipelineimport`, case-insensitive, `:5, 18`), and for
each weather scenario $w$ and gas scenario $gs$:

$$\rho\sum_{sp}\; \sum_{rp}\;\sum_{t \in \text{scenario}(w,gs)} \text{dur}(sp)\,\text{mult}(sp,t)\; x_{n,k,t} \;\le\; \rho\,R_n$$

- The scenario of each representative period is selected by
  `_operational_scenario_at(rp, (w-1)·G + gs)` (`:199-205`). [V]
- It sums over the entire horizon, so it is a **cumulative** bound, with no
  probability weight. `dur` and `mult` are `duration_strat` and
  `multiple_strat`. [V]
- The rows are stored in `emp[:natural_gas_max_reserves]`, a plain vector, with
  names like `natural_gas_max_reserves[Italy,PipelineImport,1,1]`. [V]
- Because the row is per (node, terminal), a node that has two finite-reserve
  terminals gets the same $R_n$ applied to each. In the dataset this is Italy
  (DomesticProduction and PipelineImport). [V]
- Count in the dataset: 16 finite-reserve (node, terminal) pairs x W x G rows. [V]
- Assumes that each representative period has at least $W\cdot G$ operational
  scenarios. Not validated, and the single-scenario structure
  (`SimpleTimes` without `OperationalScenarios`, `model_definition.jl:47`) was
  not traced. [U]

### C4. `natural_gas_storage_balance[n, sp, (prev, t)]` (`:297-314`)

$$\rho\Big(\;\underbrace{\begin{cases}\phi_0 S_n & \text{if prev is nothing}\\ s_{n,prev}&\text{otherwise}\end{cases}}_{\text{inventory before step}} +\eta^{ch}u_{n,t}-d_{n,t}\Big) \;=\; \rho\, s_{n,t}$$

- Iterates `withprev(strategic_period)`. `prev` is `nothing` at the first hour
  of **each (representative period, scenario) block**, because TimeStruct sets
  `prev = nothing` whenever `isfirst(t)` (`TimeStruct/utils.jl:22-26`;
  `isfirst` of a `SimplePeriod` is `op == 1`). [V]
- Consequence: inventory restarts at $\phi_0 S_n = 50\%$ of capacity in every
  block. It cannot be carried between seasons, peak days, scenarios or years. [V]
- With $S_n = 0$ (20 of 37 gas nodes) the row forces $s = 0$ and
  $u\cdot\eta^{ch} = d$. [V]

### C5. `natural_gas_storage_cyclic[n, sp, scenario]` (`:315-326`)

$$\rho\, s_{n,\text{last}(scenario)} \;=\; \rho\,\phi_0 S_n$$

- `opscenarios(strategic_period)` for a strategic period with representative
  periods returns the flattened list of (representative period, scenario)
  blocks (`TimeStruct/op_scenarios/strat_periods.jl:226-230`), so there is
  one row per block. [V]
- The end-of-block inventory must equal the starting 50 %. [V]

### C6. `natural_gas_storage_max_capacity[n, t]` (`:327-335`)

$$\rho\, s_{n,t} \;\le\; \rho\, S_n$$

- No separate bound on charge or discharge rate and no storage cost. [V]

### C7. `natural_gas_pipeline_capacity_limit[(i,j), t]` (`:336-350`)

$$f_{ij,t} \;+\; \mathbb{1}[\text{Hydrogen on},\,(i,j)\in\text{RepurposableGasCorridor}]\;\cdot Y_{ij,sp(t)} \;\le\; C^{pipe}_{ij}$$

- $Y$ = `hydrogenRepurposedGasPipelineCapInstalled[i,j,sp]`, a Hydrogen
  investment variable that removes gas capacity (same units as the gas flow in
  this constraint; the conversion to hydrogen capacity uses
  $\mu/\text{hydrogenMWhPerTon}$ in `hydrogen.jl:620-626`). [V]
- Arcs are directed; each direction has its own capacity. [V]
- Gas-only arcs that touch Sleipner or Draupner are not electricity arcs, so
  they do not appear among derived hydrogen corridors. Whether they can be
  repurposed was only read from `read_csv.jl:256-262`. [I/U]

### C8. `meet_transport_natural_gas_demand[n, t]` (`:351-365`)

$$m_{n,t} + h_{n,t} \;\ge\; \frac{D_{n,p(t)}}{8760\,\mu}$$

- Index: onshore gas nodes. **Created only when `transport_demand` is true**,
  which the YAML path sets to `hydrogen_enabled` (`user_interface.jl:292`). In a
  gas-only run the constraint does not exist. [V]
- 8760 is hard-coded; the time resolution of the structure is not used. [V]
- Inequality (`>=`): `Met + Shed` may exceed the demand; the surplus is gas that
  appears in the balance (C9) but has no use, which only happens if it is free.
  Since `Met` has no cost, the optimiser uses just the necessary amount. [I]

### C9. `natural_gas_flow_balance[n, t]` (`:366-403`)

$$\sum_{k\in\text{Term}(n)} x_{n,k,t} + \sum_{i\in\text{in}(n)} f_{in,t} + \eta^{dis}d_{n,t}
=\sum_{g\in\mathcal G_n} q_{n,g,t} + \sum_{r} \text{ref}_{n,r,t} + \sum_{j\in\text{out}(n)} f_{nj,t} + u_{n,t} + m_{n,t} + I_{n,t}$$

- Equality, one row per gas node and operational period. [V]
- $\text{ref}$ = `reformerNaturalGas` and is included only if that variable
  exists and the node is a reformer location. [V]
- $m$ appears only if `transport_demand` and the node is onshore. [V]
- $I$ = `industry_natural_gas_demand`, zero unless the Industry module has
  created `cementProduced` (`industry.jl:319-344`). [V]
- No loss term, no venting. Surplus gas must be stored or not imported. [V]
- Dual of this row divided by the objective weight is the nodal gas price
  written in `naturalGasOperationalDuals.csv` (doc 07). [V]

## 3. Constraint summary

| ID | Name | Rows (dataset-level count) | Type | Activation |
|---|---|---|---|---|
| C1 | `natural_gas_for_power` | 148 x T | equality | gas |
| C2 | `natural_gas_terminal_capacity_limit` | 44 x T | inequality | gas |
| C3 | `natural_gas_max_reserves` | 16 x W x G | inequality | gas |
| C4 | `natural_gas_storage_balance` | 37 x T | equality | gas |
| C5 | `natural_gas_storage_cyclic` | 37 x number of blocks | equality | gas |
| C6 | `natural_gas_storage_max_capacity` | 37 x T | inequality | gas |
| C7 | `natural_gas_pipeline_capacity_limit` | 94 x T | inequality | gas |
| C8 | `meet_transport_natural_gas_demand` | 35 x T | inequality | Hydrogen flag |
| C9 | `natural_gas_flow_balance` | 37 x T | equality | gas |

T = number of operational periods; counts use the sets of `full_model_int` and
come from set sizes, not from a model build. [I]

## 4. Constraints outside the gas module that involve gas variables

| Constraint | Module | Gas content | Reference |
|---|---|---|---|
| `flow_balance[n,t]` | Power | subtracts $\pi\sum_j f_{nj,t}$ at the sending node | `model_definition.jl:245-259` |
| `hydrogen_reformer_natural_gas` | Hydrogen | $\mu\cdot\text{ref} = \text{MWh H}_2/\eta^{ref}$ | `hydrogen.jl:329-337` |
| `hydrogen_repurpose_capacity` | Hydrogen | repurposed build $\le$ gas capacity | `hydrogen.jl:632-639` |
| `hydrogen_pipeline_installed` | Hydrogen | repurposed capacity counts as hydrogen capacity | `hydrogen.jl:612-627` |
| `co2_flow_balance` | Hydrogen/CO2 | CO2 captured by gas CCS generators, via `genOperational`, not via `ngForPower` | `hydrogen.jl:521-548` |
| `node_emission` | Power | CO2 from gas generators, via `genOperational`, not via `ngForPower` | `model_definition.jl:769-784` |

Observation: emissions of gas generators are computed from generation and the
emission factor, so the gas flow variables and the emissions are tied only
through $P$ in C1. [V]

## 5. Solver conditioning

- Reserve and storage rows are scaled by 1e-3 on both sides because reserves
  reach about 5.6e8 ton against coefficients near 7.3e3. The scaling does not
  change the feasible set (`natural_gas.jl:7-11`). [V]
- Shadow prices of C4 must be multiplied by $\rho$ again; the result writer does
  this (`results.jl:1450-1455`). [V]

## 6. NUTS2 observations (no design)

- C1 couples gas and power only through the shared node name; the generator must
  sit at the gas node itself (`empire_sets.jl:646-655`). [V]
- C7 is a per-arc bound with a per-arc capacity and no lengths. [V]
- C8 needs one total demand per onshore node and period. [V]
- C3 is a per-node cumulative total; a finer node split would need a rule to
  split $R_n$, and this version has no such rule. [I]

## 7. Open questions

1. Whether applying the node reserve once per finite-reserve terminal (C3) is
   intended. [U]
2. Single-scenario (`nscens == 1`) behaviour of `opscenarios` in C3 and C5. [U]
3. The 8760 constant in C8. [U]
4. Whether surplus in C8 (`Met` above demand) can occur. It would require a
   non-positive gas price at that node. [I]
