# AutoMapper: Detailed Design Documentation

## 1. System Architecture and Design Principles

### 1.1 Architectural Overview

AutoMapper follows a **modular, layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│           Presentation Layer (CLI)                      │
│              AutoMapper.py                              │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────┐
│           Application Layer (Tools)                     │
│  ┌──────────────┬─────────────────┬──────────────┐    │
│  │ Unified      │ Molecule        │ Map          │    │
│  │ Cleaner      │ Converter       │ Processor    │    │
│  └──────────────┴─────────────────┴──────────────┘    │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────┐
│           Business Logic Layer                          │
│  ┌──────────────┬─────────────────┬──────────────┐    │
│  │ PathSearch   │ AtomObject      │ Queue        │    │
│  │ Algorithm    │ Builder         │ Management   │    │
│  └──────────────┴─────────────────┴──────────────┘    │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────┴────────────────────────────────────┐
│           Data Access Layer                             │
│  ┌──────────────────────┬────────────────────────┐    │
│  │ LammpsSearchFuncs    │ LammpsTreatmentFuncs   │    │
│  │ (File Parsing)       │ (File I/O)             │    │
│  └──────────────────────┴────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Design Principles

**1. Single Responsibility Principle**
- Each module has one primary responsibility
- Example: `LammpsSearchFuncs` only searches/parses, never modifies data

**2. Open/Closed Principle**
- Modules open for extension, closed for modification
- Example: `Atom` class can be subclassed for extended functionality

**3. Dependency Inversion**
- High-level modules don't depend on low-level details
- Example: `MapProcessor` depends on abstractions (function interfaces), not implementations

**4. Interface Segregation**
- Focused, specific interfaces
- Example: Separate functions for different search operations

**5. Don't Repeat Yourself (DRY)**
- Common operations abstracted into utilities
- Example: `clean_data()` used by multiple modules

## 2. Component Design Details

### 2.1 AutoMapper.py (CLI Controller)

**Design Pattern:** Command Pattern

```python
┌──────────────────────────────────────┐
│        AutoMapper.py                 │
├──────────────────────────────────────┤
│ + parse_arguments()                  │
│ + validate_arguments()               │
│ + execute_command()                  │
├──────────────────────────────────────┤
│ - tool: str                          │
│ - directory: str                     │
│ - args: argparse.Namespace           │
└──────────────────────────────────────┘
         │
         │ delegates to
         ├──> file_unifier()
         ├──> lammps_to_molecule()
         └──> map_processor()
```

**Key Design Decisions:**

1. **Argument Validation at CLI Level**
   - Rationale: Fail fast, provide immediate user feedback
   - Implementation: argparse with custom error messages

2. **Tool Selection via Command Pattern**
   - Rationale: Easy to add new tools without modifying core logic
   - Implementation: Dictionary-based dispatch or if-elif chain

3. **No Business Logic in CLI**
   - Rationale: Keep CLI thin, testable business logic
   - Implementation: All logic delegated to tool modules

**Control Flow:**
```
User Input → Parse Args → Validate Args → Select Tool → Execute Tool → Output
     ↓           ↓            ↓              ↓              ↓
   argv     Namespace    Check required  if/elif      Call function
                         arguments       dispatch      with args
```

### 2.2 Atom Class (Domain Model)

**Design Pattern:** Value Object with Behavior

```python
┌─────────────────────────────────────────────────────────────┐
│                    Atom                                      │
├─────────────────────────────────────────────────────────────┤
│ Attributes:                                                  │
│ - atomID: str                                                │
│ - atomType: str                                              │
│ - element: str                                               │
│ - bondingAtom: bool                                          │
│                                                              │
│ Neighbor Data (Immutable):                                  │
│ - firstNeighbourIDs: list[str]                              │
│ - secondNeighbourIDs: list[str]                             │
│ - thirdNeighbourIDs: list[str]                              │
│ - firstNeighbourElements: list[str]                         │
│ - secondNeighbourElements: list[str]                        │
│ - thirdNeighbourElements: list[str]                         │
│                                                              │
│ Neighbor Data (Mutable):                                    │
│ - mappedNeighbourIDs: list[str]                             │
│ - mappedNeighbourElements: list[str]                        │
├─────────────────────────────────────────────────────────────┤
│ Methods:                                                     │
│ + check_mapped(mappedIDs, searchIndex, elementDict)         │
│ + map_elements(atomObject, preDict, postDict)               │
│   └─> Returns: (mapList, missingPre, missingPost, queue)    │
└─────────────────────────────────────────────────────────────┘
```

**Key Design Decisions:**

1. **Separation of Mutable and Immutable State**
   - Rationale: Preserve original topology for fingerprinting
   - Implementation: Duplicate lists (mapped vs. original)

2. **Rich Domain Model**
   - Rationale: Encapsulate mapping logic within Atom
   - Implementation: `map_elements()` method handles neighbor mapping

3. **Multi-level Neighbor Storage**
   - Rationale: Support hierarchical fingerprinting
   - Implementation: Separate lists for 1st, 2nd, 3rd order neighbors

**State Transitions:**
```
Initial State:
    mappedNeighbourIDs = all neighbors
    ↓
After check_mapped():
    mappedNeighbourIDs = neighbors - already mapped atoms
    ↓
After map_elements():
    mappedNeighbourElements updated
    neighbor IDs consumed from mappedNeighbourIDs
```

**Invariants:**
1. `len(mappedNeighbourIDs) == len(mappedNeighbourElements)` always
2. `mappedNeighbourIDs ⊆ firstNeighbourIDs` always
3. `firstNeighbourIDs` never changes after construction

### 2.3 Queue Class (Data Structure)

