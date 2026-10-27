# Natural Gas system overview (Phase 2, Part A)

Based on commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8` (this branch adds
only research notes). Line numbers are relative to `src/` unless a path is
given. Data examples come from `data/full_model_int`. Nothing was run.

Labels: **[V]** verified in source or data, **[C]** conceptual explanation,
**[U]** unverified / open question.

Scope: system overview only. Equations and CSV-by-CSV detail are for later
parts. `NaturalGas/TerminalCost_stochastic.csv` and other secondary files are
not covered.

## 1. System overview

The gas module is a **steady, flow-based network model** added to the same
JuMP model as Power (`natural_gas.jl`). It has no investment decisions and no
gas losses. [V]

```
 terminals (supply)  --->  node balance  <--->  directed pipelines <---> other nodes
 DomesticProduction          |   |  \
 LNGImport                   |   |   +-- storage (charge / discharge / inventory)
 PipelineImport              |   |
 RussianGas                  v   v
                  consumption: gas power plants, [hydrogen reformers],
                               [transport demand], [industry]
```

| Element | Implemented? | Variable | Reference |
|---|---|---|---|
| Supply by terminal | yes | `ngTerminalImport[node, terminal, t]` (ton/h) | `natural_gas.jl:105-150` |
| Directed pipeline flow | yes | `ngTransmission[from, to, t]` (ton/h) | `:110, :144` |
| Gas for power plants | yes | `ngForPower[node, generator, t]` | `:271-280` |
| Storage | **yes** (see 3.4) | `ngStorageOperational / Charge / Discharge[node, t]` | `:120-126, :297-335` |
| Reformer gas | only with Hydrogen | `reformerNaturalGas` (hydrogen module) | `:386-392` |
| Transport demand | only with Hydrogen (see 4) | `transportNaturalGasDemandMet / Shed` | `:351-365` |
| Industry demand | only with Industry | `industry_natural_gas_demand(...)` | `:402`; `industry.jl:319-344` |
| Gas investment (new pipelines, storage, terminals) | **no**: capacities are fixed data | none | [V] |
| Pipeline losses / gas cost of transport | **no** | none | [V] |
| Gas flow physics (pressure, linepack) | **no** | none | [V] |

Activation: YAML `natural_gas: true` (default false). Hydrogen and Industry
require it, and the three modules require the CSV dataset layout.
Gas runs also require `number_of_gas_scenarios = 1` (`empire_structs.jl:611`;
`user_interface.jl:129-140`). [V]

Time: variables are indexed by operational period of the TimeStruct structure
(strategic periods, representative periods, scenarios, 1-hour steps), so gas
flows are hourly. Supply capacity and prices depend on the strategic period
only. [V]

## 2. Gas supply

Four terminal types (`Sets/NaturalGasTerminals.csv`). A terminal is a (node,
type) pair listed in `Sets/NaturalGasTerminalsOfNode.csv`; there are 44 pairs
at 28 of the 37 gas nodes. [V]

| Type | Pairs | Finite reserve? |
|---|---|---|
| DomesticProduction | 14 | yes |
| LNGImport | 16 | no |
| PipelineImport | 2 (Spain, Italy) | yes |
| RussianGas | 12 | no |

The type name has no behaviour of its own: the code distinguishes only
"finite reserve" (case-insensitive `domesticproduction` or `pipelineimport`,
`natural_gas.jl:5, 18`) from the rest. LNGImport and RussianGas differ from
domestic production only through their data (capacity, price). [V]

- **Capacity** (`TerminalCapacity.csv`, ton/h per node, terminal, strategic
  period): upper bound `terminal_import <= capacity` in
  `natural_gas_terminal_capacity_limit` (`:281-296`). Example: Germany
  RussianGas 4269.41 t/h in period 1 and 0 in period 7, so supply routes close
  over time. [V]
- **Cost** (`TerminalCost.csv`, EUR/ton per node, terminal, period, gas
  scenario): enters the objective as
  `weight * cost * import` (`:420-436`). Domestic and pipeline gas are mostly
  278 EUR/t, which equals 20 EUR/MWh at 13.9 MWh/t. German LNG rises from
  793 to 939 EUR/t. [V]
- **Reserves** (`Reserves.csv`, ton per node, 15 nodes): total imports of a
  finite-reserve terminal over the whole horizon are limited
  (`natural_gas_max_reserves`, `:183-242`). [V] Example (arithmetic on data):
  Germany DomesticProduction at full capacity yields 481.63 t/h x 8760 = 4.2
  Mt/yr; its reserve is 13.4 Mt, so the reserve binds after about 3.2 years of
  full output. Whether this binds in a given run depends on the horizon and
  weights and was not run. [C]
- **Surprise:** the reserve is per (node, terminal), so Italy, which has both
  DomesticProduction and PipelineImport but one reserve value, can use that
  value twice. [V from code `:192-236`; intent **[U]**]

## 3. Gas network

### 3.1 Nodes
37 gas nodes (35 onshore + Sleipner + Draupner), all inside the power `Node`
set (checked in `empire_sets.jl:628`). Nine have no terminal. Gas nodes carry
no attributes of their own; everything is keyed by node name. [V]

### 3.2 Pipelines
94 directed arcs (`NaturalGasDirectionalLines.csv`), each with its own
capacity in ton/h (`PipelineCapacity.csv`); 80 have a reverse arc, 14 are
one-way, 14 have zero capacity. Capacity is asymmetric
(France to Switzerland 704.9, Switzerland to France 248.5 t/h). [V]
The pipeline list is independent of the electricity grid, although 86 of the
94 arcs are also electricity arcs. [V]

- Constraint: `natural_gas_pipeline_capacity_limit` (`:336-350`):
  `flow + hydrogen-repurposed capacity <= capacity`. The repurposing term
  exists only when Hydrogen is on. [V]
- **No pipeline length** appears in the gas module. [V]
- Pipeline operation uses electricity: 0.02445 MWh per ton, one global value
  (`PipelineElectricityUse.csv`), charged at the sending node in the power
  balance (`natural_gas.jl:165-181`; `model_definition.jl:252-255`). [V]

### 3.3 Node balance
`natural_gas_flow_balance` (`:366-403`), an **equality** per gas node and
operational period:

```
terminal imports + inflow + storage discharge
   = gas for power + reformer gas + outflow + storage charge
     + transport demand met + industry gas
