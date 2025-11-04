# AutoMapper: Detailed Theoretical Foundations with Mathematical Formulations

## 1. Introduction and Theoretical Framework

AutoMapper is a computational tool designed to automate the generation of input files for LAMMPS molecular dynamics simulations, specifically for the `fix bond/react` command. The theoretical foundation of AutoMapper is based on graph theory, molecular topology, and combinatorial optimization.

## 2. Mathematical Representation of Molecular Structures

### 2.1 Molecular Graph Representation

A molecule is represented as an undirected graph G = (V, E) where:

- **V = {v₁, v₂, ..., vₙ}**: Set of vertices representing atoms
- **E ⊆ V × V**: Set of edges representing chemical bonds

Each vertex vᵢ has associated properties:
- **aᵢ**: Atom identifier (unique integer)
- **tᵢ**: Atom type (integer representing force field type)
- **eᵢ**: Chemical element symbol (H, C, N, O, etc.)
- **xᵢ, yᵢ, zᵢ**: Cartesian coordinates

Each edge (vᵢ, vⱼ) ∈ E represents a chemical bond with properties:
- **bᵢⱼ**: Bond type (single, double, triple, etc.)

### 2.2 Adjacency Relations

For any atom vᵢ, we define k-th order neighbor sets:

**First-order neighbors (directly bonded atoms):**
```
N₁(vᵢ) = {vⱼ ∈ V : (vᵢ, vⱼ) ∈ E}
```

**Second-order neighbors (atoms 2 bonds away):**
```
N₂(vᵢ) = {vₖ ∈ V : ∃vⱼ ∈ N₁(vᵢ), vₖ ∈ N₁(vⱼ), vₖ ∉ N₁(vᵢ) ∪ {vᵢ}}
```

**Third-order neighbors (atoms 3 bonds away):**
```
N₃(vᵢ) = {vₘ ∈ V : ∃vₖ ∈ N₂(vᵢ), vₘ ∈ N₁(vₖ), vₘ ∉ N₂(vᵢ) ∪ N₁(vᵢ) ∪ {vᵢ}}
```

**General k-th order neighbors:**
```
Nₖ(vᵢ) = {v ∈ V : d(vᵢ, v) = k} \ ⋃ⱼ₌₁ᵏ⁻¹ Nⱼ(vᵢ)
```

where d(vᵢ, v) is the graph distance (shortest path length) between vᵢ and v.

## 3. Atom Fingerprinting and Molecular Topology

### 3.1 Element-Based Fingerprints

For each atom vᵢ, we define element fingerprints based on neighbor sets:

**First-order element fingerprint:**
```
F₁(vᵢ) = sort({eⱼ : vⱼ ∈ N₁(vᵢ)})
```

**Second-order element fingerprint:**
```
F₂(vᵢ) = sort({eₖ : vₖ ∈ N₂(vᵢ)})
```

**Third-order element fingerprint:**
```
F₃(vᵢ) = sort({eₘ : vₘ ∈ N₃(vᵢ)})
```

The sort operation creates an alphabetically ordered concatenated string of element symbols, ensuring fingerprint uniqueness independent of atom ordering.

### 3.2 Fingerprint Uniqueness Criteria

For a set of atoms S = {v₁, v₂, ..., vₘ} with the same element e, an atom vᵢ ∈ S is uniquely identifiable at level k if:

```
∀vⱼ ∈ S, j ≠ i : Fₖ(vᵢ) ≠ Fₖ(vⱼ)
```

**Uniqueness Hierarchy:**
1. Check F₁(vᵢ) uniqueness
2. If not unique, check F₂(vᵢ) uniqueness
3. If still not unique, check F₃(vᵢ) uniqueness
4. If none provide uniqueness, use element-based inference

### 3.3 Element Multiplicity Counting

For element occurrence comparison:

```
C(e, vᵢ, k) = |{v ∈ Nₖ(vᵢ) : element(v) = e}|
```

This counts occurrences of element e in the k-th order neighborhood of vᵢ.

## 4. Molecular Mapping Problem

### 4.1 Problem Formulation

Given:
- Pre-reaction graph: Gₚᵣₑ = (Vₚᵣₑ, Eₚᵣₑ)
- Post-reaction graph: Gₚₒₛₜ = (Vₚₒₛₜ, Eₚₒₛₜ)
- Bonding atoms: Bₚᵣₑ ⊂ Vₚᵣₑ, Bₚₒₛₜ ⊂ Vₚₒₛₜ with user-defined correspondence