**Design Pattern:** Adapter Pattern (wraps collections.deque)

```python
┌──────────────────────────────────────┐
│            Queue                     │
├──────────────────────────────────────┤
│ - elements: collections.deque        │
├──────────────────────────────────────┤
│ + empty() -> bool                    │
│ + add(x: list[list[Atom, Atom]])    │
│ + get() -> list[Atom, Atom]         │
└──────────────────────────────────────┘
```

**Key Design Decisions:**

1. **FIFO Queue Implementation**
   - Rationale: BFS requires FIFO ordering
   - Implementation: Wrap `collections.deque` for O(1) operations

2. **Batch Addition**
   - Rationale: Multiple atom pairs added simultaneously
   - Implementation: `add()` accepts list of pairs

3. **Simple Interface**
   - Rationale: Hide complexity of deque operations
   - Implementation: Three methods only (empty, add, get)

**Usage Pattern:**
```python
queue = Queue()
queue.add([[atom1_pre, atom1_post], [atom2_pre, atom2_post]])
while not queue.empty():
    current_pair = queue.get()
    # Process pair
    new_pairs = process(current_pair)
    queue.add(new_pairs)
```

### 2.4 PathSearch Module (Core Algorithm)

**Design Pattern:** Strategy Pattern (BFS Strategy)

```python
┌─────────────────────────────────────────────────────────────┐
│                 PathSearch Module                            │
├─────────────────────────────────────────────────────────────┤
│ + map_from_path(...)                                         │
│   ├─> Initialize data structures                            │
│   ├─> map_delete_atoms()                                    │
│   ├─> queue_bond_atoms()                                    │
│   ├─> run_queue()  [Main BFS Loop]                          │
│   ├─> map_missing_atoms()                                   │
│   └─> map_create_atoms()                                    │
│                                                              │
│ + map_delete_atoms(pre, post, mappedList)                   │
│ + map_missing_atoms(missingPre, missingPost, mappedList)    │
│ + map_create_atoms(create, mappedList)                      │
│ + get_missing_atom_objects(missingList, atomDict)           │
└─────────────────────────────────────────────────────────────┘
```

**Sequence Diagram for map_from_path:**

```
User → map_from_path : call with parameters
    map_from_path → element_atomID_dict : create element dicts
    map_from_path → build_atom_objects : create atom objects (pre & post)
    map_from_path → map_delete_atoms : add delete atoms to map
    map_from_path → queue_bond_atoms : initialize queue with bonding atoms
    map_from_path → run_queue : execute BFS mapping
        loop while queue not empty
            run_queue → Queue : get()
            run_queue → Atom : check_mapped()
            run_queue → Atom : map_elements()
            Atom → compare_symmetric_atoms : resolve ambiguity
            Atom → run_queue : return new maps, missing, queue atoms
            run_queue → Queue : add(queue_atoms)
        end
    map_from_path → map_missing_atoms : handle missing atoms
        loop 3 iterations max
            map_missing_atoms → compare_symmetric_atoms : match missing atoms
        end
    map_from_path → map_create_atoms : add create atoms to map
    map_from_path → User : return mappedIDList
```

**Key Design Decisions:**

1. **BFS Strategy**
   - Rationale: Level-order traversal ensures nearest neighbors mapped first
   - Implementation: Queue-based iteration

2. **Separate Handling of Special Atoms**
   - Rationale: Different logic for delete/create/missing atoms
   - Implementation: Dedicated functions for each type

3. **Iterative Missing Atom Resolution**
   - Rationale: May need multiple passes as more atoms are mapped
   - Implementation: While loop with counter limit (3 iterations)

4. **Stateless Functions**
   - Rationale: Easy testing, no hidden side effects
   - Implementation: All state in mappedIDList parameter

### 2.5 MapProcessor Module (Orchestrator)

**Design Pattern:** Facade Pattern

```python
┌─────────────────────────────────────────────────────────────┐
│                 MapProcessor Module                          │
├─────────────────────────────────────────────────────────────┤
│ + map_processor(...) [Main Orchestrator]                    │
│   ├─> lammps_to_molecule (pre)                             │
│   ├─> lammps_to_molecule (post)                            │
│   ├─> map_from_path                                        │
│   ├─> build_atom_objects                                   │
│   ├─> is_cyclic (pre & post)                               │
│   ├─> is_ring_opening                                      │
│   ├─> keep_all_neighbours                                  │
│   ├─> find_edge_atoms                                      │
│   ├─> verify_edge_atoms                                    │
│   ├─> extend_edge_atoms                                    │
│   ├─> get_byproducts                                       │
│   ├─> create_partial_map                                   │
│   ├─> lammps_to_molecule (partial pre)                    │
│   ├─> lammps_to_molecule (partial post)                   │
│   └─> output_map                                           │
│                                                              │
│ + is_cyclic(atomDict, bondingAtoms, reactionType)          │
│ + is_ring_opening(prePres, postPres, map)                  │
│ + keep_all_neighbours(atomDict, bondingAtoms, partialSet)  │
│ + find_edge_atoms(atomDict, partialSet)                    │
│ + verify_edge_atoms(edgeAtoms, map, preDict, postDict)     │
│ + extend_edge_atoms(extendDict, ...)                       │
│ + get_byproducts(atomDict, bondingAtoms)                   │
│ + create_partial_map(map, preSet, postSet)                 │
│ + output_map(map, bondingAtoms, edgeAtoms, ...)            │
│ + bfs(graph, startAtom, endAtom, breakLink)                │
└─────────────────────────────────────────────────────────────┘
```

**Control Flow Diagram:**