```

Gas cannot be vented or lost; surplus has to be stored or the supply reduced.
[V] Gas-power conversion: `13.9 * ngForPower = genOperational / efficiency`
(`:271-280`; 13.9 MWh/ton is hard-coded in `empire_structs.jl`). [V]

### 3.4 Storage (implemented)
Storage **is** implemented, with three variables per gas node and operational
period and three constraints [V]:

- `natural_gas_storage_balance` (`:297-314`): inventory(t) = inventory(t-1) +
  charge efficiency x charge - discharge. At the first step of a block the
  previous inventory is 50 % of capacity (`storageInitialFraction`).
- `natural_gas_storage_cyclic` (`:315-326`): the inventory at the end of each
  scenario block must return to 50 % of capacity.
- `natural_gas_storage_max_capacity` (`:327-335`): inventory <= capacity.

Capacity comes from `StorageCapacity.csv` (ton; 17 nodes), and nodes not listed
have capacity 0, so they cannot store. [V] Charge and discharge efficiencies are
both 1.0 by default, and charge/discharge have no explicit rate limit and no
cost. [V from `empire_structs.jl` defaults and absence of such constraints]

Interpretation **[C/U]**: the blocks restart at 50 % for every representative
period and scenario, so storage behaves as short-term (within-season) balancing
and cannot move gas between seasons or years. This follows from the
`withprev`/`opscenarios` structure and was not traced further.

## 4. Gas demand

| Consumer | Active when | Data / link | Status |
|---|---|---|---|
| Gas-fired power | `natural_gas: true` | `genOperational`, `genEfficiency` | **endogenous**, always present [V] |
| Transport gas demand | **`hydrogen: true`** | `Transport/NaturalGasDemand.csv` (MWh/yr per onshore node and period), curtail cost 100000 | exogenous; implemented but gated [V] |
| Hydrogen reformers | `hydrogen: true` | `reformerNaturalGas`, `ReformerLocation` nodes | endogenous [V] |
| Industry | `industry: true` (needs gas) | `industry_natural_gas_demand` | endogenous [V] |

**Surprising dependency:** transport gas demand is built from the Hydrogen
flag, not the gas flag: `gas_transport_demand = hydrogen_enabled`
(`user_interface.jl:292`), and both the variables and
`meet_transport_natural_gas_demand` are created only then (`natural_gas.jl:156-161,
351`). In a gas-only run the demand CSV is read and validated, but no
constraint uses it, so the gas-only system serves only the power plants. [V]
Why the author tied it to Hydrogen **[U]**: presumably because in the
reference model transport demand is a competitor for gas to be replaced by
hydrogen.

Transport demand is spread evenly over the year (`/ (8760 * 13.9)`; 8760 is
hard-coded), with no hourly or weather shape. Germany period 1 is
506,030,486 MWh/yr, about 4156 t/h. Total demand across nodes falls from 2334
TWh/yr in period 1 to 0 in periods 6 and 7. Unmet demand is shed at the
curtail cost. [V] The file header says EUR/MWh but the constant multiplies a
ton/h variable (`natural_gas.jl:437-445`); whether that matches the reference
model is **[U]**.

## 5. Cross-sector interactions

All links are by **node-name equality**; there are no mapping sets (see
`01_integrated_architecture.md`).

| Direction | What is exchanged | Direct math coupling? | Reference |
|---|---|---|---|
| Power to Gas | `genOperational` of gas generators sets gas burned | **yes** | `natural_gas.jl:271-280, 366-403` |
| Gas to Power | pipeline electricity added to the power `flow_balance` | **yes** | `natural_gas.jl:165-181`; `model_definition.jl:252-255` |
| Gas to Power cost | generator fuel price replaced by terminal prices | cost logic, no variable | `utils.jl:268-273` |
| Gas to Power emissions | CO2 factor of gas generators in the emission cap; CCS plants have a smaller factor | shared data | `genCO2TypeFactor.csv` [V] |
| Hydrogen to Gas | reformers draw gas | **yes** | `natural_gas.jl:386-392` |
| Hydrogen to Gas | repurposed pipelines reduce gas capacity | **yes** | `natural_gas.jl:342-349` |
| Gas to CO2 (via Hydrogen) | captured CO2 of gas CCS plants and of reformers feeds the CO2 network | **yes**, inside the hydrogen module | `hydrogen.jl:519-548` |
| Industry to Gas | industry gas use | **yes** | `industry.jl:319-344` |

CO2 is not a separate module; it is built inside the hydrogen functions and is
active with `hydrogen: true`. The gas module itself contains no CO2 variable.
Gas combustion emissions are calculated on the power side, from generation,
efficiency and emission factors. [V; the emission-cap constraint line was not
re-read for this note]

## 6. Economic logic

There is one objective: minimise discounted total system cost (`model_definition.jl:202`).
Gas contributes two terms (`natural_gas.jl:420-445`) [V]:

1. `natural_gas_terminal_import`: sum of weight x terminal price x import.
   The weight is `objective_weight(..., "avg_year")`, i.e. probability x
   discount x period length.
2. `natural_gas_transport_shedding`: weight x curtail cost x shed gas.

Because pipelines, storage and gas transport have no cost, the choice among
sources is a trade-off between **price** and **limits**. [C, based on [V]
formulation]:

- Cheaper terminals are used first until their hourly capacity, their reserve,
  or pipeline capacity towards the demand node blocks them.
- The reserve couples periods: using domestic gas early lowers what is left
  later, so the model allocates finite reserves to the periods where
  alternatives are most expensive.
- Gas power plants do not pay a fuel price directly; their fuel cost is the
  shadow of the gas balance, so prices of the terminal prices propagate to
  electricity cost. They still pay variable O&M and, via the emission price,
  carbon cost.
- Hypothetical illustration (not data): if node A has gas at 278 EUR/t but only
  100 t/h of pipeline to node B, and B has LNG at 800 EUR/t, the model buys
  LNG at B only for the demand that exceeds 100 t/h.

No scenario run was made, so none of these behaviours are confirmed in
results.

## 7. NUTS2 relevance (observations only)

- Gas nodes are power-node names. Gas generators must sit at a gas node
  (`empire_sets.jl:650-655`) and the balance picks them up only by node-name
  equality. [V]
- Gas data is national and uses a single unit: supply capacity, reserves,
  storage, demand and pipeline capacity are totals attached to named nodes,
  not intensities. [V]
- Pipelines are a separate directed graph with its own capacities and no
  lengths, so any regional graph would need its own arc data. [V]
- Demand is per onshore node and strategic period only. [V]
- `OnshoreNode` is stored with the gas sets but also decides the hydrogen and
  CO2 network. [V]
- Hydrogen/CO2 pipeline arcs come from electricity arcs, but gas arcs do not;
  gas-only arcs (those touching Sleipner and Draupner) cannot be repurposed
  to hydrogen. [I from `read_csv.jl:256-262`; **[U]**]

## 8. Open questions

1. Curtail cost units (EUR/MWh label vs per-ton application). **[U]**
2. Italy reserve applied to two terminals. **[U]**
3. Intent of tying transport demand to the Hydrogen flag. **[U]**
4. Draupner and Sleipner: no terminal, a `GasOCGT` generator and electricity
   links to NO2/NO5 only; their gas supply is by pipeline from the connected
   nodes. Behaviour in the power balance not traced. **[U]**
5. Storage horizon semantics (block reset to 50 %) not exhaustively traced.
   **[U]**
6. Why 14 pipeline arcs have zero capacity. **[U]**
7. Gas-only run: confirm that no other constraint reads the transport demand
   data. **[U]**
