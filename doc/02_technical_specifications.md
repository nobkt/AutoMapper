# AutoMapper: Detailed Technical Specifications

## 1. System Overview

### 1.1 System Architecture

AutoMapper is a command-line Python application with a modular architecture consisting of:

```
┌─────────────────────────────────────────────┐
│         AutoMapper.py (CLI Interface)       │
└───────────────┬─────────────────────────────┘
                │
                ├──> LammpsUnifiedCleaner (clean tool)
                │    └──> LammpsTreatmentFuncs
                │    └──> LammpsSearchFuncs
                │
                ├──> LammpsToMolecule (molecule tool)
                │    └──> LammpsSearchFuncs
                │    └──> LammpsTreatmentFuncs
                │
                └──> MapProcessor (map tool)
                     ├──> PathSearch
                     │    ├──> AtomObjectBuilder
                     │    ├──> QueueFuncs
                     │    └──> LammpsSearchFuncs
                     ├──> AtomObjectBuilder
                     ├──> LammpsToMolecule
                     └──> LammpsTreatmentFuncs
```

### 1.2 Core Components

**Component 1: AutoMapper.py**
- Role: Command-line interface and argument parser
- Input: Command-line arguments
- Output: Calls to appropriate tool modules
- Dependencies: argparse, sys, natsort

**Component 2: LammpsUnifiedCleaner**
- Role: Unify atom types across multiple LAMMPS data files
- Input: Multiple LAMMPS data files, coefficients file
- Output: Cleaned and unified data files
- Dependencies: LammpsTreatmentFuncs, LammpsSearchFuncs

**Component 3: LammpsToMolecule**
- Role: Convert LAMMPS data files to molecule format
- Input: LAMMPS data file, bonding atoms, optional parameters
- Output: LAMMPS molecule format file
- Dependencies: LammpsSearchFuncs, LammpsTreatmentFuncs

**Component 4: MapProcessor**
- Role: Create molecule files and equivalence map
- Input: Pre/post data files, bonding atoms, elements
- Output: Pre/post molecule files, automap.data
- Dependencies: PathSearch, AtomObjectBuilder, LammpsToMolecule

**Component 5: PathSearch**
- Role: Perform BFS-based atom mapping
- Input: Molecule files, element types, bonding atoms
- Output: Equivalence map (list of atom ID pairs)
- Dependencies: AtomObjectBuilder, QueueFuncs, LammpsSearchFuncs

**Component 6: AtomObjectBuilder**
- Role: Create and manipulate Atom objects
- Input: Molecule file, element dictionary
- Output: Dictionary of Atom objects
- Dependencies: LammpsSearchFuncs, LammpsTreatmentFuncs

**Component 7: QueueFuncs**
- Role: Queue management for BFS algorithm
- Input: Atom pairs to process
- Output: Ordered queue of atom pairs
- Dependencies: collections.deque

**Component 8: LammpsSearchFuncs**
- Role: Parse and search LAMMPS files
- Input: LAMMPS file data
- Output: Extracted data sections
- Dependencies: natsort

**Component 9: LammpsTreatmentFuncs**
- Role: File I/O and data cleaning utilities
- Input: File data
- Output: Cleaned data or saved files
- Dependencies: natsort

## 2. Data Structures and Formats

### 2.1 Atom Object Specification

```python
class Atom:
    """
    Represents a single atom with topological information.
    
    Attributes:
        atomID (str): Unique identifier for the atom
        atomType (str): Force field type identifier
        element (str): Chemical element symbol (H, C, N, O, etc.)
        bondingAtom (bool): True if atom participates in bond formation
        
        # Neighbor IDs (mutable during mapping)
        mappedNeighbourIDs (list[str]): Current unmapped neighbor IDs
        
        # Neighbor IDs (immutable reference)
        firstNeighbourIDs (list[str]): Original 1st order neighbor IDs
        secondNeighbourIDs (list[str]): Original 2nd order neighbor IDs
        thirdNeighbourIDs (list[str]): Original 3rd order neighbor IDs
        
        # Element lists (mutable during mapping)
        mappedNeighbourElements (list[str]): Elements of unmapped neighbors
        
        # Element lists (immutable reference)
        firstNeighbourElements (list[str]): Elements of 1st order neighbors
        secondNeighbourElements (list[str]): Elements of 2nd order neighbors
        thirdNeighbourElements (list[str]): Elements of 3rd order neighbors
    """
```