```
┌─────────────────────────┐
│ Initial Molecule Files  │
│ (full structures)       │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Initial Map Creation    │
│ (PathSearch BFS)        │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Build Atom Objects      │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Cycle Detection         │
│ (for each bonding atom) │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Ring Opening Check      │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Preserve Nearby Atoms   │
│ (4 bond distance)       │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Add Delete/Create Atoms │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Find Edge Atoms         │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Verify/Extend Edges     │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Add Byproducts          │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Create Partial Map      │
│ (renumber atoms)        │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Rebuild Molecule Files  │
│ (partial structures)    │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│ Output Map File         │
│ (automap.data)          │
└─────────────────────────┘
```

**Key Design Decisions:**

1. **Facade Pattern**
   - Rationale: Simplify complex workflow for callers
   - Implementation: Single entry point with many internal steps

2. **Step-by-Step Processing**
   - Rationale: Clear, debuggable workflow
   - Implementation: Sequential function calls with intermediate state

3. **Immutable Input, New Output**
   - Rationale: Preserve original data, create new files
   - Implementation: Never modify input files, always create new ones

4. **Context Manager for Directory Changes**
   - Rationale: Ensure return to original directory
   - Implementation: `restore_dir()` context manager

5. **Separate Concerns**
   - Rationale: Each function handles one aspect of partial structure
   - Implementation: Dedicated functions for cycles, edges, byproducts, etc.

### 2.6 LammpsSearchFuncs Module (Data Access)

**Design Pattern:** Repository Pattern

```python
┌─────────────────────────────────────────────────────────────┐
│              LammpsSearchFuncs Module                        │
├─────────────────────────────────────────────────────────────┤
│ File Parsing:                                                │
│ + find_sections(lines) -> list[int]                         │
│ + get_data(sectionName, lines, sections) -> list[list]      │
│ + get_coeff(coeffName, settingsData) -> list[list]          │
│                                                              │
│ Graph Operations:                                            │
│ + get_neighbours(atomIDs, bonds) -> dict[str, list[str]]    │
│ + get_additional_neighbours(neighbourDict, atomID, ...)     │
│                                                              │
│ Search Operations:                                           │
│ + pair_search(bond, bondAtom) -> str                        │
│ + search_loop(bonds, bondAtom) -> list[str]                 │
│                                                              │
│ Utilities:                                                   │
│ + element_atomID_dict(fileName, elementsByType) -> dict     │
│ + edge_atom_fingerprint_ids(edgeAtoms, bonds, validSet)     │
└─────────────────────────────────────────────────────────────┘
```

**Key Design Decisions:**

1. **Centralized Parsing Logic**
   - Rationale: Consistent file format handling
   - Implementation: Reusable parsing functions

2. **Flexible Section Retrieval**
   - Rationale: Different sections needed by different tools
   - Implementation: `get_data()` with section name parameter

3. **Graph View of Molecule**
   - Rationale: Enable topological operations
   - Implementation: Adjacency list representation via `get_neighbours()`

4. **Natural Sorting**
   - Rationale: Human-readable atom ID ordering (1, 2, 10 not 1, 10, 2)
   - Implementation: Use `natsort` library

### 2.7 LammpsTreatmentFuncs Module (Utilities)

**Design Pattern:** Utility Module (Static Methods)

```python
┌─────────────────────────────────────────────────────────────┐
│            LammpsTreatmentFuncs Module                       │
├─────────────────────────────────────────────────────────────┤
│ Data Cleaning:                                               │
│ + clean_data(lines) -> list[str]                            │
│   └─> Remove comments, blank lines, strip whitespace        │
│                                                              │
│ File I/O:                                                    │
│ + save_text_file(fileName, data) -> None                    │
│   └─> Write data with proper formatting                     │
│                                                              │
│ Formatting:                                                  │
│ + format_section(data, columnWidths) -> list[str]           │
│   └─> Align columns for readability                         │
└─────────────────────────────────────────────────────────────┘
```

**Key Design Decisions:**

1. **Stateless Utilities**
   - Rationale: Pure functions, easy to test
   - Implementation: No instance state, only parameters

2. **Consistent Data Cleaning**
   - Rationale: Uniform preprocessing across modules
   - Implementation: Single `clean_data()` function used everywhere

3. **Formatted Output**
   - Rationale: Human-readable output files
   - Implementation: Column alignment in `save_text_file()`

## 3. Data Flow Design

### 3.1 Clean Tool Data Flow

```
Input Files                Processing               Output Files
─────────────              ──────────              ──────────────
pre.data ────┐                                    ┌──> cleanedpre.data
              ├──> Load ──> Unify ──> Renumber ──┤
post.data ───┤     Files    Types     Atoms      └──> cleanedpost.data
              │                ↕                   ┌──> cleanedsettings.in
settings.in ─┘                                    └──> (optional)
                         Update Coeffs
                         (if provided)
```

**Data Transformations:**

1. **Type Collection:**
```
File1: {1: 'C', 2: 'H', 3: 'O'}
File2: {1: 'C', 2: 'N', 3: 'H'}
    ↓
Unified: {1: 'C', 2: 'H', 3: 'O', 4: 'N'}
```

2. **Type Mapping:**
```
File1 Type Mapping: {1→1, 2→2, 3→3}
File2 Type Mapping: {1→1, 2→4, 3→2}
```

3. **Atom Updates:**
```
File1 Atoms: [(1, type=1), (2, type=2), (3, type=3)]
File2 Atoms: [(1, type=1), (2, type=2), (3, type=3)]
    ↓ Apply Mapping
File1 Atoms: [(1, type=1), (2, type=2), (3, type=3)]  # No change
File2 Atoms: [(1, type=1), (2, type=4), (3, type=2)]  # Types changed
```

### 3.2 Molecule Tool Data Flow

