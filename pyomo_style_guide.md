# Pyomo Optimierungsmodell – Style-Vorlage

Allgemeine Struktur- und Werkzeug-Vorlage für MILP/LP-Modelle in Pyomo.
Basiert auf dem Coding-Stil von `thermal_model_main_-_Copy.ipynb`.

---

## 1. Libraries

```python
import pyomo.environ as pyo   # Modellierung + Solver-Interface
import pandas as pd            # Daten laden, Ergebnisse speichern
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import plotly.express as px
import os
```

**Solver:** `cbc` für MILP (enthält Binärvariablen), `glpk` für reine LP.

---

## 2. Notebook-Struktur

```
## Imports
## Assumptions      ← Freitext: was wird vereinfacht/angenommen?
## Data             ← Daten laden, aufbereiten
## Parameters       ← reine Python-Zahlen (keine pyo-Objekte)
## Model            ← m = pyo.ConcreteModel(), Sets, pyo.Param
## Variables        ← pyo.Var
## Constraints      ← pyo.Constraint mit Rule-Funktionen
## Objective        ← pyo.Objective
## Run Solver       ← solver.solve() + assert auf Status
## Extract Results  ← get_values() → DataFrame, CSV
## Plot Results     ← Plotly
```

---

## 3. Modell-Objekt & Sets

```python
m = pyo.ConcreteModel()

# Zeitindex (oder anderer Index)
m.T = pyo.Set(initialize=range(len(df)))
```

---

## 4. Parameter vs. Variablen

- **Parameter** = vor dem Solve bekannte Werte → normale Python-Zahlen oder `pyo.Param`
- **Variablen** = vom Solver zu bestimmende Werte → `pyo.Var`

```python
# Zeitreihen-Parameter
m.price = pyo.Param(m.T, initialize=dict(enumerate(df['price'])))

# Skalare bleiben einfache Python-Variablen
max_capacity = 100
cost_rate    = 50
```

---

## 5. Variablen

```python
# Stetige Variable mit Bounds
m.x = pyo.Var(m.T, bounds=(0.0, max_capacity))

# Binäre Variable (0 oder 1) – macht das Problem zum MILP
m.on = pyo.Var(m.T, within=pyo.Binary)

# Hilfsvariable ohne explizite Bounds (z.B. für €-Terme)
m.cost = pyo.Var(m.T)
```

---

## 6. Constraints

Jeder Constraint: **Rule-Funktion** + Registrierung. Randfälle (t=0) explizit behandeln.

```python
def my_constraint_rule(m, t):
    if t == 0:
        return m.x[t] == initial_value
    else:
        return m.x[t] == m.x[t - 1] + m.delta[t]

m.my_constraint = pyo.Constraint(m.T, rule=my_constraint_rule)
```

### Grundregeln

| Erlaubt | Verboten |
|---|---|
| `parameter * variable` | `variable * variable` (nicht-linear) |
| `variable + variable` | `if m.variable[t]:` (Pyomo-Var in if) |
| `pyo.quicksum(...)` | Pyomo-Var als Python-Zahl behandeln |

### Entweder-oder-Logik (Big-M-Trick)

Wenn eine Bedingung von einer Binärvariable abhängen soll — kein `if`, sondern:

```python
# Wenn on=1: x >= lower_bound   (aktiv)
# Wenn on=0: x >= 0             (inaktiv, weil lower_bound * 0 = 0)
return m.x[t] >= lower_bound * m.on[t]

# Wenn on=0: x <= 0  → x == 0  (erzwingt Aus)
# Wenn on=1: x <= upper_bound   (inaktiv)
return m.x[t] <= upper_bound * m.on[t]
```

### Zeitliche Verkettung

```python
def continuity_rule(m, t):
    if t == 0:
        return m.x[t] == initial_value
    else:
        return m.x[t] == m.x[t - 1]   # oder + delta, - discharge, etc.

m.continuity = pyo.Constraint(m.T, rule=continuity_rule)
```

---

## 7. Objective

```python
def objective_rule(m):
    return pyo.quicksum(m.revenue[t] - m.cost[t] for t in m.T)

m.obj = pyo.Objective(rule=objective_rule, sense=pyo.maximize)
# sense=pyo.minimize für Kostenminimierung
```

---

## 8. Solver & Statusprüfung

```python
solver = pyo.SolverFactory("cbc")
result = solver.solve(m, tee=False)   # tee=True für verbose logs

# Immer prüfen – nie blind Ergebnisse lesen
assert result.solver.status == pyo.SolverStatus.ok
assert result.solver.termination_condition == pyo.TerminationCondition.optimal
```

---

## 9. Ergebnisse extrahieren

```python
results_df = df.copy()
results_df["x"]    = m.x.get_values().values()
results_df["on"]   = m.on.get_values().values()
results_df["cost"] = m.cost.get_values().values()

print("Objective:", pyo.value(m.obj))

os.makedirs("results", exist_ok=True)
results_df.to_csv("results/output.csv", index=False)
```

---

## 10. Checkliste vor dem Solve

- [ ] Alle Variablen haben sinnvolle `bounds`
- [ ] Alle `t == 0` Randfälle in Constraints explizit behandelt
- [ ] Kein `variable * variable` in Constraints
- [ ] Kein `if m.variable[t]:` in Rule-Funktionen
- [ ] Solver-Status per `assert` geprüft
- [ ] `pyo.quicksum` statt `sum()` in der Zielfunktion