**Invariants:**
1. `len(firstNeighbourIDs) == len(firstNeighbourElements)`
2. `mappedNeighbourIDs ⊆ firstNeighbourIDs`
3. `bondingAtom` atoms must have at least one neighbor

**Methods:**
- `check_mapped(mappedIDs, searchIndex, elementDict)`: Update mapped neighbors
- `map_elements(atomObject, preAtomObjectDict, postAtomObjectDict)`: Map neighbors

### 2.2 Queue Data Structure

```python
class Queue:
    """
    FIFO queue for BFS atom pair processing.
    
    Attributes:
        elements (collections.deque): Double-ended queue of atom pairs
    """
```

**Operations:**
- `empty() -> bool`: Check if queue is empty, O(1)
- `add(x: list[list[Atom, Atom]])`: Add atom pairs to queue, O(k) where k = len(x)
- `get() -> list[Atom, Atom]`: Remove and return front pair, O(1)

### 2.3 File Format Specifications

#### 2.3.1 LAMMPS Data File Format (Input)

```
# Comment line
N atoms
M bonds
P angles
Q dihedrals

T atom types
B bond types
A angle types
D dihedral types

xlo xhi
ylo yhi
zlo zhi

Masses

1 mass1
2 mass2
...

Atoms  # full atom style

atomID moleculeID atomType charge x y z
...

Bonds

bondID bondType atomID1 atomID2
...

Angles

angleID angleType atomID1 atomID2 atomID3
...

Dihedrals

dihedralID dihedralType atomID1 atomID2 atomID3 atomID4
...
```

**Parsing Rules:**
1. Section headers are single alphabetic words
2. Data lines contain space-separated values
3. Comments start with '#'
4. Blank lines separate sections

#### 2.3.2 LAMMPS Molecule File Format (Output)

```
# Comment line

N atoms
M bonds
P angles
Q dihedrals

Coords

atomID x y z
...

Types

atomID atomType
...

Charges

atomID charge
...

Bonds

bondID bondType atomID1 atomID2
...

Angles

angleID angleType atomID1 atomID2 atomID3
...

Dihedrals

dihedralID dihedralType atomID1 atomID2 atomID3 atomID4
...

Special Bond Counts

atomID n12 n13 n14
...

Special Bonds

atomID specialBond1 specialBond2 ... specialBondn
...
```

**Key Differences from Data Files:**
1. No box dimensions
2. Separate Coords section for coordinates
3. Types section for atom types
4. Special Bond Counts and Special Bonds sections required

#### 2.3.3 Map File Format (automap.data)

```
#This is an AutoMapper generated map
N equivalences
D deleteIDs
E edgeIDs
C createIDs

BondingIDs

bondingID1
bondingID2
...

DeleteIDs

deleteID1
deleteID2
...

EdgeIDs

edgeID1
edgeID2
...

CreateIDs

createID1
createID2
...

Equivalences

preID1    postID1
preID2    postID2
...
```

**Format Specifications:**
- N, D, E, C are integer counts
- IDs are space or tab separated in equivalences
- All sections optional except BondingIDs and Equivalences

### 2.4 Internal Data Representations

#### 2.4.1 Mapped ID List

```python
mappedIDList: list[list[str, str]]
# Example: [['1', '1'], ['2', '3'], ['3', '2'], ...]
# Each inner list: [preAtomID, postAtomID]
```

**Properties:**
- Ordered by preAtomID (using natural sort)
- Bijective mapping (each ID appears at most once)
- Element consistency guaranteed

#### 2.4.2 Atom Object Dictionary

```python
atomObjectDict: dict[str, Atom]
# Key: atomID (string)
# Value: Atom object
# Example: {'1': Atom(...), '2': Atom(...), ...}
```

**Properties:**
- Keys match atom IDs in molecule file
- Complete coverage of all non-create atoms
- Consistent neighbor references

#### 2.4.3 Element Dictionary

```python
elementDict: dict[str, str]
# Key: atomID (string)
# Value: element symbol (string)
# Example: {'1': 'C', '2': 'H', '3': 'O', ...}
```

**Construction:**
```python
elementDict = {atomID: elementsByType[int(atomType) - 1] 
               for atomID, atomType in zip(atomIDs, atomTypes)}
```