```
Input                      Processing                   Output
──────                     ──────────                   ──────
data.data ──> Load ──> Parse ──> Filter ──> Renumber ──> molecule.data
              File     Sections   (optional)  (optional)
                          │
                          ├─> Atoms
                          ├─> Bonds
                          ├─> Angles
                          ├─> Dihedrals
                          └─> Calculate Special Bonds
```

**Data Transformations:**

1. **Section Extraction:**
```
Raw Data File:
    "Atoms\n1 1 1 0.0 0.0 0.0\n2 1 2 0.0 1.0 0.0\n..."
    ↓ Parse
Atoms Section:
    [['1', '1', '1', '0.0', '0.0', '0.0'],
     ['2', '1', '2', '0.0', '1.0', '0.0'], ...]
```

2. **Special Bond Calculation:**
```
Bonds: [(1,2), (2,3), (3,4), (4,5)]
For Atom 2:
    1-2: [1, 3]           (directly bonded)
    1-3: [4]              (through atom 3)
    1-4: [5]              (through atoms 3,4)
    → n12=2, n13=1, n14=1
```

3. **Molecule Format Output:**
```
Coords:
    1 0.0 0.0 0.0
    2 0.0 1.0 0.0
Types:
    1 1
    2 2
Special Bond Counts:
    1 2 1 1
    2 2 1 1
Special Bonds:
    1 1 3 4 5
    2 1 3 4 5
```

### 3.3 Map Tool Data Flow

```
Input                           Processing                        Output
─────────                       ──────────                        ──────
pre.data ────┐                                                   ┌─> pre-molecule.data
              ├─> Molecule ─┐                                   │
post.data ───┘   Creation    │                                  ├─> post-molecule.data
                              │                                   │
                              ├─> PathSearch ──> Mapping        └─> automap.data
                              │    (BFS)          Complete
                              │                       │
bonding atoms ───────────────┘                       │
elements by type ─────────────────────────────────────┤
delete atoms ─────────────────────────────────────────┤
create atoms ─────────────────────────────────────────┘
                                                       │
                                                       ↓
                                              Partial Structure
                                              Determination
                                                       │
                                                       ├─> Cycle Detection
                                                       ├─> Ring Opening Check
                                                       ├─> Neighbor Preservation
                                                       ├─> Edge Finding
                                                       ├─> Edge Verification
                                                       └─> Byproduct Addition
                                                       │
                                                       ↓
                                              Renumber & Rebuild
```

**Data Transformations:**

1. **Initial Mapping:**
```
Pre Atoms: {1:'C', 2:'H', 3:'H', 4:'O', 5:'H'}
Post Atoms: {1:'C', 2:'O', 3:'H', 4:'H'}
Bonding: [(1,1), (4,2)]
    ↓ BFS Mapping
Mapped: [(1,1), (4,2), (2,3), (3,4), (5,None-deleted)]
```

2. **Partial Structure Selection:**
```
Full Structure: {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, ...}
Bonding Atoms: {1, 4}
4-Bond Radius: {1, 2, 3, 4, 5, 6, 7}
Cycles: {1, 2, 3, 4, 8}
    ↓ Union
Partial Structure: {1, 2, 3, 4, 5, 6, 7, 8}
```

3. **Renumbering:**
```
Original IDs: [1, 3, 5, 7, 9]
    ↓ Natural Sort & Sequential Renumber
Renumber Map: {1→1, 3→2, 5→3, 7→4, 9→5}
New IDs: [1, 2, 3, 4, 5]
```

4. **Map Output:**
```
Bonding: [1, 4]
Edge: [7, 8]
Delete: []
Create: []
Equivalences:
    1    1
    2    3
    3    2
    4    4
    5    5
```

## 4. Algorithm Design Details

### 4.1 BFS Mapping Algorithm Design

**Algorithmic Paradigm:** Breadth-First Search with Constraint Satisfaction

**Pseudocode:**
```
procedure BFS_MAPPING(pre_graph, post_graph, bonding_atoms):
    initialize queue Q
    initialize mapping M = {}
    
    # Seed with bonding atoms
    for each (pre_bond, post_bond) in bonding_atoms:
        M[pre_bond] = post_bond
        Q.enqueue((pre_bond, post_bond))
    
    # BFS traversal
    while Q not empty:
        (pre_atom, post_atom) = Q.dequeue()
        
        # Get unmapped neighbors
        pre_neighbors = neighbors(pre_atom) \ mapped_atoms(M)
        post_neighbors = neighbors(post_atom) \ mapped_atoms(M)
        
        # Match neighbors
        for each pre_neighbor in pre_neighbors:
            post_match = find_correspondence(
                pre_neighbor, 
                post_neighbors, 
                M, 
                pre_graph, 
                post_graph
            )
            
            if post_match found:
                M[pre_neighbor] = post_match
                if element(pre_neighbor) != 'H':
                    Q.enqueue((pre_neighbor, post_match))
            else:
                add pre_neighbor to missing_list
    
    # Handle missing atoms
    resolve_missing_atoms(missing_list, M, post_graph)
    
    return M
end procedure
```

**Design Rationale:**

1. **Why BFS?**
   - Ensures nearest neighbors mapped before distant ones
   - Provides level-order traversal
   - Natural for molecular topology (connectivity)

2. **Why Skip Hydrogen in Queue?**
   - Hydrogen has no downstream neighbors typically
   - Reduces queue size significantly
   - Mapping determined by heavier atom it's bonded to

3. **Why Separate Missing Atom Handling?**
   - Initial mapping may not resolve all atoms
   - Iterative refinement needed as more context available
   - Cleaner separation of concerns

### 4.2 Fingerprint Matching Design

**Algorithmic Paradigm:** Hierarchical String Matching

