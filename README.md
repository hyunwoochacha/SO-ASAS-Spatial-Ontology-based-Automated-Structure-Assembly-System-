# RevitBridge

OWL/RDF 온톨로지 기반의 Autodesk Revit 교량 구조물 자동 배치 플러그인

## Overview

RevitBridge는 교량 구조물의 하부구조(교각, 교대)와 상부구조(슬래브, 방호벽)를 OWL 온톨로지(.ttl)로 정의하고, SPARQL 쿼리를 통해 요소 간 관계를 추출한 후, Revit API를 사용하여 자동으로 배치·결합하는 플러그인입니다.

## Architecture

```
RevitBridge/
├── RevitBridge/                    # Revit Addin 프로젝트
│   ├── Command.cs                  # 핵심 로직 (모든 배치/결합 명령)
│   ├── MainWindow.xaml/.cs         # 메인 UI (TreeView 기반 메뉴)
│   ├── UserControl1.xaml/.cs       # 보조 UI
│   ├── ViewModels/                 # MVVM ViewModel
│   ├── CombinationChoiceWindow.xaml/.cs
│   ├── PierSelectionWindow.xaml/.cs
│   ├── AbutmentSelectionWindow.xaml/.cs
│   └── ...
├── ontology/                       # OWL/RDF 온톨로지 파일 (Raw Data)
│   ├── WBS_Bridge.ttl              # 교각(A2) + 교대(A1) + 상부구조
│   ├── WBS_Bridge_RevitBridge2.ttl # 교각(A1) + 교대(A2) + 상부구조
│   └── WBS_Bridge2.ttl             # 교각(A1~A5) + 교대(A1,A2) + 상부구조
└── App1/                           # 독립 실행 WPF 앱
```

## Command Classes

`Command.cs`에 정의된 주요 클래스:

| Class | 기능 | 사용 TTL |
|-------|------|----------|
| `BridgeController` | 메인 UI 실행 (IExternalCommand) | - |
| `ArrangePierElements` | 교각 요소 배치 (A1→A7 반복) | WBS_Bridge2.ttl |
| `ArrangeAbutmentElements` | 교대 요소 배치 | WBS_Bridge.ttl |
| `ArrangeSuperStructureElements` | 상부구조 배치 | WBS_Bridge.ttl |
| `CombineBridgeElements_Pier_Pier` | 교각-상부-교각 결합 | WBS_Bridge2.ttl |
| `CombineBridgeElements_Pier_Abutment` | 교각-상부-교대 결합 | WBS_Bridge.ttl |
| `CombineBridgeElements_Abutment_Pier` | 교대-상부-교각 결합 | WBS_Bridge_RevitBridge2.ttl |
| `CombineBridgeElements_Abutment_Abutment` | 교대-상부-교대 결합 | WBS_Bridge_RevitBridge2.ttl |

## Ontology Structure (TTL)

### Classes

```turtle
ex:Bridge → ex:SuperStructure → ex:Slab, ex:Protectivewall
          → ex:SubStructure   → ex:Abutment → ex:AbutmentFooting, ex:AbutmentFoundation,
                                               ex:AbutmentWall, ex:AbutmentWingWall, ex:AbutmentCap
                              → ex:Pier     → ex:PierFooting, ex:PierFoundation,
                                               ex:PierColumn, ex:PierCoping
```

### Object Properties

| Property | 의미 | 예시 |
|----------|------|------|
| `ex:isAttachedTo` | 구조적 연결 | PierCoping → PierColumn |
| `ex:isAdjacentOf` | 인접 관계 | WingWall → AbutmentWall |
| `ex:isPuttingOn` | 위에 위치 | ProtectiveWall → Slab |
| `ex:isPartOf` | 소속 관계 | All → Bridge_Instance |
| `ex:isBearingWith` | 지지 관계 | PierCoping → Slab |

### Data Properties

| Property | Type | 설명 |
|----------|------|------|
| `ex:name` | xsd:string | Revit 요소 이름 (매칭 키) |
| `ex:material` | xsd:string | 재료 |
| `ex:staLocation` | xsd:string | 측점 위치 |
| `ex:width/length/height` | xsd:float | 치수 |

### Instance Naming Convention

```
URI:  ex:{PierID}_Pier{Component}_Instance    (예: ex:A1_PierColumn_Instance)
Name: ex:name "{PierID}_Pier_{Component}"     (예: "A1_Pier_Column")
```

현재 정의된 Pier 인스턴스:

| Pier ID | Coping | Column | Foundation | Footing |
|---------|--------|--------|------------|---------|
| A1 | A1_Pier_Coping | A1_Pier_Column | A1_Pier_Foundation | A1_Pier_Footing |
| A2 | A2_Pier_Coping | A2_Pier_Column | A2_Pier_Foundation | A2_Pier_Footing |
| A3 | A3_Pier_Coping | A3_Pier_Column | A3_Pier_Foundation | A3_Pier_Footing |
| A5 | A5_Pier_Coping | A5_Pier_Column | A5_Pier_Foundation | A5_Pier_Footing |

