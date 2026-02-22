# SO-ASAS: Spatial Ontology-based Automated Structure Assembly System

A Revit plugin that automates bridge structure assembly using OWL/RDF spatial ontology and SPARQL queries.

## Overview

SO-ASAS defines the spatial relationships of bridge structural components (piers, abutments, slabs, protective walls) through an OWL ontology in Turtle (.ttl) format. The system extracts component relationships via SPARQL queries and automatically assembles elements in Autodesk Revit using the Revit API.

### System Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Revit Environment                 │
│  ┌───────────┐    ┌────────────┐    ┌────────────┐  │
│  │  Revit    │◄───│  Assembly  │◄───│  SPARQL    │  │
│  │  Elements │    │  Engine    │    │  Processor │  │
│  └───────────┘    └────────────┘    └─────┬──────┘  │
│                                           │         │
│                                    ┌──────┴──────┐  │
│                                    │  OWL/RDF    │  │
│                                    │  Ontology   │  │
│                                    │  (.ttl)     │  │
│                                    └─────────────┘  │
└─────────────────────────────────────────────────────┘
```

## Ontology Design

### Class Hierarchy

```turtle
ex:WBS_Component
 └── ex:Bridge
      ├── ex:SuperStructure
      │    ├── ex:Slab
      │    └── ex:Protectivewall
      └── ex:SubStructure
           ├── ex:Abutment
           │    ├── ex:AbutmentFooting
           │    ├── ex:AbutmentFoundation
           │    ├── ex:AbutmentWall
           │    ├── ex:AbutmentWingWall
           │    └── ex:AbutmentCap
           └── ex:Pier
                ├── ex:PierFooting
                ├── ex:PierFoundation
                ├── ex:PierColumn
                └── ex:PierCoping
```

### Spatial Relationship Properties

| Property | Description | Example |
|----------|-------------|---------|
| `ex:isAttachedTo` | Structural connection between vertically stacked components | PierCoping → PierColumn |
| `ex:isAdjacentOf` | Lateral adjacency relationship | WingWall → AbutmentWall |
| `ex:isPuttingOn` | Component placed on top of another | ProtectiveWall → Slab |
| `ex:isPartOf` | Part-whole composition | All components → Bridge |
| `ex:isBearingWith` | Load-bearing support relationship | PierCoping → Slab |

### Data Properties

| Property | Type | Description |
|----------|------|-------------|
| `ex:name` | xsd:string | Element name for Revit matching |
| `ex:material` | xsd:string | Material type |
| `ex:staLocation` | xsd:string | Station location |

## SPARQL Query Examples

### Pier Element Query

Iterates from A1 to A7, extracting element names through spatial relationships:

```sparql
PREFIX ex: <http://example.org/>

SELECT ?copingName ?columnName ?foundationName ?footingName WHERE {
    ex:A1_PierCoping_Instance ex:isAttachedTo ex:A1_PierColumn_Instance .
    ex:A1_PierColumn_Instance ex:isAttachedTo ex:A1_PierFoundation_Instance .
    ex:A1_PierFoundation_Instance ex:isAttachedTo ex:A1_PierFooting_Instance .
    ex:A1_PierCoping_Instance ex:name ?copingName .
    ex:A1_PierColumn_Instance ex:name ?columnName .
    ex:A1_PierFoundation_Instance ex:name ?foundationName .
    ex:A1_PierFooting_Instance ex:name ?footingName .
}
```

### Superstructure Query

```sparql
PREFIX ex: <http://example.org/>

SELECT ?slabName ?protectivewallLeftName ?protectivewallRightName WHERE {
    ex:Slab_Instance ex:name ?slabName .
    ex:Protectivewall_Left_Instance ex:name ?protectivewallLeftName .
    ex:Protectivewall_Right_Instance ex:name ?protectivewallRightName .
}
```

## Raw Data

Ontology source files for verification are available in the [`ontology/`](ontology/) folder:

| File | Description |
|------|-------------|
| [`WBS_Bridge.ttl`](ontology/WBS_Bridge.ttl) | Pier-Abutment configuration |
| [`WBS_Bridge2.ttl`](ontology/WBS_Bridge2.ttl) | Multi-pier (A1–A5) + Abutment configuration |
| [`WBS_Bridge_RevitBridge2.ttl`](ontology/WBS_Bridge_RevitBridge2.ttl) | Abutment-Pier configuration |

## Requirements

- Autodesk Revit 2024+
- .NET Framework 4.8
- [dotNetRDF](https://www.dotnetrdf.org/) (VDS.RDF)

## License

MIT License