**Data Structure:**
```python
Fingerprint = {
    'level1': sorted_concatenation(neighbor_elements),
    'level2': sorted_concatenation(second_neighbor_elements),
    'level3': sorted_concatenation(third_neighbor_elements)
}

Example:
    Atom with neighbors [C, H, H, O]
    Fingerprint Level 1: "CHHO"
    
    With second neighbors [C, C, H, H, H, N]
    Fingerprint Level 2: "CCHHHN"
```

**Matching Algorithm:**
```
procedure MATCH_BY_FINGERPRINT(candidates, target_atom, level):
    target_fp = fingerprint(target_atom, level)
    
    # Build fingerprint map
    fp_map = {}
    for each candidate in candidates:
        cand_fp = fingerprint(candidate, level)
        if cand_fp not in fp_map:
            fp_map[cand_fp] = []
        fp_map[cand_fp].append(candidate)
    
    # Check uniqueness
    if target_fp in fp_map and len(fp_map[target_fp]) == 1:
        return fp_map[target_fp][0]  # Unique match
    else:
        return None  # Non-unique or not found
end procedure

procedure HIERARCHICAL_MATCH(candidates, target_atom):
    for level in [1, 2, 3]:
        match = MATCH_BY_FINGERPRINT(candidates, target_atom, level)
        if match != None:
            return match
    
    # Fallback to element-based inference
    return INFER_BY_ELEMENT(candidates, target_atom)
end procedure
```

**Design Rationale:**

1. **Why Hierarchical?**
   - Start with simplest (fastest) matching
   - Progress to more complex if needed
   - Balance speed vs. accuracy

2. **Why String Concatenation?**
   - Simple to compute
   - Easy to compare
   - Sorting ensures canonical representation

3. **Why Check Uniqueness?**
   - Prevent incorrect matches in symmetric molecules
   - Only assign if unambiguous
   - Defer ambiguous cases to higher levels

### 4.3 Cycle Detection Design

**Algorithmic Paradigm:** Modified BFS Path Finding

**Graph Representation:**
```python
Graph = {
    atom_id: [neighbor1_id, neighbor2_id, ...]
    for all atoms
}

Example:
    {
        '1': ['2', '6'],
        '2': ['1', '3'],
        '3': ['2', '4'],
        '4': ['3', '5'],
        '5': ['4', '6'],
        '6': ['5', '1']
    }
    # Forms a 6-membered ring
```

**Algorithm:**
```
procedure IS_CYCLIC(graph, bonding_atom):
    for each neighbor of bonding_atom:
        # Remove direct edge to prevent backtracking
        modified_graph = graph.copy()
        modified_graph[bonding_atom].remove(neighbor)
        
        # Search for path from neighbor back to bonding_atom
        path = BFS_PATH(modified_graph, neighbor, bonding_atom)
        
        if path exists:
            # Cycle found
            cycle_atoms = set(path) ∪ {bonding_atom}
            preserved = cycle_atoms ∪ all_neighbors_of(cycle_atoms)
            return preserved
    
    return None  # No cycle
end procedure

procedure BFS_PATH(graph, start, target):
    queue = [[start]]
    visited = {start}
    
    while queue not empty:
        path = queue.dequeue()
        node = path[-1]
        
        if node == target:
            return path
        
        for each neighbor of node:
            if neighbor not in visited:
                visited.add(neighbor)
                new_path = path + [neighbor]
                queue.enqueue(new_path)
    
    return None  # No path found
end procedure
```

**Design Rationale:**

1. **Why Remove Direct Edge?**
   - Prevent trivial path: bonding_atom → neighbor → bonding_atom
   - Force search through alternative routes
   - Confirms true cyclic structure

2. **Why BFS for Path Finding?**
   - Finds shortest path (most relevant cycle)
   - Efficient for molecular graphs
   - Natural for graph traversal

3. **Why Preserve Cycle + Neighbors?**
   - Cycle atoms critical for reaction
   - Neighbors provide context for LAMMPS
   - Ensures stable partial structure

### 4.4 Partial Structure Minimization Design

**Algorithmic Paradigm:** Set Operations with Distance Constraints

**Distance-Based Selection:**
```
procedure KEEP_NEIGHBORS_WITHIN_DISTANCE(graph, bonding_atoms, max_dist):
    preserved = {}
    
    for each bonding_atom in bonding_atoms:
        current_level = {bonding_atom}
        visited = {bonding_atom}
        
        for dist from 1 to max_dist:
            next_level = {}
            for each atom in current_level:
                for each neighbor of atom:
                    if neighbor not in visited:
                        next_level.add(neighbor)
                        visited.add(neighbor)
            current_level = next_level
        
        preserved = preserved ∪ visited
    
    return preserved
end procedure
```

**Set Operations:**
```
Final Partial Structure = 
    Distance_Based_Set 
    ∪ Cycle_Preserved_Set 
    ∪ Delete_Atoms_Set 
    ∪ Create_Atoms_Set 
    ∪ Byproduct_Set
```

**Design Rationale:**

1. **Why Distance Threshold = 4?**
   - Balances completeness vs. minimality
   - Captures sufficient chemical context
   - Empirically determined optimal value

2. **Why Union of Sets?**
   - Different requirements contribute atoms
   - Ensures all critical atoms included
   - Prevents accidental omissions

3. **Why Check Edge Atoms?**
   - Verify partial structure boundary integrity
   - Prevent type changes at boundaries
   - Extend if necessary for consistency

## 5. Error Handling and Robustness

### 5.1 Error Detection Strategies