Find: Bijective mapping function M : Vₚᵣₑ → Vₚₒₛₜ such that:

```
M(vᵢ) = vⱼ ⟺ vᵢ and vⱼ represent the same atom before and after reaction
```

**Constraints:**
1. Element conservation: e(vᵢ) = e(M(vᵢ)) for all mapped atoms
2. Bonding atom correspondence: ∀b ∈ Bₚᵣₑ, M(b) ∈ Bₚₒₛₜ as specified by user
3. Topology preservation: Local neighborhood structures should be preserved where possible

### 4.2 Mapping as Graph Isomorphism Subproblem

The mapping problem can be viewed as finding a graph isomorphism between Gₚᵣₑ and Gₚₒₛₜ with:
- Deleted vertices: Vₐₑₗ = {v ∈ Vₚᵣₑ : M(v) is undefined}
- Created vertices: Vₐₑw = {v ∈ Vₚₒₛₜ : M⁻¹(v) is undefined}
- Modified edges: E_modified = (Eₚₒₛₜ \ Eₚᵣₑ) ∪ (Eₚᵣₑ \ Eₚₒₛₜ)

## 5. Breadth-First Search Algorithm for Mapping

### 5.1 Queue-Based BFS Formulation

The mapping algorithm uses a modified breadth-first search starting from bonding atoms:

**Initialization:**
```
Q ← Queue()
M ← ∅  (empty mapping)
For each bonding pair (bₚᵣₑ, bₚₒₛₜ):
    M ← M ∪ {(bₚᵣₑ, bₚₒₛₜ)}
    Q.enqueue((bₚᵣₑ, bₚₒₛₜ))
```

**Main Loop:**
```
While Q is not empty:
    (vₚᵣₑ, vₚₒₛₜ) ← Q.dequeue()
    
    N'ₚᵣₑ ← N₁(vₚᵣₑ) \ {v : (v, w) ∈ M for some w}
    N'ₚₒₛₜ ← N₁(vₚₒₛₜ) \ {w : (v, w) ∈ M for some v}
    
    For each unmapped neighbor nₚᵣₑ ∈ N'ₚᵣₑ:
        nₚₒₛₜ ← find_correspondence(nₚᵣₑ, N'ₚₒₛₜ, M)
        If nₚₒₛₜ is found:
            M ← M ∪ {(nₚᵣₑ, nₚₒₛₜ)}
            If element(nₚᵣₑ) ≠ H:
                Q.enqueue((nₚᵣₑ, nₚₒₛₜ))
```

### 5.2 Neighbor Correspondence Algorithm

The `find_correspondence` function determines atom correspondence:

```
function find_correspondence(nₚᵣₑ, N'ₚₒₛₜ, M):
    e ← element(nₚᵣₑ)
    
    // Filter candidates by element
    candidates ← {n ∈ N'ₚₒₛₜ : element(n) = e}
    
    If |candidates| = 0:
        return NULL (missing atom)
    
    If |candidates| = 1:
        return candidates[0]
    
    If e = 'H':
        // All hydrogen atoms are equivalent
        return any element from candidates
    
    // Use fingerprint matching for non-hydrogen atoms
    For k = 1 to 3:
        matches ← {c ∈ candidates : Fₖ(nₚᵣₑ) = Fₖ(c)}
        
        // Check uniqueness
        unique_fingerprints ← {}
        For each c in matches:
            If count(Fₖ(c) in candidates) = 1:
                unique_fingerprints ← unique_fingerprints ∪ {c}
        
        If nₚᵣₑ fingerprint is unique and matches exactly one in unique_fingerprints:
            return that match
    
    // Inference fallback
    return candidates[0] with warning
```

### 5.3 Element Occurrence Constraint

Before mapping neighbors, verify element occurrence consistency:

```
function allowed_maps(vₚᵣₑ, vₚₒₛₜ):
    For each element e:
        If C(e, vₚᵣₑ, 1) ≠ C(e, vₚₒₛₜ, 1) and e ≠ 'H':
            return False for element e
    return True for all elements (with H always True)
```

## 6. Cycle Detection and Ring Systems

### 6.1 Breadth-First Search for Cycles