## 3. Algorithm Specifications

### 3.1 Clean Tool Algorithm

**Function:** `file_unifier(directory, coeffFile, dataFiles)`

**Algorithm:**
```
1. Load all data files
2. Extract atom types from each file
3. Create unified type mapping:
   a. Collect all unique types across files
   b. Map each (element, properties) combination to unified type
4. Update atom types in all files using unified mapping
5. Update coefficient file:
   a. Remove unused coefficients
   b. Renumber coefficients to match unified types
6. Save cleaned files with 'cleaned' prefix
```

**Type Unification Formula:**
```
For each file i:
    typeMap_i: oldType -> unifiedType
    
Constraint: If type T1 in file1 and T2 in file2 have same element:
    unifiedType(T1) = unifiedType(T2)
```

**Complexity:** O(n·m) where n = atoms, m = files

### 3.2 Molecule Tool Algorithm

**Function:** `lammps_to_molecule(directory, fileName, saveName, bondingAtoms, deleteAtoms, validIDSet, renumberedAtomDict)`

**Algorithm:**
```
1. Load and parse LAMMPS data file
2. Extract sections: Atoms, Bonds, Angles, Dihedrals, Masses
3. If validIDSet provided:
   a. Filter all sections to include only validIDSet atoms
   b. Renumber using renumberedAtomDict
4. Calculate special bond counts:
   For each atom:
       n12 = number of 1-2 (bonded) neighbors
       n13 = number of 1-3 (angle) neighbors  
       n14 = number of 1-4 (dihedral) neighbors
5. Format output in molecule file format
6. Save to file
```

**Special Bond Calculation:**
```
For atom i:
    neighbors_12 = {j : (i,j) in Bonds}
    neighbors_13 = {k : exists j, (i,j) in Bonds AND (j,k) in Bonds, k != i}
    neighbors_14 = {l : exists j,k, (i,j), (j,k), (k,l) in path, l not in 12 or 13}
    
    n12 = |neighbors_12|
    n13 = |neighbors_13|
    n14 = |neighbors_14|
```

**Complexity:** O(n + b + a + d) where n=atoms, b=bonds, a=angles, d=dihedrals

### 3.3 Map Tool Algorithm (Complete)

**Function:** `map_processor(...)`

**High-Level Algorithm:**
```
1. Initial molecule file creation
   - Call lammps_to_molecule for pre-reaction
   - Call lammps_to_molecule for post-reaction

2. Initial map creation
   - Call map_from_path (PathSearch algorithm)
   - Returns mappedIDList

3. Partial structure determination
   a. Build atom object dictionaries
   b. Detect cycles for each bonding atom (is_cyclic)
   c. Check for ring opening (is_ring_opening)
   d. Preserve atoms within 4 bonds of bonding atoms (keep_all_neighbours)
   e. Add delete atoms to preserved set
   f. Add create atoms to post preserved set
   g. Find edge atoms (find_edge_atoms)
   h. Verify edge atoms don't change type (verify_edge_atoms)
   i. Extend if necessary (extend_edge_atoms)
   j. Add byproducts (get_byproducts)

4. Create partial map and renumber
   - Call create_partial_map
   - Get renumbering dictionaries
   - Renumber bonding, delete, create atoms

5. Rebuild molecule files with partial structure
   - Call lammps_to_molecule with validIDSet

6. Output map file
   - Format map with all sections
   - Save as automap.data
```

**Detailed Subroutine: is_cyclic**
```
Input: atomObjectDict, bondingAtoms, reactionType
Output: preservedAtomIDs dict

For each bondingAtom b in bondingAtoms:
    moleculeGraph = adjacency dict from atom objects
    preservedAtomIDs[b] = None
    
    For each neighbor n of b:
        Remove edge (b, n) from graph
        path = BFS(moleculeGraph, n, b)
        
        If path exists:
            # Cycle found
            cycleAtoms = set(path) ∪ {b}
            
            # Include all neighbors of cycle atoms
            For each atom a in cycleAtoms:
                preservedAtomIDs[b] = cycleAtoms ∪ neighbors(a)
            break
            
Return preservedAtomIDs
```

