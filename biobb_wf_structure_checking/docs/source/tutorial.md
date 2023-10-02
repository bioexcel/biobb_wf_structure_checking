# Molecular Structure Checking using BioExcel Building Blocks (biobb)

***
This tutorial aims to illustrate the process of **checking** a **molecular structure** before using it as an input for a **Molecular Dynamics** simulation. The workflow uses the **BioExcel Building Blocks library (biobb)**. The particular structure used is the crystal structure of **human Adenylate Kinase 1A (AK1A)**, in complex with the **AP5A inhibitor** (PDB code [1Z83](https://www.rcsb.org/structure/1z83)).  

**Structure checking** is a key step before setting up a protein system for **simulations**. A number of **common issues** found in structures at **Protein Data Bank** may compromise the success of the **simulation**, or may suggest that longer **equilibration** procedures are necessary.

The **workflow** shows how to:

- Run **basic manipulations on structures** (selection of models, chains, alternative locations
- Detect and fix **amide assignments** and **wrong chiralities**
- Detect and fix **protein backbone** issues (missing fragments, and atoms, capping)
- Detect and fix **missing side-chain atoms**
- **Add hydrogen atoms** according to several criteria
- Detect and classify **atomic clashes**
- Detect possible **disulfide bonds (SS)**

***NOTE***: 
- Some of the workflow steps work with **AMBER force-field nomenclature** (e.g. CYX, HID, HIE), and thus the output structure might be slightly modified to use it with different **force-fields**. 

An implementation of this workflow in a **web-based Graphical User Interface (GUI)** can be found in the https://mmb.irbbarcelona.org/biobb-wfs/ server (see https://mmb.irbbarcelona.org/biobb-wfs/help/create/structure#check).

***

## Settings

### Biobb modules used

 - [biobb_io](https://github.com/bioexcel/biobb_io): Tools to fetch biomolecular data from public databases.
 - [biobb_structure_utils](https://github.com/bioexcel/biobb_structure_utils): Tools to modify or extract information from a PDB structure.
 - [biobb_model](https://github.com/bioexcel/biobb_model): Tools to check and model 3D structures, create mutations or reconstruct missing atoms.
 - [biobb_amber](https://github.com/bioexcel/biobb_amber): Tools to setup and simulate atomistic MD simulations using AMBER MD package.
 - [biobb_chemistry](https://github.com/bioexcel/biobb_chemistry): Tools to perform chemoinformatics on molecular structures.
  
### Auxiliar libraries used

* [modeller](https://salilab.org/modeller/): Software used for homology or comparative modeling of protein three-dimensional structures.
* [jupyter](https://jupyter.org/): Free software, open standards, and web services for interactive computing across all programming languages.
* [nglview](http://nglviewer.org/#nglview): Jupyter/IPython widget to interactively view molecular structures and trajectories in notebooks.
* [plotly](https://plot.ly/python/offline/): Python interactive graphing library integrated in Jupyter notebooks.

### Conda Installation and Launch

```console
git clone https://github.com/bioexcel/biobb_wf_structure_checking.git
cd biobb_wf_structure_checking
conda env create -f conda_env/environment.yml
conda activate biobb_wf_structure_checking
jupyter-notebook biobb_wf_structure_checking/notebooks/biobb_wf_structure_checking.ipynb
```

***
## Pipeline steps
 1. [Input Parameters](#input)
 2. [Fetching PDB structure](#fetch)
 3. [PDB structure checking](#PDBchecking)
     1. [Models](#models)
     2. [Chains](#chains)
     3. [Alternative Locations](#altlocs) 
     4. [Disulfide Bridges](#ss)
     5. [Metal Ions](#metals)
     6. [Ligands](#ligands)
     7. [Hydrogen Atoms](#hydrogens)
     8. [Water Molecules](#water)
     9. [Amide Groups](#amide)
     10. [Chirality](#chirality)
     11. [Missing Side Chain Atoms](#sidechains)
     12. [Missing Backbone Atoms](#backbone)     
     13. [Atomic Clashes](#clashes)
     14. [Final Check](#finalcheck)
 8. [Questions & Comments](#questions)
 
***
<img src="https://bioexcel.eu/wp-content/uploads/2019/04/Bioexcell_logo_1080px_transp.png" alt="Bioexcel2 logo"
	title="Bioexcel2 logo" width="400" />
***

<a id="input"></a>
## Input parameters
**Input parameters** needed:
 - **pdbCode**: PDB code of the protein structure (e.g. 6M0J)


```python
import nglview
import ipywidgets

pdbCode = "1z83" # Should be replaced to check different structures 
```

<a id="fetch"></a>
***
## Fetching PDB structure
Downloading **PDB structure** with the **protein molecule** from the RCSB PDB database.<br>
Alternatively, a **PDB file** can be used as starting structure. <br>
***
**Building Blocks** used:
 - [Pdb](https://biobb-io.readthedocs.io/en/latest/api.html#module-api.pdb) from **biobb_io.api.pdb**
***


```python
# Downloading desired PDB file 
# Import module
from biobb_io.api.pdb import pdb

# Create properties dict and inputs/outputs
downloaded_pdb = pdbCode+'.pdb'
prop = {
    'pdb_code': pdbCode,
    'filter': ['ATOM', 'HETATM'],
}

#Create and launch bb
pdb(output_pdb_path=downloaded_pdb,
    properties=prop)
```

<a id="vis3D"></a>
### Visualizing 3D structure
Visualizing the downloaded/given **PDB structure** using **NGL**: 


```python
# Show original structure
view = nglview.show_structure_file(downloaded_pdb)
view.add_representation(repr_type='spacefill', selection='ion')
view.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view._remote_call('setSize', target='Widget', args=['','400px'])
view
```

<a id="PDBchecking"></a>
***
## PDB structure checking
The first step of the workflow is a complete **checking** of the **molecular structure**. The **BioBB** utility **structure_check** parses the whole **PDB file**, runs a complete **structure checking**, saves **useful information** about the structure and looks for **possible issues**. The block is printing a **report** of the information extracted from the **input structure**. 
***
**Building Blocks** used:
 - [structure_check](https://biobb-structure-utils.readthedocs.io/en/latest/utils.html#module-utils.structure_check) from **biobb_structure_utils.utils.structure_check**
***


```python
from biobb_structure_utils.utils.structure_check import structure_check

report = pdbCode + ".report.json"

structure_check(
    input_structure_path=downloaded_pdb,
    output_summary_path=report
)
```

<a id="StepByStep"></a>
***
## Structure checking, step by step
The **workflow** will go through the **report**, exploring all the different **features** analysed. The following cell is showing the **full content** of the **report**.
***


```python
import json
with open(report, 'r') as f:
  data = json.load(f)
print(json.dumps(data, indent=2))
```

<a id="models"></a>
##  Models
Presence of **models** in the structure. MD simulations require a single structure, although some structures (e.g. biounits) may be defined as a series of models, in such case all of them are usually required.
***
Use **BioBB extract_model** to select the desired **models**. 

 - [extract_model](https://biobb-structure-utils.readthedocs.io/en/latest/utils.html#module-utils.extract_model) from **biobb_structure_utils.utils.extract_model**
***


```python
print(json.dumps(data['models'], indent=2))
```


```python
from biobb_structure_utils.utils.extract_model import extract_model

pdb_models = pdbCode + ".models.pdb"

prop = {
    'models': [ 1 ]
}

extract_model(
    input_structure_path=downloaded_pdb,
    output_structure_path=pdb_models,
    properties=prop
)
```


```python
# Show modified structure
view = nglview.show_structure_file(pdb_models)
view.add_representation(repr_type='spacefill', selection='ion')
view.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view._remote_call('setSize', target='Widget', args=['','400px'])
view
```

<a id="chains"></a>
##  Chains
Presence of **chains** in the structure. MD simulations are usually performed with complete structures. However input structure may contain **several copies of the system**, or contain **additional chains** like **peptides** or **nucleic acids** that may be removed. 
***
Use **BioBB extract_chain** to select the desired **chains**. 

 - [extract_chain](https://biobb-structure-utils.readthedocs.io/en/latest/utils.html#module-utils.extract_chain) from **biobb_structure_utils.utils.extract_chain**
***


```python
print(json.dumps(data['chains'], indent=2))
```


```python
from biobb_structure_utils.utils.extract_chain import extract_chain

pdb_chains = pdbCode + ".chains.pdb"

prop = {
    'chains': [ 'A' ]
}

extract_chain(
    input_structure_path=pdb_models,
    output_structure_path=pdb_chains,
    properties=prop
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_models)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1._remote_call('setSize', target='Widget', args=['','600px'])
view2 = nglview.show_structure_file(pdb_chains)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2._remote_call('setSize', target='Widget', args=['','600px'])
ipywidgets.HBox([view1, view2])
```

<a id="altlocs"></a>
##  Alternative Locations
Presence of residues with **alternative locations**. Atoms with **alternative coordinates** and their **occupancy** are reported. MD simulations requires a **single position** for each atom. 
***
Use **BioBB fix_altloc** to select the desired **alternative locations**. 

 - [fix_altlocs](https://biobb-model.readthedocs.io/en/latest/model.html#module-model.fix_altlocs) from **biobb_model.model.fix_altlocs**
***


```python
print(json.dumps(data['altloc'], indent=2))
```


```python
from biobb_model.model.fix_altlocs import fix_altlocs

pdb_altloc = pdbCode + ".altloc.pdb"

prop = {
    'altlocs': ['A45:A', 'A67:A', 'A85:A'] 
}

fix_altlocs(
    input_pdb_path=pdb_chains,
    output_pdb_path=pdb_altloc,
    properties=prop
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_chains)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1.add_representation(repr_type='ball+stick', radius='0.4', selection='45%A 67%A 85%A')
view1.add_representation(repr_type='ball+stick', radius='0.2', selection='45%B 67%B 85%B')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_altloc)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2.add_representation(repr_type='ball+stick', radius='0.4', selection='45 67 85')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="ss"></a>
##  Disulfide Bridges
Presence of **disulfide bridges** (-S-S- bonds) based on distance criteria. **Disulfide bonds** are crucial for the **structure stability**, and MD simulations require those bonds to be correctly set. Most of the **MD packages** come with tools able to automatically identify -S-S- bonds and include them in the **system topology** parameters, but some of them need the residues involved in the bond to be **explicitly marked** (e.g. AMBER CYX residues). 
***
Use **BioBB fix_ss_bonds** to mark the **disulfide bridges**. 

 - [fix_ssbonds](https://biobb-model.readthedocs.io/en/latest/model.html#module-model.fix_ssbonds) from **biobb_model.model.fix_ssbonds**
***


```python
print(json.dumps(data['getss'], indent=2))
```


```python
from biobb_model.model.fix_ssbonds import fix_ssbonds

pdb_ssbonds = pdbCode + ".ssbonds.pdb"

fix_ssbonds(
    input_pdb_path=pdb_altloc,
    output_pdb_path=pdb_ssbonds
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_altloc)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1.add_representation(repr_type='ball+stick', radius='0.4', selection='CYS')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_ssbonds)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2.add_representation(repr_type='ball+stick', radius='0.2', selection='CYS')
view2.add_representation(repr_type='ball+stick', radius='0.4', selection='CYX')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="metals"></a>
##  Metal Ions
Presence of heteroatoms being **metal ions**. Only structural **metal ions** should be kept in MD simulations, as they require special **force field parameters**. In some cases, the **atomic coordination** between the **protein atoms** and the **metal ions** should be also specified. 
***
Use **BioBB remove_molecules** to remove the undesired **metal ions**. 

 - [remove_molecules](https://biobb-structure-utils.readthedocs.io/en/latest/utils.html#module-utils.remove_molecules) from **biobb_structure-utils.utils.remove_molecules**
***


```python
print(json.dumps(data['metals'], indent=2))
```


```python
from biobb_structure_utils.utils.remove_molecules import remove_molecules

pdb_metals = pdbCode + ".metals.pdb"

prop = {
    'molecules': [
        {
            'name': 'ZN'
        }
    ]
}
remove_molecules(input_structure_path=pdb_ssbonds,
                 output_molecules_path=pdb_metals,
                 properties=prop)

```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_chains)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1.add_representation(repr_type='ball+stick', radius='0.4', selection='CYS')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_metals)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2.add_representation(repr_type='ball+stick', radius='0.2', selection='CYS')
view2.add_representation(repr_type='ball+stick', radius='0.4', selection='CYX')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="ligands"></a>
##  Ligands
Presence of heteroatoms being **ligands**. Only important **ligands** should be kept in MD simulations, as they require special **force field parameters**. 
***
Use **BioBB remove_molecules** to remove the undesired **ligands**. 

 - [remove_molecules](https://biobb-structure-utils.readthedocs.io/en/latest/utils.html#module-utils.remove_molecules) from **biobb_structure-utils.utils.remove_molecules**
***


```python
print(json.dumps(data['ligands'], indent=2))
```


```python
from biobb_structure_utils.utils.remove_molecules import remove_molecules

pdb_ligands = pdbCode + ".ligands.pdb"

prop = {
    'molecules': [
        {
            'name': 'SO4',
        },
        {
            'name' : 'AP5'
        }
    ]
}
remove_molecules(input_structure_path=pdb_metals,
                 output_molecules_path=pdb_ligands,
                 properties=prop)

```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_metals)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_ligands)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="hydrogens"></a>
##  Hydrogen atoms
Presence of **hydrogen atoms**. MD setup can be done with the original **hydrogen atoms**, however to prevent possible problems coming from non **standard labelling**, removing them is safer. 
***
Use **BioBB reduce_remove_hydrogens** to remove the undesired **hydrogen atoms**. 

 - [reduce_remove_hydrogens](https://biobb-chemistry.readthedocs.io/en/latest/ambertools.html#module-ambertools.reduce_remove_hydrogens) from **biobb_chemistry.ambertools.reduce_remove_hydrogens**
***


```python
print(json.dumps(data['stats']['stats'], indent=2))
```


```python
from biobb_chemistry.ambertools.reduce_remove_hydrogens import reduce_remove_hydrogens

pdb_hydrogens = pdbCode + ".hydrogens.pdb"

reduce_remove_hydrogens(
    input_path=pdb_ligands,
    output_path=pdb_hydrogens
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_ligands)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1.add_representation(repr_type='spacefill', selection='hydrogen')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_hydrogens)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2.add_representation(repr_type='spacefill', selection='hydrogen')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="water"></a>
##  Water molecules
Presence of **water molecules**. **Crystallographic water molecules** may be relevant for keeping the structure stable, however in most cases only some of them are required. These can be later added using other methods.
***
Use **BioBB remove_pdb_water** to remove the undesired **water molecules**. 

 - [remove_pdb_water](https://biobb-structure-utils.readthedocs.io/en/latest/utils.html#module-utils.remove_pdb_water) from **biobb_structure-utils.utils.remove_pdb_water**
***


```python
print(json.dumps(data['stats']['stats'], indent=2))
```


```python
from biobb_structure_utils.utils.remove_pdb_water import remove_pdb_water

pdb_water = pdbCode + ".water.pdb"

remove_pdb_water(
    input_pdb_path=pdb_hydrogens,
    output_pdb_path=pdb_water
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_hydrogens)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.5', selection='water')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_water)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.5', selection='water')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="amide"></a>
##  Amide groups
Presence of incorrect **amide groups**. **Amide terminal atoms** in ***Asparagines*** and ***Glutamines*** residues can be labelled incorrectly, with swapped **Nitrogen** and **Oxygen** atoms from the **amide group**. 
***
Use **BioBB fix_amides** to fix the incorrectly assigned **amide groups**. 

 - [fix_amides](https://biobb-model.readthedocs.io/en/latest/model.html#module-model.fix_amides) from **biobb_model.model.fix_amides**
***


```python
print(json.dumps(data['amide'], indent=2))
```


```python
from biobb_model.model.fix_amides import fix_amides

pdb_amides = pdbCode + ".amides.pdb"

fix_amides(
    input_pdb_path=pdb_water,
    output_pdb_path=pdb_amides
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_water)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1.add_representation(repr_type='ball+stick', radius='0.4', selection='amide')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_amides)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2.add_representation(repr_type='ball+stick', radius='0.4', selection='amide')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="chirality"></a>
##  Chirality
Presence of incorrect **chiralities**. Side chains of ***Threonine*** and ***Isoleucine*** residues are **chiral**. <br> **Incorrect atom labelling** can lead to the **wrong chirality**. 
***
Use **BioBB fix_chirality** to fix the incorrectly assigned **chirality**. 

 - [fix_chirality](https://biobb-model.readthedocs.io/en/latest/model.html#module-model.fix_chirality) from **biobb_model.model.fix_chirality**
***


```python
print(json.dumps(data['chiral'], indent=2))
```


```python
from biobb_model.model.fix_chirality import fix_chirality

pdb_chiral = pdbCode + ".chiral.pdb"

fix_chirality(
    input_pdb_path=pdb_amides,
    output_pdb_path=pdb_chiral
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_amides)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_chiral)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="sidechains"></a>
##  Side Chains
Presence of **missing protein side chain atoms**. MD programs only work with complete residues; some of the MD packages come with tools able to **model missing side chain atoms**, but not all of them. 
***
Use **BioBB fix_side_chain** to model the **missing side chain atoms**. 

 - [fix_side_chain](https://biobb-model.readthedocs.io/en/latest/model.html#module-model.fix_side_chain) from **biobb_model.model.fix_side_chain**
***


```python
print(json.dumps(data['fixside'], indent=2))
```


```python
from biobb_model.model.fix_side_chain import fix_side_chain

pdb_side_chains = pdbCode + ".sidechains.pdb"

fix_side_chain(
    input_pdb_path=pdb_chiral,
    output_pdb_path=pdb_side_chains
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_chiral)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1.add_representation(repr_type='ball+stick', radius='0.4', selection='7 56 63 123 127 143 155')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_side_chains)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2.add_representation(repr_type='ball+stick', radius='0.4', selection='7 56 63 123 127 143 155')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="backbone"></a>
##  Backbone
Presence of **missing backbone atoms**. **PDB files** can have **missing backbone atoms**, usually placed in **flexible regions** of the molecule. **MD programs** work with amino acid libraries that need all atoms to be present. Some MD packages tools are able to model **side chain missing atoms**, but they are rarely capable of modeling **backbone atoms**. 
***
Use **BioBB fix_backbone** to model the **missing backbone atoms**. <br>

 - [fix_backbone](https://biobb-model.readthedocs.io/en/latest/model.html#module-model.fix_backbone) from **biobb_model.model.fix_backbone**

***NOTE:*** This building block uses [Modeller](https://salilab.org/modeller) to rebuilt **missing backbone atoms**. **Modeller** requires a **license** that can be easily obtained in the [Modeller registration page](https://salilab.org/modeller/registration.html). This **license** in form of a **keyword** needs to be written in a specific file in order for **Modeller** to work. This file is shown in the next cell. Alternatively, the keyword can be inserted using the **modeller_key** property of the **building block**.
 
An additional **building block** is used to extract the **canonical FASTA sequence** for the protein, to be used in **Modeler** to model the **missing backbone region**:

 - [canonical_fasta](https://biobb-io.readthedocs.io/en/latest/api.html#module-api.canonical_fasta) from **biobb_io.api.canonical_fasta**

***


```python
print(json.dumps(data['backbone'], indent=2))
```


```python
from biobb_io.api.canonical_fasta import canonical_fasta

pdb_fasta = pdbCode + ".fasta"

prop = {
    'pdb_code': pdbCode,
    'api_id': 'pdb'
}

canonical_fasta(
    output_fasta_path=pdb_fasta,
    properties=prop
)
```


```python
# Add Modeller key in the config file before running the next cell
# Alternatively, add the Modeller key in the "modeller_key" property of the fix_backbone building block
import os
conda_env_path = os.environ['CONDA_PREFIX']
modeller_key_path = conda_env_path + "/lib/modeller-10.3/modlib/modeller/config.py"
print("WARNING: Edit this file and add your Modeller KEY:\n " + modeller_key_path)
```


```python
from biobb_model.model.fix_backbone import fix_backbone

pdb_backbone = pdbCode + ".backbone.pdb"

prop = {
    
}

fix_backbone(
    input_pdb_path=pdb_side_chains,
    input_fasta_canonical_sequence_path=pdb_fasta,
    output_pdb_path=pdb_backbone,
    properties=prop
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_side_chains)
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_backbone)
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="clashes"></a>
##  Atomic Clashes
Presence of **atomic clashes**. Atoms that are **too close in space** can have a problem of **energetic repulsion**. Most clashes come from **over-compactation** of crystal structures and are naturally corrected on **system setup** or **MD equilibration**, but may lead to a significant **distortion** of the structure. **Clashes** are detected based on **distance criteria** and are classified in different groups, depending on the **atom types** involved:

- **Severe**: Atoms too close, usually indicating superimposed structures or badly modelled regions. Should be fixed.
- **Apolar**: Vdw collisions. Usually fixed during the simulation.
- **Polar and ionic**: Usually indicate wrong side chain conformations. Usually fixed during the simulation

***
Use the **BioBB_amber** module to energetically minimize the structure and fix **atomic clashes**.

 - [leap_gen_top](https://biobb-amber.readthedocs.io/en/latest/leap.html#module-leap.leap_gen_top) from **biobb_amber.leap.leap_gen_top**
 - [sander_mdrun](https://biobb-amber.readthedocs.io/en/latest/sander.html#module-sander.sander_mdrun) from **biobb_amber.sander.sander_mdrun**
 - [amber_to_pdb](https://biobb-amber.readthedocs.io/en/latest/ambpdb.html#module-ambpdb.amber_to_pdb) from **biobb_amber.ambpdb.amber_to_pdb**

***


```python
print(json.dumps(data['clashes'], indent=2))
```


```python
from biobb_amber.leap.leap_gen_top import leap_gen_top

pdb_amber = pdbCode + ".amber.pdb"
top_amber = pdbCode + ".amber.top"
crd_amber = pdbCode + ".amber.crd"

prop = {
    'forcefield': ['protein.ff14SB']
}

leap_gen_top(
    input_pdb_path=pdb_backbone,
    output_pdb_path=pdb_amber,
    output_top_path=top_amber,
    output_crd_path=crd_amber,
    properties=prop
)
```


```python
from biobb_amber.sander.sander_mdrun import sander_mdrun

trj_amber = pdbCode + ".amber.crd"
rst_amber = pdbCode + ".amber.rst"
log_amber = pdbCode + ".amber.log"

prop = {
    'simulation_type' : 'minimization',
    'mdin' : {
        'ntb' : 0,        # Periodic Boundary. No periodicity is applied and PME (Particle Mesh Ewald) is off.
        'cut' : 12,       # Nonbonded cutoff, in Angstroms.
        'maxcyc' : 500,   # Maximum number of cycles of minimization.
        'ncyc' : 50       # Minimization will be switched from steepest descent to conjugate gradient after ncyc cycles.
    }
}

sander_mdrun(
     input_top_path=top_amber,
     input_crd_path=crd_amber,
     output_traj_path=trj_amber,
     output_rst_path=rst_amber,
     output_log_path=log_amber,
     properties=prop
)
```


```python
from biobb_amber.ambpdb.amber_to_pdb import amber_to_pdb

pdb_amber_min = pdbCode + ".amber-min.pdb"

amber_to_pdb(
      input_top_path=top_amber,
      input_crd_path=rst_amber,
      output_pdb_path=pdb_amber_min
)
```


```python
# Show modified structure
view1 = nglview.show_structure_file(pdb_backbone)
view1.add_representation(repr_type='ball+stick', selection='all')
view1.add_representation(repr_type='spacefill', selection='ion')
view1.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view1._remote_call('setSize', target='Widget', args=['','450px'])
view2 = nglview.show_structure_file(pdb_amber_min)
view2.clear_representations()
view2.add_representation(repr_type='cartoon', selection='all')
view2.add_representation(repr_type='ball+stick', selection='all')
view2.add_representation(repr_type='spacefill', selection='ion')
view2.add_representation(repr_type='spacefill', radius='0.3', selection='water')
view2._remote_call('setSize', target='Widget', args=['','450px'])
ipywidgets.HBox([view1, view2])
```

<a id="fix_pdb"></a>
##  Fix PDB

Information included in the **original structure** is sometimes **lost** during the **preparation process**. Typical examples are the **residue numeration** and the **structure chains**, that are **changed** or even **removed** by some **MD tools** (as happened with the chains in the previous **energy minimization** process). The **fix_pdb** building block renumerates the residues in a **PDB structure** according to reference **sequences from UniProt** and assigns **chain ids** accordingly.
***
Use the **BioBB_model** module to ***fix*** the structure, **renumerating** the residues and assigning **chain ids**.

 - [fix_pdb](https://biobb-model.readthedocs.io/en/latest/fix_pdb.html#module-model.fix_pdb) from **biobb_model.model.fix_pdb**

***


```python
from biobb_model.model.fix_pdb import fix_pdb

final_pdb = pdbCode + ".final.pdb"

fix_pdb(
    input_pdb_path=pdb_amber_min,
    output_pdb_path=final_pdb
)
```

<a id="finalcheck"></a>
##  Final Check
Checking the **final structure** to analyse the **possible issues** present in the **modified structure** in comparison with the issues we found with the **original structure** at the beginning of the workflow. 
***
**Building Blocks** used:
 - [structure_check](https://biobb-structure-utils.readthedocs.io/en/latest/utils.html#module-utils.structure_check) from **biobb_structure_utils.utils.structure_check**
***


```python
from biobb_structure_utils.utils.structure_check import structure_check

report_final = pdbCode + ".report_final.json"

structure_check(
    input_structure_path=final_pdb,
    output_summary_path=report_final
)
```


```python
import json
with open(report_final, 'r') as f:
  data_final = json.load(f)
print(json.dumps(data_final, indent=2))
```

Printing the possible **issues** identified by the **structure_check** building block in the **original structure** (left column) against the **checked structure** (right column). The main differences are the elimination of the **alternative locations**, **metal ions** and **ligands**. Most of the **clashes** have also disappeared thanks to the **short energetic minimization**.  

The **structure** should be now ready to start the **MD setup process**.


```python
from ipywidgets import Layout

output1 = ipywidgets.widgets.Output(layout=Layout(width='50%'))
with output1:
    display(print(json.dumps(data, indent=2)))
output2 = ipywidgets.widgets.Output(layout=Layout(width='50%'))
with output2:
    display(print(json.dumps(data_final, indent=2)))

two_columns = ipywidgets.widgets.HBox([output1, output2])

display(two_columns)
```

***
<a id="questions"></a>

## Questions & Comments

Questions, issues, suggestions and comments are really welcome!

* GitHub issues:
    * [https://github.com/bioexcel/biobb](https://github.com/bioexcel/biobb)

* BioExcel forum:
    * [https://ask.bioexcel.eu/c/BioExcel-Building-Blocks-library](https://ask.bioexcel.eu/c/BioExcel-Building-Blocks-library)