**Input Validation:**
```python
class InputValidator:
    @staticmethod
    def validate_bonding_atoms(bonding_atoms, all_atoms):
        if len(bonding_atoms) < 2:
            raise ValueError("At least 2 bonding atoms required")
        for atom in bonding_atoms:
            if atom not in all_atoms:
                raise ValueError(f"Bonding atom {atom} not in atom list")
    
    @staticmethod
    def validate_element_list(elements, num_types):
        if len(elements) != num_types:
            raise ValueError(
                f"Element list length {len(elements)} doesn't match "
                f"atom types {num_types}"
            )
    
    @staticmethod
    def validate_delete_atoms(delete_atoms):
        if len(delete_atoms) % 2 != 0:
            raise ValueError("Delete atoms must have even length (pre+post)")
```

**Runtime Error Detection:**
```python
def detect_mapping_failure(mapped_count, expected_count, threshold=0.9):
    """Detect if mapping failed to capture most atoms."""
    ratio = mapped_count / expected_count
    if ratio < threshold:
        logging.warning(
            f"Mapping only captured {ratio*100:.1f}% of atoms. "
            f"Expected > {threshold*100:.1f}%"
        )
        return True
    return False

def detect_element_mismatch(pre_atom, post_atom):
    """Detect element conservation violation."""
    if pre_atom.element != post_atom.element:
        raise ValueError(
            f"Element mismatch: pre atom {pre_atom.atomID} is "
            f"{pre_atom.element}, post atom {post_atom.atomID} is "
            f"{post_atom.element}"
        )
```

### 5.2 Recovery Strategies

**Graceful Degradation:**
```python
def map_with_fallback(pre_atom, post_candidates):
    """Try multiple strategies in order of preference."""
    
    # Strategy 1: Unique fingerprint match
    match = match_by_fingerprint(pre_atom, post_candidates, unique_only=True)
    if match:
        return match
    
    # Strategy 2: Element-based inference with warning
    match = match_by_element(pre_atom, post_candidates)
    if match:
        logging.warning(f"Using inference for atom {pre_atom.atomID}")
        return match
    
    # Strategy 3: Add to missing list for later resolution
    logging.info(f"Deferring atom {pre_atom.atomID} to missing list")
    return None
```

**Iterative Refinement:**
```python
def resolve_missing_atoms(missing_pre, missing_post, mapped_list, max_iterations=3):
    """Iteratively resolve missing atoms as more context becomes available."""
    
    for iteration in range(max_iterations):
        if not missing_post:
            break  # All resolved
        
        resolved_this_iteration = []
        for pre_atom in missing_pre:
            match = try_match_missing(pre_atom, missing_post, mapped_list)
            if match:
                resolved_this_iteration.append((pre_atom, match))
        
        if not resolved_this_iteration:
            break  # No progress, stop iterating
        
        # Update lists
        for pre, post in resolved_this_iteration:
            mapped_list.append([pre, post])
            missing_pre.remove(pre)
            missing_post.remove(post)
    
    if missing_post:
        logging.warning(f"Could not resolve {len(missing_post)} atoms")
```

### 5.3 Logging and Debugging

**Logging Levels:**
```python
# DEBUG: Detailed algorithm steps
logging.debug(f"Mapping pre atom {pre_id} to post atom {post_id} "
              f"via fingerprint level {level}")

# INFO: Major milestones
logging.info(f"Created {len(mapped_list)} equivalences")

# WARNING: Potential issues
logging.warning(f"Using inference for symmetric atom {atom_id}")

# ERROR: Critical failures
logging.error(f"Cannot find bonding atom {atom_id} in molecule")
```

**Debug Mode Features:**
```python
if debug:
    # Print intermediate states
    print(f"Queue state: {len(queue.elements)} pairs remaining")
    print(f"Mapped atoms: {len(mapped_list)}")
    print(f"Missing atoms: {len(missing_pre)} pre, {len(missing_post)} post")
    
    # Validate invariants
    assert len(mapped_list) <= len(pre_atoms)
    assert len(set([m[0] for m in mapped_list])) == len(mapped_list)  # Unique
```

## 6. Performance Optimization

### 6.1 Algorithmic Optimizations

**Early Termination:**
```python
def find_unique_match(candidates, target, fingerprint_func):
    """Stop as soon as unique match found."""
    fingerprints = [fingerprint_func(c) for c in candidates]
    target_fp = fingerprint_func(target)
    
    matches = [i for i, fp in enumerate(fingerprints) if fp == target_fp]
    
    if len(matches) == 1:
        return candidates[matches[0]]  # Unique match, return immediately
    else:
        return None  # Continue to next level
```

**Caching:**
```python
class AtomWithCache:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._fingerprint_cache = {}
    
    def get_fingerprint(self, level):
        """Cache fingerprints to avoid recomputation."""
        if level not in self._fingerprint_cache:
            self._fingerprint_cache[level] = self._compute_fingerprint(level)
        return self._fingerprint_cache[level]
```

**Batch Processing:**
```python
def update_all_mapped_neighbors(atoms, mapped_list, element_dict):
    """Update multiple atoms in one pass."""
    mapped_ids = set(m[0] for m in mapped_list)
    
    for atom in atoms:
        atom.mappedNeighbourIDs = [
            n for n in atom.firstNeighbourIDs 
            if n not in mapped_ids
        ]
        atom.mappedNeighbourElements = [
            element_dict[n] for n in atom.mappedNeighbourIDs
        ]
```

### 6.2 Data Structure Optimizations

**Set-Based Membership:**
```python
# Slow: O(n) for each lookup
if atom_id in mapped_id_list:  # list lookup
    ...

# Fast: O(1) for each lookup
mapped_id_set = set(mapped_id_list)
if atom_id in mapped_id_set:  # set lookup
    ...
```

