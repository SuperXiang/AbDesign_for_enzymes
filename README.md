# AbDesign for enzymes
Scripts and data to run AbDesign as described in Tools for protein science 2020.   
Below I'll describe the different steps, data needed and example command lines to generate a repertoire of structures for the GH10 xylanase family. Steps 2-4 are using PDB accession number 4pud as an example but should be repeated for each of the structures of the chosen protein family.
To run the following, you are required to install:
  * python > 3.5 
  * Rosetta (free for academic users)
  * PyMOL 

## 1 Structures and Segmentation scheme
The list of xylanase structures was taken from [here](http://www.cazy.org/GH10_structure.html) and downloaded from the PDB. All structures must be aligned to the template structure, which can be done using the **alignto** command in PyMOL.

## 2 Structures idealization
To generate an idealized version of a protein structure, use (change the *-s* path to your structure):
```bash 
ROSETTA_SCRIPTS @idealization/flags -s idealization/4pud.pdb -out:prefix ideal_ 
```
The idealized version is ideal_4pud.pdb.gz. 

## 3 Backbone fragments databases
### 3.1 Initial crude fragment extraction
*Segmentation point selection*: After we define the segmentation scheme in step 1, here we start by a rough segmentation (to be further refined in following steps). Say we want a segment starting at residue X and ending at residues Y, we will start from X-3 and Y+3.   

The segmentation points in this step are (numbering corresponds to template structure pdbID: 3w24):
| Segment | start | end |
| :---: | :---: | :---: |
| 1 | 19 | 47 |
| 2-4 | 44 | 189 |
| 5-6 | 183 | 256 |
| 7-8 | 246 | 324 |

The following command is used to extract a fragment structure corresponding to the fragment starting with *start_res* and ends with *end_res* in the *source* structure (i.e. the template). This command should be used for each protein in the family for each of the segments.  
For example, extracting the first fragment from structure 4pud.pdb would use the following command:
```bash 
ROSETTA_SCRIPTS @backbone_database/flags_cod -s backbone_database/4pud.pdb -out:prefix blade1_  -parser:protocol backbone_database/cut_out_domain.xml -parser:script_vars source=template_data/3w24_template.pdb.gz start_res=19 end_res=47
```
The output pdb of the fragment will be located at: pdbs/blade1_4pud.pdb.gz (you need to create the pdbs folder before running the command).  
### 3.2 Refinement of fragment alignment to template
In this step, fragments from the same segment are refined according to the template structure, such that they all have the exact same ends of the fragment to better assemble later (details in the paper).  
  
*Segmentation point selection*: Here we are using the exact residues for the start and end of the segment. Make sure the segments are not overlapping (e.g. a segment cannot start at residue 47 if the previous one ended at 48).  
In this step the template's fragments start & end at the following positions: 
| Segment | start | end |
| :---: | :---: | :---: |
| 1 | 19 | 45 |
| 2-4 | 46 | 186 |
| 5-6 | 187 | 249 |
| 7-8 | 250 | 320 |

In the command below, the fragment from step 2.1 is being matched to the fragment 1 from the template
```bash 
pymol -c ./backbone_database/find_best_match.py template_data/blade1_template.pdb backbone_database/blade1_4pud.pdb.gz
```
The resulting pdb file will be located at *pdbs/fbm_blade1_4pud.pdb*.
## 4 PSSM
First, you need to generate a PSSM for each of the structures of your protein family. We used PSSM generated by [PROSS](https://pross.weizmann.ac.il/step/pross-terms/) (details in the paper).  
The following command extracts a sub-PSSM, matching a fragment of the protein:
```bash
./pssm/cut_pssm_for_fragment.py backbone_database/fbm_blade1_4pud.pdb pssm/4pud.pssm
```
The output pssm is *fbm_blade1_4pud.pssm*.  
## 5 Torsion database
This step extracts the torsion angles from each of the fragment. Before running the command, we have to prepare a few files:
  1. **Directories**:  we need to create to the following directories to hold the output: ```mkdir pdbs scores  db```
  2. **Structures names**: rename the structures of the fragments to include only the pdbID, e.g.: 4pud.pdb or 4pud.pdb.gz
  3. **pdb_profile_match**: a file mapping names of pdbs. See example at *torsions_database/pdb_profile_match*
  4. **flags_pssm**: mapping pdb fragments to their PSSM files. See the format at *torsions_database/flags_pssm*
  5. **flags**: When running with a different protein family, change the path to the appropriate template structure, the catalytic residues to keep conformation and all other paths to the correct ones. 
  6. **splice_out.xml**: When running with a different protein family, change the segments section of the Splice mover to match your naming and segmentation scheme. The frm1 & frm2 tags correspond to the start and end of the protein, respectively, which are kept constant in all designs (i.e. residues before and after the first and last segment)  
  
*Segmentation point selection*: The segmentation points here are **one residue into the segment** relative to step 3.2.
| Segment | start | end |
| :---: | :---: | :---: |
| 1 | 20 | 44 |
| 2-4 | 47 | 185 |
| 5-6 | 188 | 248 |
| 7-8 | 251 | 319 |  

To generate the torsion database file for 4pud.pdb (i.e. the fragment's structure from step 3.2) use:
```bash
ROSETTA_SCRIPTS @torsions_database/flags -out:prefix 4pud_ -parser:script_vars source=torsions_database/4pud.pdb db=db/blade1_4pud.db start_res=20 end_res=44 current_segment=blade1
```
The torsion angles of the fragment will be generated at *db/blade1_4pud.db*. The fragment in the context of the template is located at *pdbs/4pud_3w24_template.pdb.gz* (not needed for later steps, but useful for debugging).   

For further explanations please [see](https://www.rosettacommons.org/docs/latest/scripting_documentation/RosettaScripts/Movers/SpliceOut).


## 6 Assembly of backbones
Here we will generate a new backbone by combining different fragments. Input files to prepare:
  1. **Directories**:  we need to create to the following directories to hold the output: ```mkdir pdbs scores```
  2. **pdb_profile_match & flags_pssm**: as previous step
  3. **flags**: as before, change paths if using a different family
  4. **Torsion database**: for each segment, combine all torsions calculated in the previous step to a single file, each line is the torsion of a single fragment of this segment. For example see: *backbone_assembly/blade1.db*
  5. **splice_in.xml**: When running with a different protein family, change the segments section of the Splice mover to match your naming and segmentation scheme. The frm1 & frm2 tags correspond to the start and end of the protein, respectively, which are kept constant in all designs (i.e. residues before and after the first and last segment)
  
  Example command to generate a new backbone (change the pdbID in the entries to generate a backbone from different fragments):
  ```bash
ROSETTA_SCRIPTS @backbone_assembly/flags -s template_data/3w24_template.pdb.gz -out:prefix 4pud_4qdmB_1xyzA_1e5nB_ -parser:script_vars entry_blade1=4pud entry_blade2_4=4qdmB entry_blade5_6=1xyzA entry_blade7_8=1e5nB
```