To detect if a bonding atom bᵢ is part of a cyclic structure:

```
function is_cyclic(G, bᵢ):
    For each neighbor n ∈ N₁(bᵢ):
        // Search for path from n back to bᵢ without using the direct edge
        G' ← G with edge (bᵢ, n) removed
        path ← BFS(G', n, bᵢ)
        If path exists:
            return (True, path ∪ {bᵢ})
    return (False, ∅)
```

The BFS algorithm for path finding:

```
function BFS(G, start, target):
    discovered ← {v : False for all v ∈ V}
    discovered[start] ← True
    queue ← [[start]]
    
    While queue is not empty:
        path ← queue.dequeue()
        node ← path[-1]
        
        If node = target:
            return path
        
        For each neighbor n ∈ N₁(node) in G:
            If not discovered[n]:
                discovered[n] ← True
                new_path ← path + [n]
                queue.enqueue(new_path)
    
    return NULL
```

### 6.2 Preserved Atom Set for Cyclic Structures

When a cycle is detected with path P = {p₁, p₂, ..., pₖ}, the preserved atom set is:

```
Pᵣₑₛₑᵣᵥₑₐ(P) = P ∪ ⋃ᵢ₌₁ᵏ N₁(pᵢ)
```

This ensures all cycle atoms and their immediate neighbors are preserved in partial structures.

### 6.3 Ring Opening Detection

For ring opening reactions, compare pre- and post-reaction preserved sets:

```
function is_ring_opening(Pₚᵣₑ, Pₚₒₛₜ, M):
    If Pₚᵣₑ ≠ ∅ and Pₚₒₛₜ = ∅:
        // Ring is present in pre but not post
        return (Pₚᵣₑ, mapped_atoms(Pₚᵣₑ, M))
    Else:
        return (∅, ∅)
```

## 7. Partial Structure Minimization

### 7.1 Neighbor Distance Preservation

To create minimal partial structures, keep atoms within distance d from bonding atoms:

```
Sₐ(B, d) = {v ∈ V : min{δ(v, b) : b ∈ B} ≤ d}
```

where δ(v, b) is the shortest path distance between v and b.

For AutoMapper, d = 4 is used:

```
Sₚₐᵣₜᵢₐₗ = Sₐ(B, 4) ∪ Pᵣₑₛₑᵣᵥₑₐ ∪ Dₑₗₑₜₑ ∪ Cᵣₑₐₜₑ
```

### 7.2 Edge Atom Identification

Edge atoms are boundary atoms of the partial structure:

```
Eₐgₑ = {v ∈ Sₚₐᵣₜᵢₐₗ : ∃n ∈ N₁(v), n ∉ Sₚₐᵣₜᵢₐₗ}
```

### 7.3 Edge Atom Verification and Extension

Edge atoms must not be too close to atoms that change type during reaction:

```
function verify_edge_atoms(Eₐgₑ, M, Gₚᵣₑ, Gₚₒₛₜ):
    extend_dict ← {}
    For each eₚᵣₑ ∈ Eₐgₑ:
        eₚₒₛₜ ← M(eₚᵣₑ)
        If type(eₚᵣₑ) ≠ type(eₚₒₛₜ):
            // Edge atom changes type, extend to its neighbors
            extend_dict[eₚᵣₑ] ← N₁(eₚᵣₑ)
    return extend_dict
```

### 7.4 Byproduct Molecule Detection

Byproducts are disconnected molecular fragments:

```
function get_byproducts(Gₚₒₛₜ, Bₚₒₛₜ):
    // Perform connected component analysis
    visited ← {v : False for all v ∈ Vₚₒₛₜ}
    main_component ← BFS_component(Gₚₒₛₜ, Bₚₒₛₜ[0])
    
    For each v in main_component:
        visited[v] ← True
    
    byproducts ← {}
    For each v ∈ Vₚₒₛₜ:
        If not visited[v]:
            byproducts ← byproducts ∪ {v}
    
    return byproducts
```

## 8. Atom Renumbering for Partial Structures

### 8.1 Sequential Renumbering Function

When creating partial structures, atoms must be renumbered sequentially:

```
function create_renumbering_map(Sₚₐᵣₜᵢₐₗ):
    sorted_atoms ← sort(Sₚₐᵣₜᵢₐₗ)  // Natural sort
    renumber_map ← {}
    For i = 1 to |sorted_atoms|:
        renumber_map[sorted_atoms[i]] ← i
    return renumber_map
```

