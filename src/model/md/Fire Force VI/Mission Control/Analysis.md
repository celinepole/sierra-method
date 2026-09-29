---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Fire Force Analysis

Module 4 prescribed how Fire Force should be authored. This notebook measures whether it
was, and what the model says once it is. Each section states an engineering question, shows
live evidence, and says what the evidence means. The numbers come from the model at render
time; the prose is written about meaning, so it does not go stale when a mass changes.

**Scope.** Every query here runs against the Mission Control bundle. The traceability
section is also invoked from a second page at a narrower scope, to show that the same
analysis travels.

---

## 1 · Traceability coverage

**Question.** Does every concern reach an entity that can act on it?

```compose
template: https://www.modelware.io/sierra/operational-analysis/traceability
```

**Interpretation.** One capability, `C9`, is provided by no entity, and the objective that
requires it, `O5`, is therefore the only objective with an entirely empty row in the
matrix - nothing in the operational architecture delivers "deliver payload imagery promptly
to the ground segment".

That objective and that capability arrived with the X-Band downlink increment, together
with the components and ports that implement it. The structure was added and the operational
allocation was not. Neither half is invalid on its own, which is exactly why no shape caught
it: `C9` is a well-formed capability and every entity is a well-formed entity. The hole is
*between* the instances, and only a query that walks the chain can see it.

**Action.** Assign `C9` to the entity that will actually downlink - most plausibly
`FireCloud` or a new ground-segment entity - or retire `O5` if the downlink is out of scope
for this increment.

---

## 2 · Conformance: does the model follow the ports rule?

**Question.** The Ports pattern this method gained in Module 4 says that a declared port is
a boundary something connects to. Which ports does no connection touch?

```table
---
orderBy: ["Component asc", "Port asc"]
stylesheet:
  - selector: cell[col === "Direction" && value]
    target: value
    style:
      padding: 2px 10px
      border-radius: 999px
      font-size: 11px
      font-weight: 600
      color: "#ffffff"
      background-color: "#2563EB"
---
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>

SELECT ?Component ?Port ?Direction
WHERE {
  ?component component:hasPort ?port .
  ?port component:direction ?Direction .
  FILTER NOT EXISTS { ?inbound  oml:hasTarget ?port }
  FILTER NOT EXISTS { ?outbound oml:hasSource ?port }
  BIND(REPLACE(STR(?component), "^.*[#/]", "") AS ?Component)
  BIND(REPLACE(STR(?port), "^.*[#/]", "") AS ?Port)
}
```

**Interpretation.** Four ports are declared and unconnected, and three of them are the
`ServiceSegment` ports added when the Ports pattern was first dogfooded. Declaring the
boundary was the easy half; agreeing what connects to it was left for later, and nothing
was tracking that it had been.

This is the difference in cadence the two modules describe. The authoring rule warns the one
engineer editing one port. This query counts the population at review time and shows that
the debt is systematic rather than incidental.

**Action.** Wire the three `ServiceSegment` ports to the platform, or record in each
description that the far end is outside the system boundary - the answer the VHF interface
already gives.

---

## 3 · Near miss: components that are almost wired

**Question.** Which components are *partly* connected? A component with no ports at all is a
design decision; a component with ports and fewer connections is unfinished work.

```table
---
orderBy: ["Links asc"]
---
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>

SELECT ?Component (COUNT(DISTINCT ?port) AS ?Ports) (COUNT(DISTINCT ?link) AS ?Links)
WHERE {
  ?component component:hasPort ?port .
  OPTIONAL {
    { ?link oml:hasSource ?port }
    UNION
    { ?link oml:hasTarget ?port }
  }
  BIND(REPLACE(STR(?component), "^.*[#/]", "") AS ?Component)
}
GROUP BY ?Component
HAVING (COUNT(DISTINCT ?link) < COUNT(DISTINCT ?port))
```

**Interpretation.** Two components sit between "not started" and "done". `PropulsionSegment`
has two ports and one connection, which is the more interesting of the pair: somebody wired
its power and stopped before its command input, and a rule that only asks "is this port
connected" reports it identically to a component nobody has touched.

A near-miss query is worth writing precisely because a binary conformance check cannot tell
the two apart, and they call for different conversations.

---

## 4 · Mass integrity

**Question.** What do the parts actually weigh, and can the roll-up be trusted?

