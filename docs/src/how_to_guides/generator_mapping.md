# How to customize the generator mapping

PowerAnalytics groups generators into fuel categories — `Coal`, `NG-CC`, `Wind`, `Storage`
and so on — using a YAML file. This page describes that file's format, the order in which
its rules are matched, and how to supply your own.

The built-in file lives at `deps/generator_mapping.yaml`, reachable as
`PowerAnalytics.GENERATOR_MAPPING_FILE`.

## File format

Each top-level key is a category name, and its value is a list of rules that map to it:

```yaml
PV:
  - {gentype: Any, primemover: PVe, fuel: null}
Wind:
  - {gentype: Any, primemover: WT, fuel: null}
  - {gentype: Any, primemover: WS, fuel: null}
NG-CT:
  - {gentype: Any, primemover: GT, fuel: NATURAL_GAS}
Biopower:
  - {gentype: Any, primemover: null, fuel: MUNICIPAL_WASTE}
Hydropower:
  - {gentype: HydroGen, primemover: null, fuel: null}
```

A rule has four fields, all optional:

| field          | default | meaning                                                                                           |
|:-------------- |:------- |:------------------------------------------------------------------------------------------------- |
| `gentype`      | `Any`   | a `PowerSystems.Component` type name, matched against the component's type **and its supertypes** |
| `primemover`   | `null`  | a `PowerSystems.PrimeMovers` member, e.g. `GT`, `WT`, `PVe`                                       |
| `fuel`         | `null`  | a `PowerSystems.ThermalFuels` member, e.g. `NATURAL_GAS`                                          |
| `ext_category` | `null`  | a value read from the component's `ext` dictionary                                                |

`null` means "don't care" — the rule matches regardless of that attribute. Enum names must
match PowerSystems exactly; a name that does not exist raises a `KeyError` when
PowerAnalytics loads, so typos fail loudly rather than silently never matching.

Two rules with identical `(gentype, fuel, primemover, ext_category)` raise an error at load
time, so a category cannot be shadowed by accident.

### The `__META` block

```yaml
__META:
  non_generators:
    - Storage
    - Load
    - Source
```

Categories listed under `non_generators` are excluded from generator-only views. This is
what separates `parse_generator_categories` from `parse_injector_categories`.

## How a component is matched

`get_generator_category` walks the rules in a fixed order and takes the **first** match:

```
for t in supertypes(gentype),
    pm in (primemover, nothing),
    f in (fuel, nothing),
    ext in (ext, nothing)
```

Three consequences worth internalizing:

 1. **Type specificity dominates.** The component's own type is tried before its supertypes,
    and `Any` is the *last* supertype. So `{gentype: HydroGen, …}` beats `{gentype: Any, …}`
    for a `HydroTurbine`, and rules written against `Any` act as general fallbacks.
 2. **Prime mover beats fuel.** Within one type level, the component's actual prime mover is
    tried before `nothing`. So `{primemover: GT, fuel: NATURAL_GAS}` → `NG-CT` wins over the
    fuel-only `{primemover: null, fuel: NATURAL_GAS}` → `NG-Steam`. This is what lets
    fuel-only rules act as catch-alls without shadowing the specific ones.
 3. **Fuel-only rules are the safety net.** A rule with `primemover: null` matches any prime
    mover carrying that fuel. The built-in file has one for every member of
    `PowerSystems.ThermalFuels`, so no fuel-bearing thermal unit goes uncategorized whatever
    its prime mover.

## There is deliberately no blanket catch-all

The built-in file has no `{gentype: Any, primemover: null, fuel: null}` rule, and adding one
is not recommended. Without it, a component whose combination matches nothing produces an
`@error "No mapping defined ..."` naming the exact combination, and lands in the default
`Other` category. That error is the only signal that a system contains something the mapping
does not describe; a blanket rule would silence it and quietly mis-attribute the energy.

If you are seeing that error, the fix is to add the specific rule it names, not a catch-all.

This can only arise for components with **no fuel** — renewables and storage, which are
categorized by prime mover alone. Every thermal fuel is already covered.

## Using your own file

Copy `deps/generator_mapping.yaml`, edit it, and pass the path:

```julia
using PowerAnalytics

# Legacy fuel-dictionary path
fuel_dict = make_fuel_dictionary(sys; generator_mapping_file = "my_mapping.yaml")

# ComponentSelector path
categories = parse_injector_categories("my_mapping.yaml")   # includes Storage/Load/Source
generators = parse_generator_categories("my_mapping.yaml")  # excludes them
```

Both read the same file. `parse_generator_mapping_file` returns the selector dictionary and
the `__META` block together if you need both.

To check your file covers what you expect, sweep the enums:

```julia
using PowerAnalytics, PowerSystems
mapping = PowerAnalytics.get_generator_mapping("my_mapping.yaml")
unmapped = [
    (f, pm)
    for f in instances(ThermalFuels), pm in instances(PrimeMovers)
    if PowerAnalytics.get_generator_category(
        ThermalStandard, string(f), pm, nothing, mapping) === nothing
]
```

An empty result means every fuel and prime-mover combination resolves.

## Naming categories

Category names are a public contract. PowerGraphics.jl keys its color palettes off them, so
renaming a category silently drops its color. If you add a category, add a matching entry to
your PowerGraphics `color-palette.yaml`.