**Detailed Subroutine: keep_all_neighbours**
```
Input: atomObjectDict, bondingAtoms, partialAtomsSet
Output: Updated partialAtomsSet

maxDistance = 4

For each bondingAtom b:
    currentLevel = {b}
    visited = {b}
    
    For distance d = 1 to maxDistance:
        nextLevel = {}
        For each atom a in currentLevel:
            For each neighbor n of a:
                If n not in visited:
                    nextLevel.add(n)
                    visited.add(n)
        currentLevel = nextLevel
    
    partialAtomsSet = partialAtomsSet ∪ visited

Return partialAtomsSet
```

**Detailed Subroutine: find_edge_atoms**
```
Input: atomObjectDict, partialAtomsSet
Output: edgeAtoms list

edgeAtoms = []

For each atom a in partialAtomsSet:
    neighbors = atomObjectDict[a].firstNeighbourIDs
    For each neighbor n in neighbors:
        If n not in partialAtomsSet:
            # Atom a has neighbor outside partial structure
            edgeAtoms.append(a)
            break

Return edgeAtoms
```

**Detailed Subroutine: create_partial_map**
```
Input: mappedIDList, prePartialAtomsSet, postPartialAtomsSet
Output: filteredMap, preRenumberDict, postRenumberDict, partialMappedIDList

# Filter map to only include partial structure atoms
filteredMap = []
partialMappedIDList = []

For each [preID, postID] in mappedIDList:
    If preID in prePartialAtomsSet AND postID in postPartialAtomsSet:
        filteredMap.append([preID, postID])
    Else:
        partialMappedIDList.append([preID, postID])

# Create renumbering dictionaries
preAtomsSorted = naturalsort(prePartialAtomsSet)
postAtomsSorted = naturalsort(postPartialAtomsSet)

preRenumberDict = {preAtomsSorted[i]: str(i+1) for i in range(len(preAtomsSorted))}
postRenumberDict = {postAtomsSorted[i]: str(i+1) for i in range(len(postAtomsSorted))}

# Apply renumbering to filtered map
renamedMap = [[preRenumberDict[pre], postRenumberDict[post]] 
              for [pre, post] in filteredMap]

Return renamedMap, preRenumberDict, postRenumberDict, partialMappedIDList
```

### 3.4 PathSearch Algorithm (Detailed)

**Function:** `map_from_path(...)`

**Complete Algorithm:**
```
1. Initialize
   - Create element dictionaries for pre and post molecules
   - Build atom object dictionaries
   - Create empty mappedIDList
   - Create empty Queue
   - Create empty missingPreAtomList, missingPostAtomList

2. Map delete atoms (if provided)
   - Add user-specified delete atom pairs to mappedIDList

3. Queue bonding atoms
   - For each bonding pair (pre, post):
       * Add pair to mappedIDList
       * Add pair to Queue

4. Run BFS mapping (run_queue)
   While Queue not empty:
       [preAtom, postAtom] = Queue.get()
       
       # Update mapped neighbors
       preAtom.check_mapped(mappedIDList, 0, preElementDict)
       postAtom.check_mapped(mappedIDList, 1, postElementDict)
       
       # Map neighbors
       newMap, missingPre, missingPost, queueAtoms = 
           preAtom.map_elements(postAtom, preAtomObjectDict, postAtomObjectDict)
       
       # Add results
       mappedIDList.extend(newMap)
       missingPreAtomList.extend(missingPre)
       missingPostAtomList.extend(missingPost)
       Queue.add(queueAtoms)

5. Handle missing atoms
   missingCheckCounter = 1
   While missingCheckCounter < 4 AND len(missingPostAtomObjects) > 0:
       For each missingPreAtom:
           # Get atom objects for missing atoms
           Find matches in missingPostAtoms by:
               a. Element match
               b. Neighbor count match
               c. Fingerprint match (if inference allowed)
           
           If match found:
               Add to mappedIDList
               Add to Queue if not hydrogen
               Remove from missing lists
       
       missingCheckCounter += 1

6. Map create atoms (if provided)
   - Add (createAtomPre, createAtomPost) pairs to mappedIDList

7. Return mappedIDList
```