### 8.2 Map Transformation

The equivalence map M is transformed for partial structures:

```
M'ₚₐᵣₜᵢₐₗ = {(Rₚᵣₑ(vₚᵣₑ), Rₚₒₛₜ(vₚₒₛₜ)) : (vₚᵣₑ, vₚₒₛₜ) ∈ M, 
              vₚᵣₑ ∈ Sₚₐᵣₜᵢₐₗ,ₚᵣₑ, vₚₒₛₜ ∈ Sₚₐᵣₜᵢₐₗ,ₚₒₛₜ}
```

where Rₚᵣₑ and Rₚₒₛₜ are the renumbering maps for pre- and post-reaction structures.

## 9. Computational Complexity Analysis

### 9.1 Time Complexity

**Atom Object Construction:** O(n · m)
- n = number of atoms
- m = average number of bonds per atom (typically constant ≈ 3-4)

**BFS Mapping:** O(n · (n + m))
- Each atom processed once: O(n)
- Fingerprint computation per atom: O(n + m) in worst case
- Total: O(n²) for dense graphs, O(n) for sparse molecular graphs

**Cycle Detection:** O(n · (n + m))
- BFS performed for each bonding atom
- Typically small number of bonding atoms (2-4)

**Overall Complexity:** O(n²) worst case, O(n) typical case for molecular structures

### 9.2 Space Complexity

**Atom Objects:** O(n · k)
- k = maximum neighbor order (k = 3 in AutoMapper)

**Queue and Mapping:** O(n)

**Overall Space Complexity:** O(n)

## 10. Theoretical Guarantees and Limitations

### 10.1 Correctness Guarantees

**Theorem 1 (Element Conservation):** 
For all mapped atoms (vₚᵣₑ, vₚₒₛₜ) ∈ M, element(vₚᵣₑ) = element(vₚₒₛₜ).

*Proof:* By construction in find_correspondence, candidates are filtered by element before assignment.

**Theorem 2 (Local Topology Preservation):**
If no atoms are deleted or created in the neighborhood of a mapped atom pair (vₚᵣₑ, vₚₒₛₜ), then the neighbor sets are isomorphic.

*Proof:* The BFS algorithm maps all neighbors sequentially, preserving the connectivity structure.

### 10.2 Limitations and Edge Cases

**Symmetry Ambiguity:**
When multiple atoms have identical fingerprints up to 3rd order neighbors, the algorithm uses inference (first available match). This may be incorrect in rare cases of high molecular symmetry.

**Chirality:**
The algorithm does not explicitly handle stereochemistry. Chiral centers may be mapped incorrectly if mirror images exist.

**Large Ring Systems:**
For molecules with many large rings, the preserved atom set may become excessively large, limiting partial structure minimization.

## 11. Extensions and Future Directions

### 11.1 Higher-Order Fingerprints

Extending fingerprints to k > 3 orders could improve disambiguation:

```
Fₖ(vᵢ) = sort({eₘ : vₘ ∈ Nₖ(vᵢ)}), k > 3
```

Trade-off: Increased computational cost vs. improved uniqueness.

### 11.2 Stereochemistry Integration

Incorporating 3D geometric information:

```
F_stereo(vᵢ) = (F₁(vᵢ), chirality_descriptor(vᵢ))
```

where chirality_descriptor uses Cartesian coordinates to determine stereochemistry.

### 11.3 Machine Learning Enhancement

Use graph neural networks to learn optimal fingerprint representations:

```
h_vᵢ⁽ᵏ⁾ = σ(W⁽ᵏ⁾ · AGG({h_vⱼ⁽ᵏ⁻¹⁾ : vⱼ ∈ N₁(vᵢ)}))
```

where h_vᵢ⁽ᵏ⁾ is the k-th layer embedding of atom vᵢ.

## 12. Conclusion

AutoMapper's theoretical foundation combines graph theory, combinatorial optimization, and molecular topology to solve the molecular mapping problem efficiently. The BFS-based algorithm with fingerprint matching provides a robust solution with O(n) typical-case complexity and strong correctness guarantees under element conservation constraints. The cycle detection and partial structure minimization algorithms ensure minimal yet complete representations for LAMMPS bond/react simulations.