**Dictionary Lookups:**
```python
# Organize data for O(1) access
atom_dict = {atom.atomID: atom for atom in atoms}
element_dict = {atom.atomID: atom.element for atom in atoms}

# Fast access
atom = atom_dict[atom_id]
element = element_dict[atom_id]
```

### 6.3 Memory Optimizations

**Lazy Evaluation:**
```python
def get_third_neighbors(atom, neighbor_dict, bonding_atoms):
    """Compute only when needed, not at construction."""
    if not hasattr(atom, '_third_neighbors'):
        atom._third_neighbors = compute_third_neighbors(
            atom, neighbor_dict, bonding_atoms
        )
    return atom._third_neighbors
```

**Generator Expressions:**
```python
# Memory efficient for large lists
valid_atoms = (atom for atom in all_atoms if atom.atomID in valid_set)

# Process without materializing full list
for atom in valid_atoms:
    process(atom)
```

## 7. Testing Strategy

### 7.1 Unit Testing

**Test Atom Class:**
```python
def test_atom_check_mapped():
    atom = Atom('1', '1', 'C', False, ['2', '3'], [], [], ['H', 'O'], [], [])
    mapped_list = [['2', '2'], ['4', '4']]
    element_dict = {'1': 'C', '3': 'O'}
    
    atom.check_mapped(mapped_list, 0, element_dict)
    
    assert atom.mappedNeighbourIDs == ['3']
    assert atom.mappedNeighbourElements == ['O']

def test_atom_map_elements():
    pre_atom = create_test_atom('1', neighbors=['2', '3'])
    post_atom = create_test_atom('1', neighbors=['2', '3'])
    
    map_list, missing_pre, missing_post, queue = pre_atom.map_elements(
        post_atom, pre_dict, post_dict
    )
    
    assert len(map_list) == 2
    assert missing_pre == []
    assert missing_post == []
```

**Test Queue:**
```python
def test_queue_fifo():
    q = Queue()
    q.add([['1', '1'], ['2', '2']])
    
    first = q.get()
    assert first == ['1', '1']
    
    second = q.get()
    assert second == ['2', '2']
    
    assert q.empty()
```

**Test Search Functions:**
```python
def test_get_neighbours():
    bonds = [['1', '1', '1', '2'], ['2', '1', '2', '3']]
    atom_ids = ['1', '2', '3']
    
    neighbors = get_neighbours(atom_ids, bonds)
    
    assert neighbors['1'] == ['2']
    assert neighbors['2'] == ['1', '3']
    assert neighbors['3'] == ['2']
```

### 7.2 Integration Testing

**Test Complete Mapping:**
```python
def test_map_simple_reaction():
    # Setup test data
    pre_file = create_test_lammps_file('pre.data')
    post_file = create_test_lammps_file('post.data')
    
    # Run mapping
    mapped_list = map_from_path(
        '.', 'pre.data', 'post.data', 
        ['H', 'C', 'O'], False,
        ['1', '2'], None, ['1', '2'], None, None
    )
    
    # Verify results
    assert len(mapped_list) > 0
    assert all(len(pair) == 2 for pair in mapped_list)
    verify_element_conservation(mapped_list, pre_file, post_file)
```

**Test Cycle Detection:**
```python
def test_cyclic_molecule():
    # Create 6-membered ring
    atom_dict = create_cyclic_atoms(6)
    bonding_atoms = ['1', '4']  # Opposite sides of ring
    
    preserved = is_cyclic(atom_dict, bonding_atoms, 'Pre-bond')
    
    assert preserved['1'] is not None
    assert len(preserved['1']) >= 6  # At least ring atoms
```

### 7.3 System Testing

**Test Complete Workflow:**
```python
def test_full_map_workflow():
    # Create test files
    setup_test_files()
    
    # Run complete map tool
    result = map_processor(
        '.', 'pre.data', 'post.data',
        'pre.mol', 'post.mol',
        ['1', '2'], ['1', '2'],
        None, ['H', 'C', 'O'], None, False
    )
    
    # Verify all outputs created
    assert os.path.exists('pre.mol')
    assert os.path.exists('post.mol')
    assert os.path.exists('automap.data')
    
    # Verify output format
    verify_molecule_format('pre.mol')
    verify_map_format('automap.data')
```

## 8. Maintenance and Evolution

### 8.1 Code Organization Principles

**Module Cohesion:**
- Each module has single, clear responsibility
- Related functions grouped together
- Minimal cross-module dependencies

**Function Length:**
- Target: <50 lines per function
- Break complex functions into subfunctions
- Each function does one thing well

**Naming Conventions:**
```python
# Files: lowercase_with_underscores
# Classes: PascalCase
# Functions: lowercase_with_underscores
# Constants: UPPERCASE_WITH_UNDERSCORES
# Private: _leading_underscore
```

### 8.2 Documentation Standards

**Module Docstrings:**
```python
"""
Module: MapProcessor

Purpose: Orchestrate creation of molecule files and equivalence maps
         for LAMMPS bond/react simulations.

Key Functions:
    - map_processor: Main entry point
    - is_cyclic: Detect cyclic structures
    - create_partial_map: Generate minimal structure

Author: Matthew Bone
Date: 30/07/2021
"""
```

**Function Docstrings:**
```python
def map_processor(...):
    """
    Create molecule files and equivalence map for bond/react.
    
    This function orchestrates the complete workflow:
    1. Initial molecule file creation
    2. BFS-based atom mapping
    3. Partial structure determination
    4. File renumbering and output
    
    Args:
        directory (str): Working directory path
        preDataFileName (str): Pre-reaction data file
        ...
    
    Returns:
        list: [mappedIDList, partialMappedIDList]
        
    Raises:
        ValueError: If bonding atoms invalid
        FileNotFoundError: If data files missing
        
    Example:
        >>> result = map_processor('.', 'pre.data', 'post.data', ...)
        >>> print(f"Mapped {len(result[0])} atoms")
    """
```

