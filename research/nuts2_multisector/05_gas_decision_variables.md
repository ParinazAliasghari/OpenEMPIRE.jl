# Natural Gas documentation 3/5: decision variables

Part 3 of the series. Commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8`. Paths
are relative to `src/`. Nothing was run.

Labels: **[V]** verified, **[I]** interpretation, **[U]** unverified.

---

## 1. Declaration mechanism

Variables are created in `create_natural_gas_variables!`
(`natural_gas.jl:91-163`), called from `create_variables`
(`model_definition.jl:178`) when `natural_gas` is true, after the power
variables and before Hydrogen and Industry. [V]

- All eight variables are declared with `@variable(emp, name[...] >= 0;
  container = IndexedVarArray)`. They are **continuous** with an explicit lower
  bound of 0 and **no explicit upper bound**. There are no binary or integer
  variables and no fixed variables in the gas module. [V]
- `IndexedVarArray` (SparseVariables) is a sparse container. Declaring an index
  product creates nothing by itself; variables exist only after
  `unsafe_insertvar!(var, indices...)` is called for a tuple (`:140-161`). [V]
- Solver defaults: no start values, no warm start and no branching priority are
  set for gas variables in this file. Upper limits come entirely from
  constraints (doc 06). [V]
- Periods `t` are operational periods of the TimeStruct structure (1-hour steps
  inside representative periods and operational scenarios). [V]

## 2. Variable dictionary

| Julia name | Math | Declared over | Actually inserted for | Unit | Bounds | Type |
|---|---|---|---|---|---|---|
| `ngTerminalImport[n,k,t]` | $x_{n,k,t}$ | gas nodes x terminal types x periods | the 44 `TerminalsOfNode` pairs x all periods | ton/h | $\ge 0$ | continuous |
| `ngTransmission[i,j,t]` | $f_{ij,t}$ | gas nodes x gas nodes x periods | the 94 directed arcs x all periods | ton/h | $\ge 0$ | continuous |
| `ngForPower[n,g,t]` | $q_{n,g,t}$ | gas nodes x gas generators x periods | (node, gas generator) pairs at gas nodes (148 in the dataset) | ton/h | $\ge 0$ | continuous |
| `ngStorageOperational[n,t]` | $s_{n,t}$ | gas nodes x periods | **all 37 gas nodes** | ton | $\ge 0$ | continuous |
| `ngStorageCharge[n,t]` | $u_{n,t}$ | gas nodes x periods | all 37 gas nodes | ton per step | $\ge 0$ | continuous |
| `ngStorageDischarge[n,t]` | $d_{n,t}$ | gas nodes x periods | all 37 gas nodes | ton per step | $\ge 0$ | continuous |
| `transportNaturalGasDemandMet[n,t]` | $m_{n,t}$ | gas nodes x periods | onshore gas nodes, **only if `transport_demand` is true** | ton/h | $\ge 0$ | continuous |
| `transportNaturalGasDemandShed[n,t]` | $h_{n,t}$ | gas nodes x periods | same as above | ton/h | $\ge 0$ | continuous |

Insertion loops: terminals `:140-142`, arcs `:143-145`, generators `:146-148`,
storage `:149-153`, transport `:156-161`. [V]

Units note: charge and discharge are flows within one 1-hour step, so numerically
ton per step equals ton/h; `s` is a stock in ton. The storage balance adds
`charge` and subtracts `discharge` to the stock without an explicit duration
factor, which is consistent only for 1-hour steps (the model builds
`SimpleTimes(hours, 1)`; `model_definition.jl:44-48`). [V from code, I for the
unit reading]

## 3. Activation conditions

| Variable family | Created when |
|---|---|
| terminal, transmission, for-power, storage | `natural_gas: true` |
| transport met / shed | `natural_gas: true` **and** `transport_demand = true` |

Where `transport_demand` comes from:

- `create_model` passes `gas_transport_demand = hydrogen_enabled`
  (`user_interface.jl:292`). Gas-only YAML runs therefore get empty transport
  variables. The code comment says InternalEMPIRE declares them inside its
  Hydrogen block and the sparse empty container avoids "millions of free
  columns" (`natural_gas.jl:154-155`). [V]
- The lower-level `create_variables` and `create_constraints` default to
  `gas_transport_demand = natural_gas` (`model_definition.jl:86, 220`). A
  caller that skips `create_model` and enables gas alone therefore gets
  transport demand. The two entry points are inconsistent. [V]

## 4. Variables from other modules that appear in gas constraints

| Variable | Module | Role in gas constraints |
|---|---|---|
| `genOperational[n,g,t]` | Power | right-hand side of `natural_gas_for_power` |
| `reformerNaturalGas[n,plant,t]` | Hydrogen | consumption term in `natural_gas_flow_balance` (`:386-392`) |
| `hydrogenRepurposedGasPipelineCapInstalled[i,j,sp]` | Hydrogen | reduces pipeline capacity (`:342-349`) |
| industry cement/ammonia production | Industry | gas demand via `industry_natural_gas_demand` (`industry.jl:319-344`) |

These are looked up through `haskey(JuMP.object_dictionary(emp), ...)`, so the
gas constraints silently omit the term if the variable does not exist. [V]

Gas variables used by other modules:

- `ngTransmission` enters the **power** flow balance through
  `natural_gas_pipeline_electricity_demand` (`natural_gas.jl:165-181`,
  `model_definition.jl:252-255`). [V]

## 5. Scale and size notes

- Sparse indexing keeps the count at: 44 x T terminal, 94 x T arc, 148 x T
  generator, 3 x 37 x T storage, and up to 2 x 35 x T transport, where T is the
  number of operational periods (a comment cites 19,440 for one configuration,
  `natural_gas.jl:57-58`; the figure depends on the config). [V]
- Storage variables exist even where storage capacity is 0 (20 of 37 gas nodes
  in the dataset). They are then forced to 0 by `natural_gas_storage_max_capacity`
  and the cyclic row, adding columns without freedom. [V from code and data]

## 6. NUTS2 observations (no design)

- All gas variables are indexed by node name strings of the gas node set;
  nothing is indexed by a region attribute. [V]
- The generator variable `ngForPower` is indexed by the same (node, generator)
  pairs as the power module, so it follows the power node of the generator. [V]
- Pipelines are indexed by gas arcs, not by electricity arcs. [V]

## 7. Open questions

1. Whether charge and discharge have an implicit duration factor intended by
   the reference model (they are used as per-step quantities). [U]
2. The two entry points' different `transport_demand` defaults (section 3). [U]
