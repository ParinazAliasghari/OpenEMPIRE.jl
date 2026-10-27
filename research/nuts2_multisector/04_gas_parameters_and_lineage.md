# Natural Gas documentation 2/5: parameters and data lineage

Part 2 of the series (see `03_gas_sets_and_csv_dictionary.md` for sets and CSV
files). Commit `cbf353d3cd033cb0ebab1a062a0986e4560290b8`, data
`data/full_model_int`. Paths are relative to `src/` unless they start with
`data/`. Nothing was run.

Labels: **[V]** verified, **[I]** interpretation, **[U]** unverified.

---

## 1. Where parameters live

All numeric gas inputs are fields of `NaturalGasParams`
(`empire_structs.jl:11-30`), stored as `par.NaturalGas` inside `EmpireParams`.
Table-backed fields default to empty dictionaries; scalars carry defaults that
reproduce InternalEMPIRE's fixed assumptions. [V]

Lifecycle of a gas parameter:

```
CSV  ->  _read_natural_gas_params_csv (read_csv.jl:735)  ->  par.NaturalGas
     ->  validate_natural_gas (empire_structs.jl:572-707, fatal if issues,
         user_interface.jl:257)
     ->  accessor with default (empire_structs.jl:460-480)
     ->  constraint / objective / result writer
```

Gas parameters are not modified by any preprocessing pass; the only derived
quantities are listed in section 3. [V]

## 2. Parameter dictionary

Notation: n node, (i,j) arc, k terminal type, g gas generator, p strategic
period index, s gas scenario, t operational period.

| Symbol | Julia field / accessor | Indices | Unit | Source | Default if absent | Accessor |
|---|---|---|---|---|---|---|
| $C^{pipe}_{ij}$ | `pipelineCapacity`, `natural_gas_pipeline_capacity(par,from,to)` | arc | ton/h | `PipelineCapacity.csv` | 0.0 | `empire_structs.jl:460` |
| $\pi$ | `pipelinePowerDemandPerTon` | scalar | MWh/ton | `PipelineElectricityUse.csv` | struct default 0.0 (file mandatory) | `empire_structs.jl:14` |
| $S_n$ | `storageCapacity`, `natural_gas_storage_capacity` | node | ton | `StorageCapacity.csv` | 0.0 | `:462` |
| $R_n$ | `reserves`, `natural_gas_reserves` | node | ton | `Reserves.csv` | 0.0 | `:464` |
| $C^{term}_{n,k,p}$ | `terminalCapacity`, `natural_gas_terminal_capacity` | node, terminal, period | ton/h | `TerminalCapacity.csv` | 0.0 | `:466` |
| $c_{n,k,p,s}$ | `terminalCost`, `natural_gas_terminal_cost` | node, terminal, period, gas scenario | EUR/ton | `TerminalCost.csv` | 99999.0 | `:468-478` |
| $D_{n,p}$ | `transportDemand`, `natural_gas_transport_demand` | onshore node, period | MWh/yr | `NaturalGasDemand.csv` | 0.0 | `:479` |
| $\kappa$ | `transportCurtailCost` | scalar | labelled EUR/MWh | `CurtailCost.csv` | struct default 10000.0; file value 100000 | `:23` |
| $\mu$ | `mwhPerTon` | scalar | MWh/ton | **code constant** | 13.9 | `:24` |
| $\phi_0$ | `storageInitialFraction` | scalar | fraction | code constant | 0.5 | `:25` |
| $\eta^{ch}$ | `storageChargeEfficiency` | scalar | fraction | code constant | 1.0 | `:26` |
| $\eta^{dis}$ | `storageDischargeEfficiency` | scalar | fraction | code constant | 1.0 | `:27` |
| W | `weatherScenarioCount` | scalar | count | config `number_of_scenarios` | 1 | `:28` |
| G | `gasScenarioCount` | scalar | count | config `number_of_gas_scenarios` | 1 (validation forces 1) | `:29` |
| $\eta^{gen}_{g,p}$ | `par.genEfficiency[g][p]` | generator, period | fraction | `Generator/genEfficiency.csv` (power module) | missing profile is a validation error | `empire_structs.jl:665-673` |
| ρ | `NATURAL_GAS_ROW_SCALE` | constant | – | code | 1e-3 | `natural_gas.jl:11` |

Default behaviours worth knowing: an absent capacity is zero (a silent
closure), an absent terminal cost is 99999 EUR/ton, and an absent demand is
zero. Validation makes missing capacity, cost, reserves and demand keys fatal
whenever the module is on, so the defaults are not reachable through the normal
path. Missing **storage** capacity is not validated, so un-listed nodes silently
get zero storage. [V]

Hard-coded checks (`empire_structs.jl:579-600`): scalars finite and
non-negative; `mwhPerTon > 0`; fraction and efficiencies at most 1;
`weatherScenarioCount > 0`; `gasScenarioCount == 1`. [V]

## 3. Derived parameters and transformations