**Key Subroutine: compare_symmetric_atoms**
```
Input: postNeighbourAtomObjectList, preNeighbourAtom, outputType
Output: matched atom index or ID

# Try first-order fingerprint
uniqueMatches1 = find_unique_fingerprint_matches(
    postNeighbourAtomObjectList, 
    preNeighbourAtom, 
    'firstNeighbourElements'
)
If uniqueMatches1 has exactly one match:
    Return match

# Try second-order fingerprint
uniqueMatches2 = find_unique_fingerprint_matches(
    postNeighbourAtomObjectList, 
    preNeighbourAtom, 
    'secondNeighbourElements'
)
If uniqueMatches2 has exactly one match:
    Return match

# Try third-order fingerprint
uniqueMatches3 = find_unique_fingerprint_matches(
    postNeighbourAtomObjectList, 
    preNeighbourAtom, 
    'thirdNeighbourElements'
)
If uniqueMatches3 has exactly one match:
    Return match

# Use inference (first element match)
If allowInference:
    possibleChoices = [atom for atom in postNeighbourAtomObjectList 
                       if atom.element == preNeighbourAtom.element]
    If possibleChoices not empty:
        Print warning about inference
        Return possibleChoices[0]

Return None
```

## 4. Module Dependencies and Interfaces

### 4.1 Dependency Graph

```
AutoMapper.py
    ├── requires: argparse, sys, natsort
    ├── imports: LammpsUnifiedCleaner, LammpsToMolecule, MapProcessor
    └── calls: file_unifier(), lammps_to_molecule(), map_processor()

LammpsUnifiedCleaner
    ├── requires: os, natsort
    ├── imports: LammpsTreatmentFuncs, LammpsSearchFuncs
    └── exports: file_unifier()

LammpsToMolecule
    ├── requires: os, natsort
    ├── imports: LammpsSearchFuncs, LammpsTreatmentFuncs
    └── exports: lammps_to_molecule()

MapProcessor
    ├── requires: os, logging, contextlib, natsort, copy
    ├── imports: PathSearch, LammpsToMolecule, LammpsTreatmentFuncs, 
    │            LammpsSearchFuncs, AtomObjectBuilder
    └── exports: map_processor()

PathSearch
    ├── requires: os, logging, sys
    ├── imports: LammpsSearchFuncs, AtomObjectBuilder, QueueFuncs
    └── exports: map_from_path()

AtomObjectBuilder
    ├── requires: logging, collections
    ├── imports: LammpsSearchFuncs, LammpsTreatmentFuncs
    └── exports: Atom class, build_atom_objects(), compare_symmetric_atoms()

QueueFuncs
    ├── requires: logging, collections
    └── exports: Queue class, queue_bond_atoms(), run_queue()

LammpsSearchFuncs
    ├── requires: natsort
    ├── imports: LammpsTreatmentFuncs
    └── exports: get_data(), find_sections(), get_neighbours(), etc.

LammpsTreatmentFuncs
    ├── requires: natsort
    └── exports: clean_data(), save_text_file(), etc.
```

### 4.2 Function Interfaces

**LammpsUnifiedCleaner.file_unifier**
```python
def file_unifier(directory: str, coeffFile: str, dataFiles: list[str]) -> None:
    """
    Unify atom types across multiple LAMMPS data files.
    
    Args:
        directory: Working directory path
        coeffFile: Coefficient file name
        dataFiles: List of data file names
    
    Side Effects:
        - Creates 'cleaned<filename>' for each data file
        - Creates 'cleaned<coeffFile>' if provided
    
    Raises:
        FileNotFoundError: If files don't exist
        ValueError: If file format is invalid
    """
```

**LammpsToMolecule.lammps_to_molecule**
```python
def lammps_to_molecule(
    directory: str,
    fileName: str,
    saveName: str,
    bondingAtoms: list[str] = None,
    deleteAtoms: list[str] = None,
    validIDSet: set[str] = None,
    renumberedAtomDict: dict[str, str] = None
) -> None:
    """
    Convert LAMMPS data file to molecule format.
    
    Args:
        directory: Working directory path
        fileName: Input LAMMPS data file name
        saveName: Output molecule file name
        bondingAtoms: List of bonding atom IDs (optional)
        deleteAtoms: List of atoms to mark for deletion (optional)
        validIDSet: Set of atom IDs to include (for partial structures)
        renumberedAtomDict: Mapping of old to new atom IDs (for renumbering)
    
    Side Effects:
        - Creates '<saveName>' molecule file in directory
    
    Raises:
        FileNotFoundError: If fileName doesn't exist
        ValueError: If file format is invalid
    """
```

