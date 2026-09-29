---
template:
  id: https://www.modelware.io/sierra/operational-analysis/traceability
  name: "Traceability Coverage"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Traceability Coverage

The Sierra method says a concern becomes an objective, an objective requires capabilities,
and a capability is provided by an entity. Each link is checked while authoring. Nothing
checks that the chain reaches the end.

This analysis does. It is method-owned rather than project-owned for the reason the module
gives: if every project wrote its own coverage view, "covered" would mean something
slightly different in each, and a fix to one would not reach the others.

## Which capabilities does nobody provide?

**Question.** Every capability is supposed to be assigned to an entity that provides it.
Which ones are not?

```table
---
orderBy: ["Capability asc"]
---
PREFIX base: <https://www.modelware.io/sierra/base#>
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

SELECT ?Capability ?Objective ?Description
WHERE {
  ?capability a mission:Capability .
  OPTIONAL { ?capability base:description ?Description }
  OPTIONAL {
    ?capability mission:isRequiredBy ?objective .
    BIND(REPLACE(STR(?objective), "^.*[#/]", "") AS ?Objective)
  }
  FILTER NOT EXISTS { ?entity entity:hasCapability ?capability }
  BIND(REPLACE(STR(?capability), "^.*[#/]", "") AS ?Capability)
}
```

**Why `FILTER NOT EXISTS` and not reasoning.** Under the open world assumption, the absence
of an assignment means unknown, never missing. This query asks the closed-world question -
is an assignment *recorded in this scope* - which is the only form in which the question has
an answer.

## Which objectives does nobody deliver?

**Question.** An objective is delivered by the entities that provide its capabilities. Which
objectives have no entity behind them at all?

```matrix
---
rowColumnLabel: Objective / Entity
stylesheet:
  - selector: cell [Number(value) > 0]
    style:
      background-color: lightgreen
  - selector: row [cells.every(c => Number(c.value) === 0)]
    style:
      background-color: "#ffd6d6"
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  ?row a mission:Objective .
  ?column a entity:Entity .

  OPTIONAL {
    SELECT ?row ?column (COUNT(*) AS ?n)
    WHERE {
      ?capability mission:isRequiredBy ?row .
      ?column entity:hasCapability ?capability .
    }
    GROUP BY ?row ?column
  }
}
ORDER BY ?row ?column
```

**Read the zeros, not the greens.** The grid is built from the full objective-by-entity
cross product before anything is counted, so a pair with no path shows a zero rather than
vanishing. A matrix assembled from the links that exist has no empty cells and therefore
cannot answer this question. An entirely red row is an objective that no entity delivers.

## Where does the chain break?

**Question.** Seen end to end, where does a concern stop reaching an entity?

```graph
---
layout:
  mode: dag
  dag:
    rankDir: LR
  fit: true
group:
  byPredicate: true
stylesheet:
  - selector: node [value.includes("Needs")]
    group: concern
    style:
      fill: "#ffe08a"
  - selector: node [outgoing.some(e => e.label === "requires")]
    group: objective
    style:
      fill: "#a5d8ff"
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

CONSTRUCT {
  ?concern   mission:derives      ?objective .
  ?objective mission:requires     ?capability .
  ?entity    entity:hasCapability ?capability .
}
WHERE {
  ?capability mission:isRequiredBy ?objective .
  OPTIONAL { ?objective mission:isDerivedFrom ?concern }
  OPTIONAL { ?entity entity:hasCapability ?capability }
}
```

A capability with nothing to its right is one nobody provides. The graph shows the same
fact as the table above, but it shows *where* in the chain the break sits, which the table
cannot.