| Derived quantity | Formula | Where | Notes |
|---|---|---|---|
| Hourly transport demand | $d_{n,p} = D_{n,p} / (8760\,\mu)$ ton/h | `natural_gas.jl:359-364` | 8760 is hard-coded and is **not** the model's `operational_hours_per_year`; demand is spread flat over the year |
| Reserve row coefficient | $\rho \cdot \text{duration\_strat}(sp)\cdot\text{multiple\_strat}(sp,t)$ | `:215-221` | `duration_strat` is the number of years in the strategic period (5 in the shipped config), `multiple_strat` scales a modelled representative period to the hours of a year it stands for (in the TimeStruct test, regular seasons get `(8760 - 3*24)/(4*168)` and peak days get 1; `test/test_timestruct.jl:10-14`) |
| Reserve right-hand side | $\rho R_n$ | `:233` | row scaled by 1e-3 to keep coefficients comparable (`:7-11`) |
| Storage rows | all terms scaled by $\rho$ | `:304-334` | the dual is rescaled in the output writer (`results.jl:1450-1455`) |
| Generator gas input | $q_{n,g,t} = P_{n,g,t}/(\eta^{gen}_{g,p}\,\mu)$ | `:271-280` | enters via a constraint, not a parameter |
| Pipeline electricity at node n | $\pi \sum_{j \in \text{out}(n)} f_{nj,t}$ MW | `:165-181` | an affine expression, subtracted in the power balance |
| Generator marginal cost (gas generators) | $\left(\frac{3.6}{\eta}\right)(0 + \text{carbon}) + \text{varOM}$ in EUR/MWh | `utils.jl:228-282` | fuel term set to zero when gas is on; carbon term uses `co2_price` and `co2_content`, with 90 % removal for CCS technologies |
| Scenario split | weather index and gas index from the combined scenario index | `scenario.jl:92-95` | gas index is always 1 because G = 1 |

Unit remarks:
- `3.6` converts MWh to GJ (1 MWh = 3.6 GJ) in the generator cost and emission
  formulas. [V]
- Flows are ton per hour, and the objective multiplies by a weight that includes
  the number of hours each operational period represents (see doc 07). [V]
- 13.9 MWh/ton (about 50 MJ/kg) is used for the energy content of every gas
  conversion, including Hydrogen reformers and Industry. [V]

## 4. Lineage per parameter (defined / modified / used)

| Parameter | Defined | Modified | Used |
|---|---|---|---|
| `pipelineCapacity` | `read_csv.jl:750-773` | never | `natural_gas.jl:349` (limit); `hydrogen.jl:638` (repurposing bound); `results.jl:1117` (output) |
| `pipelinePowerDemandPerTon` | `read_csv.jl:775-785` | never | `natural_gas.jl:175`; `results.jl` pipeline table |
| `storageCapacity` | `read_csv.jl:787-798` | never | `natural_gas.jl:307, 325, 334`; `results.jl:1221` |
| `reserves` | `read_csv.jl:787-798` | converter resolved duplicates before the file | `natural_gas.jl:233` |
| `terminalCapacity` | `read_csv.jl:800-829` | never | `natural_gas.jl:288`; `results.jl:1062` |
| `terminalCost` | `read_csv.jl:831-870` | converter resolved duplicates | `natural_gas.jl:432-435`; `results.jl:1063` |
| `transportDemand` | `read_csv.jl:872-900` | never | `natural_gas.jl:359`; `results.jl:1268` |
| `transportCurtailCost` | `read_csv.jl:902-912` | never | `natural_gas.jl:441`; **also** `hydrogen.jl:890-898` |
| `mwhPerTon` | `empire_structs.jl:24` | never | `natural_gas.jl:277, 364`; `hydrogen.jl:334, 624`; `industry.jl:391, 398, 687` |
| storage scalars | `empire_structs.jl:25-27` | never | `natural_gas.jl:307-325, 379` |

## 5. Worked numerical lineage: parameter values from data

| Quantity | Raw value | After transformation | Meaning |
|---|---|---|---|
| Germany DomesticProduction capacity | 481.625 ton/h (all 7 periods) | 481.625 | at 8760 h: 4.22 Mt/yr |
| Germany reserve | 13,368,462.36 ton | scaled row RHS 13,368.46 | binds after about 3.2 years of full output |
| Germany transport demand, period 1 | 506,030,485.89 MWh/yr | 4155.83 ton/h | $=506030485.89/(8760 \cdot 13.9)$ |
| Germany→Netherlands capacity | 2522.33 ton/h | 2522.33 | about 35.1 GWh/h of gas energy |
| Pipeline electricity at that flow | 0.02444604 MWh/ton | 61.66 MW | $2522.33 \times 0.02444604$ |
| GasCCGT, period 1 | efficiency 0.574 | 12.53 ton/h per 100 MW | $100/(0.574 \cdot 13.9)$ |
| Terminal price | 278 EUR/ton | | $=20$ EUR/MWh at 13.9 MWh/ton |

## 6. Cross-sector parameter dependencies

- `mwhPerTon` is a gas parameter but is consumed by Hydrogen (reformer gas,
  pipeline repurposing) and Industry (feedstock demand, CO2 factors).
- `transportCurtailCost` is a gas parameter that also prices Hydrogen-side
  shedding (`hydrogen.jl:890-898`).
- `genEfficiency` is a power parameter that determines gas demand.
- `pipelineCapacity` is read by Hydrogen as the upper bound for repurposed
  capacity. [V]

## 7. NUTS2 observations (no design)

- Capacities, reserves, storage and demand are totals attached to a node name,
  not intensities per area. [V]
- Time keys are integer strategic positions. [V]
- 13.9, 8760, 0.5 and 1.0 are fixed in code. [V]
- Gas arcs have no length parameter, so any distance-based scaling would need
  data that this module does not have. [V]

## 8. Open questions

1. `CurtailCost` unit: label is EUR/MWh, value multiplies a ton/h shedding
   variable. Is this identical to InternalEMPIRE? [U]
2. Whether `8760` should follow `operational_hours_per_year`. The constraint
   hard-codes it, and InternalEMPIRE parity was not re-checked in this audit. [U]
3. `duration_strat` value equals the config `leap_years_investment`; this is the
   `years_period` argument to `TwoLevel` (`model_definition.jl:70`). The
   mapping from YAML to that argument was not traced. [I]