**MapProcessor.map_processor**
```python
def map_processor(
    directory: str,
    preDataFileName: str,
    postDataFileName: str,
    preMoleculeFileName: str,
    postMoleculeFileName: str,
    preBondingAtoms: list[str],
    postBondingAtoms: list[str],
    deleteAtoms: list[str],
    elementsByType: list[str],
    createAtoms: list[str],
    debug: bool = False
) -> list[list[list[str, str]], list[list[str, str]]]:
    """
    Create molecule files and equivalence map for bond/react.
    
    Args:
        directory: Working directory path
        preDataFileName: Pre-reaction data file name
        postDataFileName: Post-reaction data file name
        preMoleculeFileName: Output pre-reaction molecule file name
        postMoleculeFileName: Output post-reaction molecule file name
        preBondingAtoms: Pre-reaction bonding atom IDs (length >= 2)
        postBondingAtoms: Post-reaction bonding atom IDs (length >= 2)
        deleteAtoms: Atom IDs to delete (pre then post, even length)
        elementsByType: Element symbols ordered by type number
        createAtoms: Atom IDs to create in post-reaction
        debug: Enable debug logging
    
    Returns:
        [mappedIDList, partialMappedIDList]
        - mappedIDList: Full equivalence map
        - partialMappedIDList: Atoms excluded from partial structure
    
    Side Effects:
        - Creates '<preMoleculeFileName>' molecule file
        - Creates '<postMoleculeFileName>' molecule file
        - Creates 'automap.data' map file
    
    Raises:
        ValueError: If bonding atoms have wrong length
        AssertionError: If delete atoms have uneven length
    """
```

**PathSearch.map_from_path**
```python
def map_from_path(
    directory: str,
    preMoleculeFileName: str,
    postMoleculeFileName: str,
    elementsByType: list[str],
    debug: bool,
    preBondingAtoms: list[str],
    preDeleteAtoms: list[str],
    postBondingAtoms: list[str],
    postDeleteAtoms: list[str],
    createAtoms: list[str]
) -> list[list[str, str]]:
    """
    Create equivalence map using BFS path search.
    
    Args:
        directory: Working directory path
        preMoleculeFileName: Pre-reaction molecule file name
        postMoleculeFileName: Post-reaction molecule file name
        elementsByType: Element symbols ordered by type number
        debug: Enable debug logging
        preBondingAtoms: Pre-reaction bonding atom IDs
        preDeleteAtoms: Pre-reaction delete atom IDs
        postBondingAtoms: Post-reaction bonding atom IDs
        postDeleteAtoms: Post-reaction delete atom IDs
        createAtoms: Post-reaction create atom IDs
    
    Returns:
        mappedIDList: List of [preAtomID, postAtomID] pairs
    
    Raises:
        ValueError: If mapping fails
    """
```

**AtomObjectBuilder.build_atom_objects**
```python
def build_atom_objects(
    fileName: str,
    elementDict: dict[str, str],
    bondingAtoms: list[str],
    createAtoms: list[str] = []
) -> dict[str, Atom]:
    """
    Build dictionary of Atom objects from molecule file.
    
    Args:
        fileName: Molecule file name
        elementDict: Mapping of atom IDs to element symbols
        bondingAtoms: List of bonding atom IDs
        createAtoms: List of create atom IDs (excluded from objects)
    
    Returns:
        atomObjectDict: Dictionary mapping atom IDs to Atom objects
    
    Raises:
        FileNotFoundError: If fileName doesn't exist
        KeyError: If element not found for atom ID
    """
```

## 5. Error Handling and Validation

### 5.1 Input Validation

**Command-Line Arguments:**
```python
# Check required arguments per tool
if tool == 'clean' and not coeff_file:
    raise ArgumentError("clean tool requires --coeff_file")

if tool == 'molecule' and not save_name:
    raise ArgumentError("molecule tool requires --save_name")

if tool == 'map' and (len(data_files) != 2 or len(save_name) != 2):
    raise ArgumentError("map tool requires 2 data_files and 2 save_names")

if tool == 'map' and (len(ba) < 4 or not ebt):
    raise ArgumentError("map tool requires --ba and --ebt arguments")
```

**File Validation:**
```python
# Check file existence
if not os.path.exists(filepath):
    raise FileNotFoundError(f"File {filepath} not found")

# Check file format
if not has_valid_lammps_header(data):
    raise ValueError(f"Invalid LAMMPS file format: {filepath}")
```