```chart
---
type: bar
data:
  labels: Unit
  datasets:
    - label: Mass assertions
      data: Assertions
options:
  indexAxis: y
  plugins:
    title:
      display: true
      text: Mass assertions by unit
    legend:
      display: false
---
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>

SELECT ?Unit (COUNT(?part) AS ?Assertions)
WHERE {
  ?part component:mass ?quantity .
  ?quantity oml:unit ?unit .
  BIND(REPLACE(STR(?unit), "^.*[#/]", "") AS ?Unit)
}
GROUP BY ?Unit
ORDER BY DESC(?Assertions)
```

The chart is the whole finding in one bar. What it costs is computed below, because SPARQL
can group by unit but cannot convert between them, and the distribution statistics that show
how badly a naive sum misleads need a library rather than a query.

```python
import statistics

result = await query("""
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>
SELECT ?part ?value ?unit
WHERE {
  ?part component:mass ?quantity .
  ?quantity oml:value ?value .
  OPTIONAL { ?quantity oml:unit ?unit }
}
""")
rows = result["rows"]

TO_KG = {"kg": 1.0, "g": 0.001}

def unit_of(iri):
    return (iri or "kg").replace("#", "/").rstrip("/").rsplit("/", 1)[-1]

def name_of(iri):
    return iri.replace("#", "/").rsplit("/", 1)[-1]

parts = []
for r in rows:
    u = unit_of(r.get("unit"))
    parts.append((name_of(r.get("part", "")), float(r.get("value", 0)), u, TO_KG.get(u, 1.0)))

naive = sum(v for _, v, _, _ in parts)
actual = sum(v * f for _, v, _, f in parts)
kg_values = [v for _, v, u, _ in parts if u == "kg"]
mean = statistics.mean(kg_values)
sd = statistics.stdev(kg_values)
odd = sorted((p for p in parts if p[2] != "kg"), key=lambda p: -p[1])

rows_html = "".join(
    f"<tr><td>{n}</td><td>{v:g} {u}</td><td>{v*f:.3f} kg</td></tr>"
    for n, v, u, f in odd
)

display(f"""
<table class="oml-md-table">
  <thead><tr><th>Measure</th><th>Value</th></tr></thead>
  <tbody>
    <tr><td>Mass assertions</td><td>{len(parts)}</td></tr>
    <tr><td>Sum of the numbers as written</td><td><b>{naive:,.1f}</b></td></tr>
    <tr><td>Sum after converting to kilograms</td><td><b>{actual:,.2f} kg</b></td></tr>
    <tr><td>Overstatement if units are ignored</td><td>{naive - actual:,.1f} kg
        ({(naive - actual) / actual * 100:,.0f}% high)</td></tr>
    <tr><td>Mean of the kilogram population</td><td>{mean:.3f} kg (sd {sd:.3f})</td></tr>
  </tbody>
</table>
<p><b>{len(odd)} assertions are not in kilograms:</b></p>
<table class="oml-md-table">
  <thead><tr><th>Part</th><th>As written</th><th>In kilograms</th></tr></thead>
  <tbody>{rows_html}</tbody>
</table>
""")
```

**Interpretation.** Four of the mass assertions are in grams and the other 115 are in
kilograms. Summed as written they contribute 2,600 - about twenty times the true mass of the
whole spacecraft - so any roll-up that adds `oml:value` without reading `oml:unit` is wrong
by that margin, and wrong in the direction that hides a mass problem behind a fake one.

The four are also the only reason the distribution looks the way it does. Excluding them,
the parts average about 1.1 kg with a standard deviation of 1.8 and a maximum of 14.5; with
them, the mean is twenty times larger than the second-heaviest part in the model.

**This is the case for a script rather than a query.** SPARQL selected the value and the
unit, which is graph work. Converting between units, computing a standard deviation, and
comparing two totals is arithmetic over the selected rows, and forcing it into SPARQL would
have produced something unreadable that still could not do the statistics.

**Action.** Either convert the four to kilograms, or make unit handling explicit in the mass
pattern so the roll-up view normalises before it sums. The second is the better fix, because
the vocabulary already carries the unit and the analysis simply was not reading it.

---

## What these queries cannot settle

A query reports what the model says. It cannot say whether `C9` *should* be assigned to
`FireCloud`, whether the `ServiceSegment` ports belong in this increment, or whether 14.5 kg
is a plausible battery cell. Consistent, valid and complete are three claims, and none of
them is correct.

Nor is silence coverage. Four of the checks written for this notebook returned nothing -
every concern has an objective, every objective has a capability, every requirement has a
stater and a priority, and no composite asserts its own mass. That is a clean result against
the rules somebody thought to encode, which is a smaller claim than a clean model.
