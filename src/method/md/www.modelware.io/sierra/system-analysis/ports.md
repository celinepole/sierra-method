---
template:
  id: https://www.modelware.io/sierra/system-analysis/ports
  name: "Ports"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Component Ports

Define the interface of each component: the ports through which it exchanges signals, energy, or material with other components.

**Why this pattern exists.** Connections can only be drawn between ports, so every connection review starts by asking "does this component actually expose that interface, and in which direction?" Ports were previously added by hand in the connections description, which led to ports without an owner, missing directions, and flows drawn the wrong way between sibling components.

**How to use this page.**

1. In the **Ports** table, add the port. Name it `<Component>.<Name>_In` or `<Component>.<Name>_Out`, set its **Direction**, and describe it.
2. In the **Component Interfaces** tree, open the owning component and add the new port to its **Ports** list. The hierarchy and descriptions are read-only context owned by the *Components* page; only the port list is edited here.
3. Run **Validate** before committing. A port that has not yet been attached to a component is reported until step 2 is done.

**Rules this page enforces.**

- Every port has exactly one direction, `In` or `Out`.
- Every port belongs to a component.
- Between *sibling* components, a connection must flow from an `Out` port to an `In` port. Connections that cross a hierarchy boundary (a parent delegating to a child, or a child exporting through its parent) are exempt, because the same direction is expected on both ends.

**Open questions.** Should ports also be typed by the item they carry (signal, energy, material)? That is deferred until a connection editor is added.

## Ports

```table-editor
---
columns: { this: { label: "Port" } }
orderBy: Component
stylesheet:
  - selector: cell[col === "Direction" && value]
    target: value
    style:
      padding: 4px 12px
      border-radius: 999px
      font-size: 12px
      font-weight: 600
      color: "#ffffff"
  - selector: cell[value === "In"]
    target: value
    style:
      background-color: "#2563EB"
  - selector: cell[value === "Out"]
    target: value
    style:
      background-color: "#EA580C"
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:PortShape
    a sh:NodeShape ;
    sh:targetClass component:Port ;
    sh:rule [
        a sh:SPARQLRule ;
        sh:construct """
            PREFIX component: <https://www.modelware.io/sierra/component#>
            CONSTRUCT { $this component:portOf ?owner }
            WHERE { ?owner component:hasPort $this . }
        """ ;
    ] ;
    sh:property [
        sh:path component:portOf ;
        sh:name "Component" ;
        dash:readOnly true ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path component:direction ;
        sh:name "Direction" ;
        sh:in ( "In" "Out" ) ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:order 2 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 3 ;
    ] ;
    sh:sparql [
        sh:message "This port is not attached to any component. Add it to its owner's Ports list in the Component Interfaces tree below." ;
        sh:select """
            PREFIX component: <https://www.modelware.io/sierra/component#>
            SELECT $this WHERE {
                FILTER NOT EXISTS { ?owner component:hasPort $this . }
            }
        """ ;
    ] ;
    sh:sparql [
        sh:message "This port feeds a connection to a sibling component, but the flow is not Out -> In. Between siblings, the source port must be 'Out' and the target port 'In'. Fix the port directions or reverse the connection." ;
        sh:select """
            PREFIX base: <https://www.modelware.io/sierra/base#>
            PREFIX component: <https://www.modelware.io/sierra/component#>
            PREFIX oml: <http://opencaesar.io/oml#>
            SELECT $this WHERE {
                ?conn a component:Connection ;
                      oml:hasSource $this ;
                      oml:hasTarget ?target .
                ?srcComp component:hasPort $this ;
                         base:isContainedBy ?parent .
                ?tgtComp component:hasPort ?target ;
                         base:isContainedBy ?parent .
                FILTER (?srcComp != ?tgtComp)
                $this component:direction ?srcDir .
                ?target component:direction ?tgtDir .
                FILTER (STR(?srcDir) != "Out" || STR(?tgtDir) != "In")
            }
        """ ;
    ] ;
    .
```

## Component Interfaces

```tree-editor
---
columns: { this: { label: "Component" } }
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .

component:ComponentShape
    a sh:NodeShape ;
    sh:targetClass component:Component ;
    dash:readOnly true ;
    sh:property [
        sh:path base:isContainedBy ;
        sh:name "Container" ;
        sh:class component:Component ;
        dash:composite true ;
    ] ;
    sh:property [
        sh:path component:hasPort ;
        sh:name "Ports" ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        sh:maxCount 1 ;
        sh:order 2 ;
    ] ;
    .