**Data Consistency:**
```python
# Check atom ID uniqueness
if len(atomIDs) != len(set(atomIDs)):
    raise ValueError("Duplicate atom IDs found")

# Check bonding atom validity
for bondingAtom in bondingAtoms:
    if bondingAtom not in atomIDs:
        raise ValueError(f"Bonding atom {bondingAtom} not in atom list")

# Check element count
if len(elementsByType) != numAtomTypes:
    raise ValueError("Element list length doesn't match atom type count")
```

### 5.2 Runtime Error Handling

**Missing Atom Detection:**
```python
if len(missingPostAtomList) > 0 after all mapping attempts:
    logging.warning(f"Missing post atoms: {missingPostAtomList}")
    # Continue with available mappings
```

**Cycle Detection Failure:**
```python
if cyclic_path is None:
    # Not a cycle, preservedAtomIDs[bondingAtom] remains None
    # Continue without cycle preservation
```

**Symmetric Atom Resolution:**
```python
if postNeighbourAtomID is None:
    if allowInference:
        # Use first element match with warning
        logging.warning(f"Inference used for atom {preAtomID}")
    else:
        # Add to missing atoms
        missingPreAtoms.append(preAtomID)
```

### 5.3 Logging Levels

**DEBUG Level:**
- Detailed mapping steps
- Fingerprint matches
- Cycle detection paths
- Queue operations

**INFO Level:**
- Tool start/completion
- File creation
- Major algorithm phases

**WARNING Level:**
- Inference-based mappings
- Missing atoms
- Potential issues

**ERROR Level:**
- File not found
- Invalid format
- Critical failures

## 6. Performance Considerations

### 6.1 Optimization Strategies

**Early Hydrogen Termination:**
Hydrogen atoms are not added to the queue in BFS, reducing queue size by ~50% for typical organic molecules.

**Natural Sorting Caching:**
Use natsort for consistent ordering without repeated conversions:
```python
sorted_atoms = natsorted(atomIDs)  # Cache result
```

**Set Operations for Atom Filtering:**
Use sets for O(1) membership testing:
```python
validAtomSet = set(validAtomIDs)
filtered = [a for a in atoms if a in validAtomSet]
```

**Neighbor Caching:**
Store computed neighbors at all levels in Atom objects to avoid recomputation.

### 6.2 Memory Management

**Atom Object Pruning:**
Remove create atoms from atom object dictionaries to reduce memory:
```python
if createAtoms is not None:
    if atomID in createAtoms:
        continue  # Skip creating Atom object
```

**Queue Size Management:**
Non-hydrogen atoms only added to queue, typically 5-10% of total atoms.

**File Streaming:**
Large files could be processed in chunks, but current implementation loads entirely into memory for simplicity.

### 6.3 Scalability Limits

**Current Implementation:**
- Tested up to ~10,000 atoms
- Memory: O(n) where n = atom count
- Time: O(n) for sparse molecular graphs

**Potential Bottlenecks:**
- Dense molecular graphs (rare in chemistry)
- Highly symmetric structures requiring extensive fingerprinting
- Very large ring systems (>20 atoms in ring)

## 7. Testing and Validation

### 7.1 Unit Test Coverage

**Test Files:**
- `test_LammpsSearchFuncs.py`: Tests for file parsing functions
- `test_LammpsToMolecule.py`: Tests for molecule conversion
- `test_UnifiedCleaner.py`: Tests for type unification

**Test Cases:**
- Valid input parsing
- Edge cases (empty sections, single atom)
- Error conditions (missing files, invalid format)

### 7.2 Integration Testing

**MapTesting.py:**
Provides test functions for complete workflow:
- `test_molecule_creation()`: Verify molecule file output
- `test_map_creation()`: Verify map file correctness
- `test_partial_structure()`: Verify partial structure logic

### 7.3 Validation Criteria

**Correctness:**
1. Element conservation: All mapped atoms preserve element
2. Connectivity preservation: Bonds maintained correctly
3. ID uniqueness: No duplicate atom IDs in output

**Completeness:**
1. All reachable atoms mapped
2. All bonding atoms included
3. All delete/create atoms handled

**Minimality:**
1. Partial structure as small as possible
2. No redundant atoms included
3. Edge atoms properly identified

