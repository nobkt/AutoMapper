# AutoMapper: Complete User Manual

## Table of Contents

1. [Introduction](#1-introduction)
2. [Installation](#2-installation)
3. [Getting Started](#3-getting-started)
4. [Tool Overview](#4-tool-overview)
5. [Clean Tool](#5-clean-tool)
6. [Molecule Tool](#6-molecule-tool)
7. [Map Tool](#7-map-tool)
8. [Advanced Usage](#8-advanced-usage)
9. [File Formats](#9-file-formats)
10. [Troubleshooting](#10-troubleshooting)
11. [Examples and Workflows](#11-examples-and-workflows)
12. [FAQ](#12-faq)
13. [Best Practices](#13-best-practices)
14. [Integration with LAMMPS](#14-integration-with-lammps)

## 1. Introduction

### 1.1 What is AutoMapper?

AutoMapper is a command-line tool designed to automate the creation of input files required for LAMMPS molecular dynamics simulations using the `fix bond/react` command. It simplifies the process of setting up chemical reactions in molecular dynamics simulations by automatically:

- Converting LAMMPS data files to molecule format
- Creating equivalence maps between pre- and post-reaction structures
- Generating minimal partial structures for efficient simulations
- Unifying atom types across multiple simulation files

### 1.2 Who Should Use AutoMapper?

AutoMapper is designed for:

- **Computational chemists** studying reaction mechanisms
- **Materials scientists** modeling polymer reactions
- **Research engineers** simulating reactive molecular systems
- **Graduate students** learning LAMMPS and reactive MD
- **Anyone** using LAMMPS `fix bond/react` command

### 1.3 Key Features

- **Automatic Atom Mapping**: Uses graph algorithms to match atoms between pre- and post-reaction structures
- **Partial Structure Generation**: Creates minimal structures containing only reaction-relevant atoms
- **Ring Detection**: Identifies and preserves cyclic structures
- **Type Unification**: Ensures consistent atom types across multiple files
- **Force Field Independent**: Works with any force field
- **Moltemplate Compatible**: Designed to work seamlessly with Moltemplate output

### 1.4 System Requirements

**Required:**
- Python 3.6 or later
- `natsort` Python package

**Optional:**
- Linux/Unix environment (recommended)
- Text editor for viewing output files
- Molecular visualization software (VMD, OVITO, etc.)

### 1.5 Related Tools

- **LAMMPS**: Molecular dynamics simulation package
- **Moltemplate**: Molecule template builder
- **VMD**: Visual Molecular Dynamics
- **OVITO**: Visualization software

## 2. Installation

### 2.1 Quick Installation

**Step 1: Install Python**

Check if Python 3.6+ is installed:
```bash
python --version
# or on some systems:
python3 --version
```

If not installed, download from [python.org](https://www.python.org/) or use your package manager:
```bash
# Ubuntu/Debian
sudo apt-get install python3

# macOS
brew install python3

# Windows
# Download from python.org
```

**Step 2: Install natsort Package**

```bash
pip3 install natsort

# Or using conda
conda install -c conda-forge natsort
```

**Step 3: Clone AutoMapper Repository**

```bash
git clone https://github.com/nobkt/AutoMapper.git
cd AutoMapper
```

**Step 4: Make Executable (Linux/Mac)**

```bash
chmod +x AutoMapper.py
```

### 2.2 Adding to PATH

To use AutoMapper from anywhere:

**Linux/Mac:**
```bash
# Add to ~/.bashrc or ~/.bash_profile
export PATH=$PATH:/path/to/AutoMapper

# Apply changes
source ~/.bashrc
```

**Windows:**
1. Right-click "This PC" → Properties
2. Advanced system settings → Environment Variables
3. Edit PATH variable
4. Add AutoMapper directory
5. Restart terminal

### 2.3 Verification

Test installation:
```bash
# If in PATH
AutoMapper.py -h

# Or direct call
python3 /path/to/AutoMapper/AutoMapper.py -h
```

You should see help message showing available tools and options.

### 2.4 Troubleshooting Installation

**Issue: "python3 not found"**
- Install Python 3.6+
- Check PATH settings

**Issue: "natsort module not found"**
```bash
pip3 install natsort --user
```

**Issue: "Permission denied"**
```bash
chmod +x AutoMapper.py
```

## 3. Getting Started

### 3.1 Basic Workflow

The typical AutoMapper workflow consists of three steps:

```
1. CLEAN → Unify atom types across files
    ↓
2. (OPTIONAL) MOLECULE → Convert individual files
    ↓
3. MAP → Create reaction map
```

### 3.2 Your First AutoMapper Run

Let's create a simple reaction map for a bond formation reaction.

**Prepare Your Files:**

1. `pre-reaction.data` - LAMMPS data file before reaction
2. `post-reaction.data` - LAMMPS data file after reaction
3. `system.in.settings` - Coefficient file (optional for clean)

**Run Clean Tool:**
```bash
AutoMapper.py . clean pre-reaction.data post-reaction.data --coeff_file system.in.settings
```

This creates:
- `cleanedpre-reaction.data`
- `cleanedpost-reaction.data`
- `cleanedsystem.in.settings` (if provided)

**Run Map Tool:**
```bash
AutoMapper.py . map cleanedpre-reaction.data cleanedpost-reaction.data \
    --save_name pre-molecule.data post-molecule.data \
    --ba 1 4 2 5 \
    --ebt H C N O
```

This creates:
- `pre-molecule.data` - Pre-reaction molecule file
- `post-molecule.data` - Post-reaction molecule file
- `automap.data` - Equivalence map

### 3.3 Understanding the Output

**Molecule Files:**
- Contain atomic coordinates, types, bonds, angles, dihedrals
- In LAMMPS molecule format
- Can be visualized with VMD or other tools

**Map File (automap.data):**
- Contains equivalence mappings
- Specifies bonding atoms
- Lists edge atoms (boundary of partial structure)
- Used directly by LAMMPS `fix bond/react`

### 3.4 Command Structure

All AutoMapper commands follow this structure:

```bash
AutoMapper.py [directory] [tool] [files] [options]
```

**Components:**
- `directory`: Working directory (use `.` for current)
- `tool`: One of `clean`, `molecule`, or `map`
- `files`: Input file names
- `options`: Tool-specific options (--ba, --ebt, etc.)

## 4. Tool Overview

### 4.1 Tool Comparison

| Feature | Clean | Molecule | Map |
|---------|-------|----------|-----|
| Purpose | Unify types | Convert format | Create reaction |
| Input files | Multiple data files | One data file | Two data files |
| Output files | Cleaned data files | One molecule file | Two molecule files + map |
| Required args | --coeff_file | --save_name | --save_name, --ba, --ebt |
| Typical use | Preparation | Individual conversion | Full workflow |

### 4.2 When to Use Each Tool

**Use CLEAN when:**
- You have multiple LAMMPS data files from different sources
- Atom types are inconsistent between files
- You need unified coefficient files

**Use MOLECULE when:**
- You need to manually create molecule files
- You want to inspect individual molecule files before mapping
- You're debugging or testing

**Use MAP when:**
- You're ready to create a complete reaction setup
- You have pre- and post-reaction structures
- You want automated partial structure generation

## 5. Clean Tool

### 5.1 Purpose and Usage

The clean tool unifies atom types across multiple LAMMPS data files, ensuring consistency and removing unused force field coefficients.

### 5.2 Command Syntax

```bash
AutoMapper.py [directory] clean [data_files...] --coeff_file [coefficient_file]
```

**Arguments:**
- `data_files`: List of LAMMPS data files (space-separated)
- `--coeff_file`: Coefficient/settings file containing force field parameters

### 5.3 Examples

**Example 1: Clean Two Files**
```bash
AutoMapper.py . clean reactant.data product.data --coeff_file ff.in
```

**Example 2: Clean Multiple Files**
```bash
AutoMapper.py . clean file1.data file2.data file3.data file4.data --coeff_file settings.in
```

**Example 3: Different Directory**
```bash
AutoMapper.py /home/user/simulations clean pre.data post.data --coeff_file coeff.in
```

### 5.4 What Clean Does

**Step 1: Type Collection**
- Reads all data files
- Identifies unique atom types
- Extracts properties for each type

**Step 2: Type Unification**
- Creates mapping between old and new types
- Ensures same element = same type across files
- Assigns sequential type numbers

**Step 3: File Updates**
- Updates atom types in each file
- Renumbers types sequentially
- Saves with "cleaned" prefix

**Step 4: Coefficient Cleaning**
- Updates coefficient file with unified types
- Removes unused coefficients
- Maintains proper numbering

### 5.5 Output Files

**Cleaned Data Files:**
- Named: `cleaned[original_name]`
- Format: Standard LAMMPS data file
- Location: Same as input directory

**Cleaned Coefficient File:**
- Named: `cleaned[original_name]`
- Format: LAMMPS settings format
- Contains only used coefficients

### 5.6 Common Issues

**Issue: "pair_coeff uses wildcards (*)"**

Solution: Expand wildcards before using clean tool:
```
# Before
pair_coeff * * lj/cut 1.0 1.0

# After
pair_coeff 1 1 lj/cut 1.0 1.0
pair_coeff 1 2 lj/cut 1.0 1.0
pair_coeff 2 2 lj/cut 1.0 1.0
```

**Issue: "Different number of types in files"**

This is normal - clean tool handles it automatically by creating unified type system.

## 6. Molecule Tool

### 6.1 Purpose and Usage

The molecule tool converts LAMMPS data files to molecule format, which is required for `fix bond/react`.

### 6.2 Command Syntax

```bash
AutoMapper.py [directory] molecule [data_file] --save_name [output_name] [options]
```

**Required Arguments:**
- `data_file`: Input LAMMPS data file
- `--save_name`: Output molecule file name

**Optional Arguments:**
- `--ba`: Bonding atom IDs (for marking special atoms)

### 6.3 Examples

**Example 1: Simple Conversion**
```bash
AutoMapper.py . molecule reactant.data --save_name reactant-molecule.data
```

**Example 2: With Bonding Atoms**
```bash
AutoMapper.py . molecule reactant.data --save_name reactant-mol.data --ba 1 4
```

**Example 3: Full Path**
```bash
AutoMapper.py /path/to/files molecule input.data --save_name output.mol
```

### 6.4 What Molecule Does

**Step 1: Parse Data File**
- Reads LAMMPS data file
- Extracts atoms, bonds, angles, dihedrals
- Identifies atom types and coordinates

**Step 2: Calculate Special Bonds**
- Computes 1-2 neighbors (bonded atoms)
- Computes 1-3 neighbors (angle atoms)
- Computes 1-4 neighbors (dihedral atoms)

**Step 3: Format Conversion**
- Converts to LAMMPS molecule format
- Separates coordinates and types
- Adds special bond information

**Step 4: File Output**
- Saves in molecule format
- Includes all necessary sections
- Ready for use with `fix bond/react`

### 6.5 Molecule File Format

A molecule file contains these sections:

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

Special Bond Counts

atomID n12 n13 n14
...

Special Bonds

atomID specialBond1 specialBond2 ...
...
```

### 6.6 Understanding Special Bonds

**1-2 Neighbors (n12):**
- Atoms directly bonded
- Example: In A-B-C, B has two 1-2 neighbors: A and C

**1-3 Neighbors (n13):**
- Atoms two bonds away
- Example: In A-B-C-D, B has one 1-3 neighbor: D

**1-4 Neighbors (n14):**
- Atoms three bonds away
- Example: In A-B-C-D-E, B has one 1-4 neighbor: E

**Why Special Bonds Matter:**
LAMMPS uses special bonds to exclude or scale non-bonded interactions between nearby atoms.

## 7. Map Tool

### 7.1 Purpose and Usage

The map tool is the most powerful AutoMapper tool. It:
1. Creates pre- and post-reaction molecule files
2. Maps atoms between structures
3. Generates minimal partial structures
4. Outputs equivalence map file

### 7.2 Command Syntax

```bash
AutoMapper.py [directory] map [pre_file] [post_file] \
    --save_name [pre_mol_name] [post_mol_name] \
    --ba [bonding_atoms...] \
    --ebt [elements...] \
    [optional_arguments]
```

**Required Arguments:**
- `pre_file`: Pre-reaction LAMMPS data file
- `post_file`: Post-reaction LAMMPS data file
- `--save_name`: Two output molecule file names
- `--ba`: Bonding atom IDs (min 4: 2 pre, 2 post)
- `--ebt`: Element symbols ordered by type number

**Optional Arguments:**
- `--da`: Delete atom IDs (atoms removed during reaction)
- `--ca`: Create atom IDs (atoms created during reaction)
- `--debug`: Enable detailed debug output

### 7.3 Examples

**Example 1: Simple Bond Formation**
```bash
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --ebt H C N O
```

**Example 2: With Atom Deletion**
```bash
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 4 2 5 \
    --da 10 0 11 0 \
    --ebt H C N O P S
```

**Example 3: With Atom Creation**
```bash
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --ca 0 15 0 16 \
    --ebt H C N O
```

**Example 4: Debug Mode**
```bash
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --ebt H C N O \
    --debug
```

### 7.4 Understanding Bonding Atoms (--ba)

Bonding atoms are atoms that form new bonds during the reaction.

**Format:** `--ba pre1 pre2 post1 post2`

- `pre1`, `pre2`: Atom IDs in pre-reaction structure that will bond
- `post1`, `post2`: Corresponding atom IDs in post-reaction structure

**Example:**
```bash
--ba 5 10 5 8
```
Means:
- Pre-reaction: atoms 5 and 10 will form a bond
- Post-reaction: atoms 5 and 8 are the same atoms (may be renumbered)

**Multiple Bonding Pairs:**
For reactions forming multiple bonds:
```bash
--ba 1 2 3 4 5 6 7 8
```
Means: atoms (1,2) and (5,6) form bonds in pre, become (3,4) and (7,8) in post

### 7.5 Understanding Elements by Type (--ebt)

Elements must be listed in order of atom type numbers.

**Example:**
If your data file has:
- Type 1 = Hydrogen
- Type 2 = Carbon
- Type 3 = Nitrogen
- Type 4 = Oxygen

Then use:
```bash
--ebt H C N O
```

**Finding Type Numbers:**
Look in your LAMMPS data file:
```
Masses

1 1.008      # Type 1 = Hydrogen
2 12.011     # Type 2 = Carbon
3 14.007     # Type 3 = Nitrogen
4 15.999     # Type 4 = Oxygen
```

### 7.6 Understanding Delete Atoms (--da)

Delete atoms are atoms that disappear during the reaction.

**Format:** `--da pre1 pre2 ... preN post1 post2 ... postN`

- First half: Pre-reaction atom IDs to delete
- Second half: Corresponding post-reaction IDs (use 0 if deleted)

**Example:**
```bash
--da 10 11 0 0
```
Means:
- Pre-reaction: atoms 10 and 11 will be deleted
- Post-reaction: use 0 (atoms don't exist)

**Why Even Length:**
You must specify the same number of pre and post IDs, even if atoms are deleted.

### 7.7 Understanding Create Atoms (--ca)

Create atoms are atoms that appear during the reaction.

**Format:** `--ca pre1 pre2 ... preN post1 post2 ... postN`

- First half: Pre-reaction IDs (use 0 if not present)
- Second half: Post-reaction atom IDs that are created

**Example:**
```bash
--ca 0 0 15 16
```
Means:
- Pre-reaction: use 0 (atoms don't exist yet)
- Post-reaction: atoms 15 and 16 are created

### 7.8 Map Tool Workflow

**Phase 1: Initial Molecule Creation**
1. Convert both data files to molecule format
2. Include all atoms (full structure)

**Phase 2: Atom Mapping**
1. Start from bonding atoms
2. Use BFS to map neighboring atoms
3. Match atoms by element and topology
4. Handle missing atoms with inference

**Phase 3: Partial Structure**
1. Detect cycles around bonding atoms
2. Preserve atoms within 4 bonds of bonding atoms
3. Include delete and create atoms
4. Find edge atoms (boundary)
5. Verify edge atoms don't change type
6. Add any byproduct molecules

**Phase 4: Output Generation**
1. Renumber atoms sequentially
2. Rebuild molecule files with partial structure
3. Create map file with equivalences

### 7.9 Output Files

**Pre-Reaction Molecule File:**
- Contains partial structure
- Atoms renumbered from 1
- Includes bonding and edge atoms

**Post-Reaction Molecule File:**
- Contains partial structure
- Matches pre-reaction structure size
- Includes bonding and edge atoms

**Map File (automap.data):**
Contains:
- Number of equivalences
- Bonding atom IDs
- Edge atom IDs (if any)
- Delete atom IDs (if any)
- Create atom IDs (if any)
- Full equivalence list

**Example automap.data:**
```
#This is an AutoMapper generated map
50 equivalences
2 edgeIDs

BondingIDs

1
4

EdgeIDs

23
24

Equivalences

1    1
2    2
3    3
...
50   50
```

### 7.10 Debug Mode

Enable with `--debug` flag:
```bash
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --ebt H C N O \
    --debug
```

**Debug Output Includes:**
- Detailed mapping steps
- Fingerprint matching attempts
- Cycle detection results
- Queue operations
- Missing atom resolution
- Inference warnings

**Example Debug Output:**
```
DEBUG: Pre: 1, Post: 1 found with user specified bond atom
DEBUG: Pre: 2, Post: 3 found with single element occurence
DEBUG: Pre: 5, Post: 7 found with firstNeighbourElements
DEBUG: Pre: 8, Post: 10 found with secondNeighbourElements
DEBUG: Cycle found: ['1', '2', '3', '4', '5', '6', '1']. Started with 2 for the Pre-bond reaction.
Note: Pre-bond atomID 15 has been assigned by inference to post-bond atomID 18...
```

## 8. Advanced Usage

### 8.1 Complex Reactions

**Multiple Bond Formation:**
```bash
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 5 6 7 8 \
    --ebt H C N O
```

**Ring Opening Reaction:**
```bash
# Pre-reaction has cycle, post doesn't
AutoMapper.py . map cyclic-pre.data linear-post.data \
    --save_name pre.mol post.mol \
    --ba 1 4 1 4 \
    --ebt H C N O \
    --debug
```

**Ring Closing Reaction:**
```bash
# Pre-reaction is linear, post has cycle
AutoMapper.py . map linear-pre.data cyclic-post.data \
    --save_name pre.mol post.mol \
    --ba 5 10 5 10 \
    --ebt H C N O
```

### 8.2 Working with Large Molecules

**For molecules with >1000 atoms:**

1. **Use clean tool first** to ensure consistency
2. **Verify bonding atoms** are correctly identified
3. **Check element list** matches all types
4. **Monitor memory usage** during mapping
5. **Use debug mode** to track progress

**Performance Tips:**
- Partial structure minimization helps automatically
- Ring detection may take longer for large rings
- Most time spent in BFS mapping phase

### 8.3 Handling Symmetric Molecules

Symmetric molecules can confuse the mapping algorithm.

**Issue:** Multiple atoms have identical environments

**Solution 1: Use Debug Mode**
```bash
--debug
```
Check inference messages for problematic atoms.

**Solution 2: Verify Element List**
Ensure element types are specific enough.

**Solution 3: Manual Verification**
After mapping, visually verify the map file is correct.

**Example Symmetric Case:**
```
Benzene ring: All carbons equivalent
AutoMapper uses inference: may map arbitrarily
Verify: Check that mapped pairs make chemical sense
```

### 8.4 Custom Partial Structures

By default, AutoMapper creates minimal partial structures automatically. To customize:

**Modify Distance Threshold:**
Edit `MapProcessor.py`:
```python
# Line ~98
maxDistance = 4  # Change to 3 for smaller, 5 for larger
```

**Force Full Structure:**
Comment out partial structure creation:
```python
# In map_processor function
# Comment out lines creating validIDSet
validIDSet = None  # This forces full structure
```

### 8.5 Batch Processing

**Process Multiple Reactions:**
```bash
#!/bin/bash
# batch_automapper.sh

reactions=("rxn1" "rxn2" "rxn3")

for rxn in "${reactions[@]}"; do
    echo "Processing $rxn..."
    AutoMapper.py . map ${rxn}_pre.data ${rxn}_post.data \
        --save_name ${rxn}_pre.mol ${rxn}_post.mol \
        --ba 1 2 3 4 \
        --ebt H C N O
done
```

**Parallel Processing:**
```bash
#!/bin/bash
# parallel_automapper.sh

parallel AutoMapper.py . map {}_pre.data {}_post.data \
    --save_name {}_pre.mol {}_post.mol \
    --ba 1 2 3 4 \
    --ebt H C N O ::: rxn1 rxn2 rxn3 rxn4
```

## 9. File Formats

### 9.1 LAMMPS Data File Format

**Required Sections:**
```
# Comment

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
...

Atoms

atomID molID type charge x y z
...

Bonds

bondID type atomID1 atomID2
...
```

**Optional Sections:**
- Angles
- Dihedrals
- Impropers

### 9.2 LAMMPS Molecule File Format

**Format Created by AutoMapper:**
```
# Molecule file

N atoms
M bonds
P angles
Q dihedrals

Coords

atomID x y z
...

Types

atomID type
...

Charges

atomID charge
...

Bonds

bondID type atomID1 atomID2
...

Special Bond Counts

atomID n12 n13 n14
...

Special Bonds

atomID id1 id2 ... idn
...
```

### 9.3 Map File Format

**AutoMapper Map File:**
```
#This is an AutoMapper generated map
N equivalences
[D deleteIDs]
[E edgeIDs]
[C createIDs]

BondingIDs

bondID1
...

[DeleteIDs

delID1
...]

[EdgeIDs

edgeID1
...]

[CreateIDs

createID1
...]

Equivalences

preID1    postID1
preID2    postID2
...
```

**Sections:**
- `[]` indicates optional section
- Equivalences section required
- BondingIDs section required

### 9.4 Coefficient File Format

**LAMMPS Settings Format:**
```
# Force field parameters

pair_coeff 1 1 lj/cut 0.0152 2.42
pair_coeff 1 2 lj/cut 0.0114 2.81
...

bond_coeff 1 harmonic 350.0 1.09
bond_coeff 2 harmonic 310.0 1.54
...

angle_coeff 1 harmonic 35.0 109.5
...

dihedral_coeff 1 opls 1.3 -0.05 0.2 0.0
...
```

## 10. Troubleshooting

### 10.1 Common Errors

**Error: "This script is not compatible with Python 2"**
- **Cause:** Using Python 2 instead of Python 3
- **Solution:** Use `python3` command or upgrade Python

**Error: "natsort was not found"**
- **Cause:** natsort package not installed
- **Solution:** `pip3 install natsort`

**Error: "clean tool requires --coeff_file"**
- **Cause:** Missing required argument
- **Solution:** Add `--coeff_file filename.in`

**Error: "map tool requires 2 data_files and 2 save_names"**
- **Cause:** Wrong number of files specified
- **Solution:** Provide exactly 2 input files and 2 output names

**Error: "map tool requires --ba and --ebt arguments"**
- **Cause:** Missing required arguments
- **Solution:** Add bonding atoms and elements:
  ```bash
  --ba 1 2 3 4 --ebt H C N O
  ```

### 10.2 Mapping Issues

**Issue: "Could not find the symmetric pair for preAtom X"**
- **Meaning:** AutoMapper couldn't uniquely identify matching atom
- **Impact:** Atom may be incorrectly mapped or missing
- **Solution:**
  1. Enable debug mode to see more details
  2. Check if molecule is highly symmetric
  3. Verify element list is correct
  4. Manually verify map file after creation

**Issue: "Note: Pre-bond atomID X has been assigned by inference..."**
- **Meaning:** AutoMapper used fallback matching
- **Impact:** Mapping may be ambiguous
- **Solution:**
  1. Review the suggested alternatives
  2. Visually verify the mapping makes sense
  3. If incorrect, you may need to manually edit map file

**Issue: Missing atoms in mapping**
- **Meaning:** Some atoms not matched
- **Possible causes:**
  1. Element list incorrect
  2. Topology too different between pre and post
  3. Actual chemical change occurred (expected)
- **Solution:**
  1. Verify element list matches types
  2. Check that pre/post represent same molecule
  3. Use --da or --ca if atoms deleted/created

### 10.3 File Format Issues

**Issue: "Section name not found in file"**
- **Cause:** LAMMPS file missing required section
- **Solution:** Ensure file has all required sections (Atoms, Bonds, etc.)

**Issue: "Invalid LAMMPS file format"**
- **Cause:** File structure doesn't match expected format
- **Solution:**
  1. Check section headers are single words
  2. Verify data line formatting
  3. Ensure blank lines between sections

**Issue: "Duplicate atom IDs found"**
- **Cause:** Same atom ID appears multiple times
- **Solution:** Fix input file to have unique atom IDs

### 10.4 Performance Issues

**Issue: AutoMapper runs very slowly**
- **Causes:**
  1. Very large molecule (>10,000 atoms)
  2. Highly symmetric structure
  3. Many rings to detect
- **Solutions:**
  1. Be patient - large molecules take time
  2. Consider breaking into smaller reactions
  3. Use faster computer if available

**Issue: Running out of memory**
- **Cause:** Molecule too large for available RAM
- **Solution:**
  1. Close other applications
  2. Use computer with more RAM
  3. Consider simplifying molecule

### 10.5 Output Issues

**Issue: Partial structure too large/small**
- **Cause:** Default distance threshold may not be optimal
- **Solution:** Modify distance threshold in code (see Advanced Usage)

**Issue: Edge atoms seem incorrect**
- **Cause:** Complex molecular structure or algorithm limitation
- **Solution:**
  1. Review edge atoms in map file
  2. Verify they don't change type
  3. Manually adjust if necessary

**Issue: Map file missing expected sections**
- **Cause:** No delete/create/edge atoms in this reaction
- **Solution:** This is normal - only relevant sections included

### 10.6 LAMMPS Integration Issues

**Issue: LAMMPS doesn't read map file**
- **Cause:** Format incompatibility with LAMMPS version
- **Solution:**
  1. Check LAMMPS version (29Oct20 or later recommended)
  2. Verify map file format matches LAMMPS expectations
  3. Try renaming "BondingIDs" to "Initiator IDs" for newer LAMMPS

**Issue: LAMMPS reaction doesn't occur**
- **Cause:** Incorrect map or molecule files
- **Solution:**
  1. Verify bonding atoms are correct
  2. Check cutoff distances in LAMMPS input
  3. Visualize structures to verify correctness

## 11. Examples and Workflows

### 11.1 Example 1: Simple C-C Bond Formation

**Scenario:** Two ethyl radicals forming butane

**Files:**
- `ethyl_radical.data` - Single ethyl radical structure
- After: Manually create or use MD to get two ethyl radicals in proximity

**Step 1: Prepare Files**
```bash
# Assume you have:
# - pre_ethyl.data (two ethyl radicals)
# - post_butane.data (butane)
```

**Step 2: Clean Files** (if needed)
```bash
AutoMapper.py . clean pre_ethyl.data post_butane.data --coeff_file ff.in
```

**Step 3: Identify Key Atoms**
Visualize structures and identify:
- Pre-reaction: C atoms that will bond (e.g., atoms 3 and 8)
- Post-reaction: Same C atoms (may be renumbered, e.g., atoms 3 and 7)

**Step 4: Determine Element List**
Look at Masses section:
```
1 1.008   # H
2 12.011  # C
```
So: `--ebt H C`

**Step 5: Run Map Tool**
```bash
AutoMapper.py . map cleanedpre_ethyl.data cleanedpost_butane.data \
    --save_name pre_rxn.mol post_rxn.mol \
    --ba 3 8 3 7 \
    --ebt H C
```

**Step 6: Verify Output**
Check created files:
- `pre_rxn.mol` - Pre-reaction molecule
- `post_rxn.mol` - Post-reaction molecule  
- `automap.data` - Reaction map

**Step 7: Use in LAMMPS**
```lammps
# In LAMMPS input script
molecule pre_mol pre_rxn.mol
molecule post_mol post_rxn.mol

fix rxn all bond/react react rxn1 all 1 5.0 pre_mol post_mol automap.data
```

### 11.2 Example 2: Ester Hydrolysis with Atom Deletion

**Scenario:** Ester + Water → Carboxylic Acid + Alcohol

**Key Features:**
- Bond breaking
- Atom deletion (from ester)
- Complex mapping

**Step 1: Prepare Structures**
```bash
# Starting files:
# - ester_water_pre.data (ester + water nearby)
# - acid_alcohol_post.data (products)
```

**Step 2: Identify Features**

Bonding atoms (for new bonds in products):
- Pre: O in water (atom 15), C in acid (atom 5)
- Post: O in alcohol (atom 12), C in acid (atom 5)

Delete atoms (O-C bond breaks in ester):
- Pre: O in ester (atom 20)
- Post: Deleted (use 0)

**Step 3: Run with Delete Atoms**
```bash
AutoMapper.py . map ester_water_pre.data acid_alcohol_post.data \
    --save_name pre.mol post.mol \
    --ba 15 5 12 5 \
    --da 20 0 \
    --ebt H C O
```

**Step 4: Verify**
Check automap.data has DeleteIDs section:
```
1 deleteIDs

DeleteIDs

20

Equivalences
...
```

### 11.3 Example 3: Ring Opening Polymerization

**Scenario:** Epoxide ring opening

**Key Features:**
- Cyclic structure opens
- Ring detection important
- Partial structure preserves ring

**Step 1: Prepare**
```bash
# Files:
# - cyclic_epoxide.data (closed ring)
# - open_epoxide.data (opened ring)
```

**Step 2: Enable Debug**
```bash
AutoMapper.py . map cyclic_epoxide.data open_epoxide.data \
    --save_name pre.mol post.mol \
    --ba 3 5 3 5 \
    --ebt H C O \
    --debug
```

**Step 3: Check Debug Output**
Look for:
```
DEBUG: Cycle found: ['1', '2', '3', '4', '5', '1']. Started with 3 for the Pre-bond reaction.
DEBUG: Creating a partial map.
```

**Step 4: Verify Ring Preserved**
In pre.mol, all ring atoms should be present even in partial structure.

### 11.4 Example 4: Complete Workflow with Moltemplate

**Scenario:** Full workflow from Moltemplate to LAMMPS

**Step 1: Create Moltemplate Files**
```bash
# reactant.lt (Moltemplate file)
# product.lt (Moltemplate file)
```

**Step 2: Generate LAMMPS Files**
```bash
moltemplate.sh reactant.lt
moltemplate.sh product.lt
```

Creates:
- `reactant.data`
- `reactant.in.settings`
- `product.data`
- `product.in.settings`

**Step 3: Clean Files**
```bash
AutoMapper.py . clean reactant.data product.data \
    --coeff_file reactant.in.settings
```

**Step 4: Create Map**
```bash
AutoMapper.py . map cleanedreactant.data cleanedproduct.data \
    --save_name pre.mol post.mol \
    --ba 10 15 10 15 \
    --ebt H C N O S
```

**Step 5: Setup LAMMPS Simulation**
```lammps
# simulation.in

# Read initial structure
read_data cleanedreactant.data

# Include force field
include cleanedreactant.in.settings

# Define molecules
molecule pre_mol pre.mol
molecule post_mol post.mol

# Setup reaction
fix rxn all bond/react react rxn1 all 1 5.0 pre_mol post_mol automap.data

# Run simulation
run 1000000
```

**Step 6: Run LAMMPS**
```bash
lammps -in simulation.in
```

### 11.5 Example 5: Multiple Reaction Sites

**Scenario:** Polymer with multiple crosslinking sites

**Approach:** Create separate map for each reaction type

**Step 1: Identify Reaction Types**
- Type A: Epoxy-amine reaction
- Type B: Thiol-ene reaction

**Step 2: Create Map for Type A**
```bash
AutoMapper.py . map pre_typeA.data post_typeA.data \
    --save_name preA.mol postA.mol \
    --ba 5 10 5 10 \
    --ebt H C N O \
    --debug
```

Creates: `automap.data` → rename to `rxnA_map.data`

**Step 3: Create Map for Type B**
```bash
AutoMapper.py . map pre_typeB.data post_typeB.data \
    --save_name preB.mol postB.mol \
    --ba 7 12 7 12 \
    --ebt H C N O S \
    --debug
```

Creates: `automap.data` → rename to `rxnB_map.data`

**Step 4: Use Both in LAMMPS**
```lammps
molecule preA_mol preA.mol
molecule postA_mol postA.mol
molecule preB_mol preB.mol
molecule postB_mol postB.mol

fix rxn all bond/react &
    react rxnA all 1 5.0 preA_mol postA_mol rxnA_map.data &
    react rxnB all 1 6.0 preB_mol postB_mol rxnB_map.data
```

## 12. FAQ

### 12.1 General Questions

**Q: Do I need to use all three tools?**
A: No. The most common workflow is: clean → map. The molecule tool is optional.

**Q: Can I use AutoMapper with force fields other than OPLS?**
A: Yes! AutoMapper is completely force field independent.

**Q: Does AutoMapper work with implicit solvent?**
A: Yes, as long as your LAMMPS files are in the correct format.

**Q: Can I visualize the output files?**
A: Yes, molecule files can be visualized with VMD, OVITO, or similar tools.

### 12.2 Technical Questions

**Q: What is the difference between bonding atoms and delete atoms?**
A: Bonding atoms form new bonds during reaction. Delete atoms are removed completely.

**Q: Why do I need to specify elements by type?**
A: AutoMapper uses element information for atom matching. Type numbers alone aren't enough.

**Q: What is a partial structure?**
A: A minimal subset of atoms required for the reaction, reducing computational cost.

**Q: Can I disable partial structure creation?**
A: Not directly from command line, but you can modify the code (see Advanced Usage).

**Q: What does "edge atom" mean?**
A: Atoms at the boundary of the partial structure, where it's "cut" from the full structure.

### 12.3 Mapping Questions

**Q: How does AutoMapper match atoms?**
A: Uses graph algorithms based on chemical element, connectivity, and local topology.

**Q: What if mapping fails?**
A: Check element list, verify files represent same molecule, and use debug mode.

**Q: Can AutoMapper handle stereochemistry?**
A: Not explicitly. Chiral centers may be mapped ambiguously in symmetric molecules.

**Q: What is "fingerprint matching"?**
A: A technique comparing atoms based on their neighbor elements up to 3 bonds away.

**Q: What does "inference" mean in debug output?**
A: AutoMapper couldn't uniquely determine match, so it made an educated guess.

### 12.4 Performance Questions

**Q: How large a molecule can AutoMapper handle?**
A: Tested up to ~10,000 atoms. Larger molecules may work but will be slower.

**Q: Why is mapping slow for my molecule?**
A: Likely due to size, symmetry, or many rings. Be patient or simplify structure.

**Q: Can I parallelize AutoMapper?**
A: Not internally, but you can process multiple reactions in parallel (see Batch Processing).

### 12.5 Integration Questions

**Q: What LAMMPS version do I need?**
A: LAMMPS 29Oct20 or later. Newer versions (29Sept21+) have additional features.

**Q: Does AutoMapper work with Moltemplate?**
A: Yes! Designed to work well with Moltemplate output.

**Q: Can I use AutoMapper with GROMACS?**
A: No, AutoMapper is specifically for LAMMPS.

**Q: How do I use the output in LAMMPS?**
A: Use `molecule` and `fix bond/react` commands (see examples).

## 13. Best Practices

### 13.1 File Preparation

**Do:**
- Use consistent atom numbering
- Ensure all required sections present
- Check force field compatibility
- Visualize structures before mapping

**Don't:**
- Mix different force fields
- Use duplicate atom IDs
- Forget required sections
- Skip the clean tool

### 13.2 Mapping Strategy

**Do:**
- Start with clean tool
- Verify bonding atoms are correct
- Double-check element list
- Use debug mode for complex cases
- Validate output files

**Don't:**
- Skip verification steps
- Ignore warning messages
- Assume inference is always correct
- Forget about delete/create atoms

### 13.3 Troubleshooting Strategy

**When things don't work:**

1. **Read the error message** - Often tells you exactly what's wrong
2. **Enable debug mode** - Get detailed information
3. **Check your inputs** - Verify files, arguments, element list
4. **Visualize structures** - Confirm they represent what you think
5. **Start simple** - Test with smaller molecule first
6. **Ask for help** - Open an issue on GitHub

### 13.4 Workflow Optimization

**For routine use:**

1. Create shell scripts for common reactions
2. Use consistent naming conventions
3. Keep reaction files organized
4. Document your workflow
5. Version control your input files

**Example script template:**
```bash
#!/bin/bash
# rxn_template.sh

REACTION=$1
PRE="${REACTION}_pre.data"
POST="${REACTION}_post.data"

echo "Processing reaction: $REACTION"

# Clean
AutoMapper.py . clean $PRE $POST --coeff_file ff.in

# Map
AutoMapper.py . map cleaned$PRE cleaned$POST \
    --save_name ${REACTION}_pre.mol ${REACTION}_post.mol \
    --ba $2 $3 $4 $5 \
    --ebt H C N O

echo "Complete: ${REACTION}_pre.mol, ${REACTION}_post.mol, automap.data"
```

Usage:
```bash
./rxn_template.sh epoxy_cure 5 10 5 10
```

## 14. Integration with LAMMPS

### 14.1 Using AutoMapper Output

**Basic LAMMPS Setup:**
```lammps
# Read initial configuration
read_data initial.data

# Define reaction molecules
molecule pre_mol pre-molecule.data
molecule post_mol post-molecule.data

# Setup reaction
fix rxn all bond/react react rxn1 all 1 5.0 pre_mol post_mol automap.data

# Run simulation
timestep 1.0
run 1000000
```

### 14.2 Fix bond/react Options

**Common options:**
```lammps
fix rxn all bond/react &
    stabilization yes statted_grp .03 &
    react rxn1 all 1 5.0 pre_mol post_mol automap.data &
    prob 1.0 12345
```

**Option explanations:**
- `stabilization yes`: Gradually introduce new atoms
- `statted_grp .03`: Stabilization group with keyword
- `prob 1.0`: Reaction probability
- `12345`: Random seed

### 14.3 Monitoring Reactions

**Track reaction events:**
```lammps
compute rxn_count all bond/react reaction_count rxn1
thermo_style custom step temp press c_rxn_count
thermo 1000
```

**Dump reaction bonds:**
```lammps
dump 1 all custom 10000 dump.rxn id type x y z
dump_modify 1 sort id
```

### 14.4 Multiple Reactions

**Simultaneous reactions:**
```lammps
fix rxn all bond/react &
    react rxn1 all 1 5.0 pre1_mol post1_mol map1.data &
    react rxn2 all 1 6.0 pre2_mol post2_mol map2.data &
    react rxn3 all 1 4.5 pre3_mol post3_mol map3.data
```

### 14.5 Common LAMMPS Issues

**Issue: "Cannot find molecule file"**
- Ensure molecule files in working directory
- Use absolute paths if needed

**Issue: "Invalid map file format"**
- Check AutoMapper version compatibility
- Verify map file sections are correct

**Issue: "No reactions occurring"**
- Increase cutoff distance
- Check bonding atom proximity
- Verify reaction probability

### 14.6 Performance Tips

**For large systems:**
- Use neighbor list updates carefully
- Limit reaction check frequency
- Consider domain decomposition
- Use efficient force fields

**Example:**
```lammps
neighbor 2.0 bin
neigh_modify every 1 delay 5 check yes

fix rxn all bond/react &
    reset_mol_ids no &
    react rxn1 all 1 5.0 pre_mol post_mol automap.data
```

---

## Appendix A: Quick Reference Card

### Common Commands

```bash
# Clean
AutoMapper.py . clean pre.data post.data --coeff_file ff.in

# Molecule
AutoMapper.py . molecule data.data --save_name mol.data

# Map (basic)
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --ebt H C N O

# Map (with delete)
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --da 5 0 \
    --ebt H C N O

# Map (with debug)
AutoMapper.py . map pre.data post.data \
    --save_name pre.mol post.mol \
    --ba 1 2 3 4 \
    --ebt H C N O \
    --debug
```

### Argument Summary

| Argument | Tool | Required | Description |
|----------|------|----------|-------------|
| directory | All | Yes | Working directory |
| tool | All | Yes | clean, molecule, or map |
| data_files | All | Yes | Input file(s) |
| --coeff_file | clean | Yes | Coefficient file |
| --save_name | molecule, map | Yes | Output name(s) |
| --ba | map | Yes | Bonding atoms |
| --ebt | map | Yes | Elements by type |
| --da | map | No | Delete atoms |
| --ca | map | No | Create atoms |
| --debug | map | No | Debug output |

## Appendix B: Error Messages Reference

| Error | Meaning | Solution |
|-------|---------|----------|
| "Python 2 not compatible" | Using Python 2 | Use python3 |
| "natsort not found" | Missing package | pip install natsort |
| "requires --coeff_file" | Missing argument | Add --coeff_file |
| "requires 2 data_files" | Wrong file count | Provide 2 files |
| "requires --ba and --ebt" | Missing arguments | Add --ba and --ebt |

## Appendix C: File Format Templates

### LAMMPS Data File Template
```
# Molecule Name

10 atoms
9 bonds
8 angles
7 dihedrals

2 atom types
1 bond types
1 angle types
1 dihedral types

0.0 20.0 xlo xhi
0.0 20.0 ylo yhi
0.0 20.0 zlo zhi

Masses

1 1.008
2 12.011

Atoms

1 1 1 0.0 0.0 0.0 0.0
2 1 2 0.0 1.0 0.0 0.0
...

Bonds

1 1 1 2
...
```

### Map File Template
```
#This is an AutoMapper generated map
N equivalences

BondingIDs

1
2

Equivalences

1    1
2    2
...
```

## Appendix D: Glossary

- **Atom Type**: Numeric identifier for force field parameters
- **Bonding Atoms**: Atoms forming new bonds during reaction
- **BFS**: Breadth-First Search algorithm
- **Create Atoms**: Atoms appearing during reaction
- **Delete Atoms**: Atoms removed during reaction
- **Edge Atoms**: Boundary atoms of partial structure
- **Element Symbol**: Chemical element (H, C, N, O, etc.)
- **Equivalence**: Mapping between pre and post atom IDs
- **Fingerprint**: Topological identifier for atom matching
- **Inference**: Educated guess when unique match impossible
- **Molecule File**: LAMMPS format for fix bond/react
- **Partial Structure**: Minimal subset for reaction
- **Special Bonds**: 1-2, 1-3, 1-4 neighbor relationships

---

**End of User Manual**

For additional support, visit:
- GitHub Repository: https://github.com/nobkt/AutoMapper
- Issue Tracker: https://github.com/nobkt/AutoMapper/issues
- LAMMPS Documentation: https://docs.lammps.org