## SPARQL Queries

### 교각 요소 조회 (ArrangePierElements)

각 교각(A1~A7)에 대해 아래 쿼리를 반복 실행:

```sparql
PREFIX ex: <http://example.org/>

SELECT ?copingName ?columnName ?foundationName ?footingName WHERE {
    ex:{PierID}_PierCoping_Instance ex:isAttachedTo ex:{PierID}_PierColumn_Instance .
    ex:{PierID}_PierColumn_Instance ex:isAttachedTo ex:{PierID}_PierFoundation_Instance .
    ex:{PierID}_PierFoundation_Instance ex:isAttachedTo ex:{PierID}_PierFooting_Instance .
    ex:{PierID}_PierCoping_Instance ex:name ?copingName .
    ex:{PierID}_PierColumn_Instance ex:name ?columnName .
    ex:{PierID}_PierFoundation_Instance ex:name ?foundationName .
    ex:{PierID}_PierFooting_Instance ex:name ?footingName .
}
```

### 교대 요소 조회 (CombineBridgeElements)

```sparql
PREFIX ex: <http://example.org/>

SELECT ?footingName ?foundationName ?wallName ?wingLeftName ?wingRightName ?capName WHERE {
    ex:A1_AbutmentFooting_Instance ex:name ?footingName .
    ex:A1_AbutmentFoundation_Instance ex:name ?foundationName .
    ex:A1_AbutmentWall_Instance ex:name ?wallName .
    ex:A1_AbutmentWingWall_Left_Instance ex:name ?wingLeftName .
    ex:A1_AbutmentWingWall_Right_Instance ex:name ?wingRightName .
    ex:A1_AbutmentCap_Instance ex:name ?capName .
}
```

### 상부구조 조회

```sparql
PREFIX ex: <http://example.org/>

SELECT ?slabName ?protectivewallLeftName ?protectivewallRightName WHERE {
    ex:Slab_Instance ex:name ?slabName .
    ex:Protectivewall_Left_Instance ex:name ?protectivewallLeftName .
    ex:Protectivewall_Right_Instance ex:name ?protectivewallRightName .
}
```

## Element Combination Flow

```
1. SPARQL로 TTL에서 요소 이름 추출
2. FindElementByName()으로 Revit 요소 탐색 (case-insensitive)
3. Action Queue에 결합 단계 등록
4. Idling 이벤트 핸들러가 0.5초 간격으로 순차 실행:
   - Footing → Foundation (위로 이동)
   - Foundation → Column (위로 이동)
   - Column → Coping (위로 이동)
5. 각 단계: BoundingBox 중심 계산 → 상위 요소를 하위 요소 위로 이동
```

### 배치 계산 로직

```
TargetZ = LowerElement.Center.Z + (LowerElement.Height / 2) + (UpperElement.Height / 2)
Translation = TargetPosition - CurrentPosition
ElementTransformUtils.MoveElement(doc, element.Id, translation)
```

## Revit Element Names (검증용)

Revit 모델에서 사용하는 실제 요소 이름:

### Pier (교각)
- `A1_Pier_Coping`, `A1_Pier_Column`, `A1_Pier_Foundation`, `A1_Pier_Footing`
- `A3_Pier_Coping`, `A3_Pier_Column`, `A3_Pier_Foundation`, `A3_Pier_Footing`
- `A5_Pier_Coping`, `A5_Pier_Column`, `A5_Pier_Foundation`, `A5_Pier_Footing`

### Abutment (교대)
- `A1_Abutment_Footing`, `A1_Abutment_Foundation`, `A1_Abutment_Wall`
- `A1_Abutment_WingWall_Left`, `A1_Abutment_WingWall_Right`, `A1_Abutment_Cap`
- `A2_Abutment_*` (동일 패턴)

### Superstructure (상부구조)
- `Slab`
- `ProtectiveWall_Left`
- `ProtectiveWall_Right`

## Raw Data

검증을 위한 온톨로지 원본 데이터는 `ontology/` 폴더에 있습니다:

- [`ontology/WBS_Bridge.ttl`](ontology/WBS_Bridge.ttl) - Pier_Abutment 명령용
- [`ontology/WBS_Bridge_RevitBridge2.ttl`](ontology/WBS_Bridge_RevitBridge2.ttl) - Abutment_Pier / Abutment_Abutment 명령용
- [`ontology/WBS_Bridge2.ttl`](ontology/WBS_Bridge2.ttl) - Pier_Pier / ArrangePierElements 명령용

## Requirements

- Autodesk Revit 2024+
- .NET Framework 4.8
- [dotNetRDF](https://www.dotnetrdf.org/) (VDS.RDF) - SPARQL/RDF 처리

## License

MIT License