### 8.3 Extension Guidelines

**Adding New Tools:**
```python
# 1. Create new module
# NewTool.py
def new_tool_function(directory, input_file, options):
    """Implement new tool logic."""
    pass

# 2. Add to AutoMapper.py
from NewTool import new_tool_function

parser.add_argument('tool', choices=['clean', 'molecule', 'map', 'newtool'])

if tool == 'newtool':
    new_tool_function(directory, args.input_file, args.options)
```

**Adding Atom Properties:**
```python
# 1. Extend Atom class
class ExtendedAtom(Atom):
    def __init__(self, *args, stereochemistry=None, **kwargs):
        super().__init__(*args, **kwargs)
        self.stereochemistry = stereochemistry

# 2. Update build_atom_objects
def build_extended_atom_objects(fileName, elementDict, ...):
    # ... existing code ...
    stereochem = calculate_stereochemistry(coordinates, neighbors)
    atom = ExtendedAtom(..., stereochemistry=stereochem)
```

**Adding Matching Strategies:**
```python
# 1. Implement new strategy
def match_by_3d_distance(candidates, target_atom, coordinates):
    """Match atoms by 3D spatial proximity."""
    target_coords = coordinates[target_atom.atomID]
    min_dist = float('inf')
    best_match = None
    
    for candidate in candidates:
        cand_coords = coordinates[candidate.atomID]
        dist = euclidean_distance(target_coords, cand_coords)
        if dist < min_dist:
            min_dist = dist
            best_match = candidate
    
    return best_match

# 2. Integrate into compare_symmetric_atoms
def compare_symmetric_atoms_extended(..., coordinates=None):
    # ... existing fingerprint matching ...
    
    if match is None and coordinates is not None:
        match = match_by_3d_distance(candidates, target, coordinates)
    
    return match
```

## 9. Future Design Considerations

### 9.1 Parallelization Opportunities

**Independent Mappings:**
```python
# Multiple bonding atom pairs can be mapped in parallel
def parallel_map_processor(pre_data, post_data, bonding_pairs_list):
    with multiprocessing.Pool() as pool:
        results = pool.starmap(
            map_single_bonding_pair,
            [(pre_data, post_data, pairs) for pairs in bonding_pairs_list]
        )
    return merge_results(results)
```

**Batch File Processing:**
```python
# Process multiple reactions simultaneously
def batch_process_reactions(reaction_list):
    with concurrent.futures.ProcessPoolExecutor() as executor:
        futures = [
            executor.submit(map_processor, **reaction_params)
            for reaction_params in reaction_list
        ]
        results = [f.result() for f in futures]
    return results
```

### 9.2 Machine Learning Integration

**Learned Fingerprints:**
```python
class LearnedAtomEmbedding:
    def __init__(self, model_path):
        self.model = load_trained_model(model_path)
    
    def embed(self, atom, graph):
        """Generate learned representation of atom."""
        features = extract_features(atom, graph)
        embedding = self.model.predict(features)
        return embedding
    
    def similarity(self, atom1, atom2):
        """Compute similarity between atoms."""
        emb1 = self.embed(atom1)
        emb2 = self.embed(atom2)
        return cosine_similarity(emb1, emb2)
```

**Confidence Scoring:**
```python
def map_with_confidence(pre_atom, post_candidates, model):
    """Return mapping with confidence score."""
    confidences = [
        model.predict_match_probability(pre_atom, candidate)
        for candidate in post_candidates
    ]
    
    best_idx = np.argmax(confidences)
    return post_candidates[best_idx], confidences[best_idx]
```

### 9.3 Visualization Integration

**Molecular Structure Viewer:**
```python
class MoleculeVisualizer:
    def visualize_mapping(self, pre_atoms, post_atoms, mapped_list):
        """Interactive 3D visualization of atom mapping."""
        fig = create_3d_plot()
        
        # Draw pre-reaction structure
        draw_atoms(fig, pre_atoms, color='blue', subplot=1)
        draw_bonds(fig, pre_atoms)
        
        # Draw post-reaction structure
        draw_atoms(fig, post_atoms, color='red', subplot=2)
        draw_bonds(fig, post_atoms)
        
        # Draw mapping arrows
        for pre_id, post_id in mapped_list:
            draw_arrow(fig, pre_atoms[pre_id], post_atoms[post_id])
        
        return fig
```

**Debugging Visualizations:**
```python
def visualize_bfs_progress(queue_history, mapped_history):
    """Animate BFS algorithm progress."""
    for step, (queue_state, mapped_state) in enumerate(zip(queue_history, mapped_history)):
        fig = create_plot()
        
        # Show current queue contents
        draw_queue(fig, queue_state)
        
        # Highlight mapped atoms
        draw_mapped_atoms(fig, mapped_state)
        
        # Show current step number
        fig.title(f"BFS Step {step}")
        
        save_frame(fig, f"bfs_step_{step}.png")
    
    create_animation("bfs_animation.gif")
```

## 10. Conclusion

AutoMapper's design emphasizes:

1. **Modularity**: Clear separation of concerns enables independent development and testing
2. **Extensibility**: Well-defined interfaces allow for future enhancements
3. **Robustness**: Error handling and validation at multiple levels
4. **Performance**: Algorithmic optimizations for typical molecular structures
5. **Maintainability**: Clean code organization and comprehensive documentation

The design choices balance theoretical correctness with practical implementation concerns, resulting in a tool that is both mathematically sound and practically useful for LAMMPS users.