## 8. Configuration and Customization

### 8.1 Configurable Parameters

**Distance Threshold (maxDistance):**
```python
maxDistance = 4  # Atoms within 4 bonds of bonding atoms preserved
```
Modifiable in `keep_all_neighbours()` function.

**Fingerprint Depth:**
```python
maxFingerprintLevel = 3  # Up to 3rd order neighbors
```
Modifiable in `Atom` class and `compare_symmetric_atoms()`.

**Inference Policy:**
```python
allowInference = True  # Allow element-based inference
```
Modifiable in `compare_symmetric_atoms()` calls.

### 8.2 Extension Points

**Custom Atom Properties:**
Extend `Atom` class with additional attributes:
```python
class ExtendedAtom(Atom):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.customProperty = value
```

**Alternative Matching Algorithms:**
Replace `compare_symmetric_atoms()` with custom implementation:
```python
def custom_symmetric_matcher(postAtoms, preAtom, outputType):
    # Custom matching logic
    return matched_atom
```

**Custom File Formats:**
Implement alternative parsers in new module:
```python
def parse_custom_format(filename):
    # Custom parsing logic
    return atomData, bondData, etc.
```

## 9. Deployment and Distribution

### 9.1 Dependencies

**Required:**
- Python 3.6+
- natsort (pip install natsort)

**Optional:**
- pytest (for testing)
- numpy (for future numerical extensions)

### 9.2 Installation

```bash
# Clone repository
git clone https://github.com/nobkt/AutoMapper.git

# Add to PATH (Linux/Mac)
export PATH=$PATH:/path/to/AutoMapper

# Make executable
chmod +x /path/to/AutoMapper/AutoMapper.py
```

### 9.3 Usage Patterns

**Basic Workflow:**
```bash
# Step 1: Clean files
AutoMapper.py . clean pre.data post.data --coeff_file settings.in

# Step 2: Create map
AutoMapper.py . map cleanedpre.data cleanedpost.data \
    --save_name pre-mol.data post-mol.data \
    --ba 1 4 2 5 \
    --ebt H C N O \
    --debug
```

**Advanced Options:**
```bash
# With delete atoms
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --da 5 6 7 8 \
    --ebt H C N O P S

# With create atoms
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --ca 5 6 \
    --ebt H C N O
```

## 10. Maintenance and Support

### 10.1 Known Issues

**Issue 1: Deprecation Warning**
- LAMMPS versions 29Sept21+ prefer "Initiator IDs" over "Bonding IDs"
- Current implementation uses "BondingIDs"
- **Compatibility:** Works with all LAMMPS versions but may show deprecation warning
- **Migration Path:** For newest LAMMPS (29Sept21+), manually edit map file:
  - Find: "BondingIDs"
  - Replace with: "Initiator IDs"
- **Production Recommendation:** Test with your specific LAMMPS version before production use
- Workaround: Manually edit map file if needed

**Issue 2: Wildcards in Coefficients**
- Clean tool requires numeric pair coefficients
- Wildcards (*) not supported
- Workaround: Expand wildcards before using clean tool

### 10.2 Future Enhancements

**Planned Features:**
1. Support for additional atom styles (not just 'full')
2. Stereochemistry handling
3. Parallel processing for large molecules
4. GUI interface
5. Integration with molecular visualization tools

### 10.3 Contribution Guidelines

**Code Standards:**
- Follow PEP 8 style guide
- Add docstrings to all functions
- Include type hints where possible
- Write unit tests for new features

**Pull Request Process:**
1. Fork repository
2. Create feature branch
3. Implement changes with tests
4. Submit pull request with description
5. Address review comments

## 11. References and Related Work

### 11.1 LAMMPS Documentation
- fix bond/react: https://docs.lammps.org/fix_bond_react.html
- Molecule files: https://docs.lammps.org/molecule.html
- Data files: https://docs.lammps.org/read_data.html

### 11.2 Academic References
- Bone, M., et al. (2022). "AutoMapper: Automated creation of reaction templates for LAMMPS molecular dynamics simulations." Computational Materials Science. DOI: 10.1016/j.commatsci.2022.111204

### 11.3 Related Tools
- Moltemplate: Molecular template builder
- VMD: Visual Molecular Dynamics
- OVITO: Visualization and analysis software
