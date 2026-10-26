# Systematic Comparison of MOP before and after the ODK migration
[author](http://purl.org/dc/terms/creator): [Philip Strömert](https://orcid.org/0000-0002-1595-3213) | [created on](http://purl.org/dc/terms/created): 01.10.2026

## Pre-ODK Migration Background
Before the migration to an Ontology Development Kit (ODK) based workflow, the two ontologies RXNO and MOP were maintained within the same GitHub repository: https://github.com/rsc-ontology/rxno. Their release workflow edited the release files `rxno.owl` and `mop.owl` directly, so these two files were the only source of truth for both ontologies, which is in contrast to an ODK-based workflow, where release files are generated automatically from a separate editor OWL file and/or component files (OWL files and/or ROBOT TSV files).

In 2021, as a result of the [Ontologies4Chem: the landscape of ontologies in chemistry](https://doi.org/10.1515/pac-2021-2007) overview paper, the NFDI4Chem team began improving both ontologies by filing issues and pull requests in this single RXNO repository. One of the first suggestions was to properly import MOP into RXNO via `owl:import` instead of (re)defining the MOP classes inside the `rxno.owl` file (see [issue #27](https://github.com/rsc-ontology/rxno/issues/27)).

While trying to resolve this issue in 2022, it became apparent that the MOP classes defined in the `rxno.owl` file were not always just copied over from `mop.owl` file, but were differently defined in both files. This led to a state in which:
* some MOP classes were identical in both files,
* some MOP classes were defined only in `rxno.owl` and thus missing from `mop.owl`,
* and some MOP classes sharing the same PURL in both files
  * represented the same reaction but carried different axioms or annotations, or
  * represented *different* reactions in MOP and RXNO, and thus had clashing PURLs.

To resolve this and enable a proper import of MOP into RXNO in an ODK-based repository, a systematic analysis was needed to identify: which MOP classes could be safely cut from RXNO and copied to MOP, which had to be split into two classes to avoid PURL clashes, and which had to be vetted for differing axiomatisation. The following sections describe how this analysis was conducted.

## Creating and Archiving a Pre-ODK Release of RXNO and MOP
Before the migration to ODK, both ontologies were published only directly in the GitHub repository's code base; no GitHub releases had ever been created. We therefore created a pre-ODK release and tag so that these versions of RXNO and MOP could be referenced: https://github.com/rsc-ontology/rxno/tree/pre-odk-release

## Using ROBOT to Identify the Differences Between the MOP Terms Defined in the Pre-ODK `rxno.owl` and `mop.owl` Files
When this task was initially addressed by NFDI4Chem in 2022, ROBOT was used to produce CSV for both RXNO and MOP to allow their comparison within a spreadsheet editor (see also: https://github.com/rsc-ontology/rxno/issues/27#issuecomment-1091415293). The comparison was then done within an Excel file that was shared with the RXNO/MOP maintainer Colin Batchelor, in order to discuss how to fix the above stated problems on a case by case (term by term) basis. This Excel file, which was later migrated to Google Sheets for better collaborative working, formed the basis for the ODK migration. However, in order to have another quality control step and provide a more reproducible approach, we decided to use the [ROBOT diff](https://robot.obolibrary.org/diff) feature to better document and compare the differences of MOP terms defined in RXNO and MOP before the ODK migration.

### Using ROBOT diff to Show the Differences Between MOP Terms Defined in RXNO and MOP
1. Download the pre-ODK RXNO and MOP release files:
    ```shell
    curl -o rxno_pre_ODK.owl https://raw.githubusercontent.com/rsc-ontology/rxno/refs/tags/pre-odk-release/rxno.owl && \
    curl -o mop_pre_ODK.owl https://raw.githubusercontent.com/rsc-ontology/rxno/refs/tags/pre-odk-release/mop.owl
    ```
2. Use the files downloaded in step 1 in ROBOT to produce ontologies that contain only the MOP classes and only reference all external ontology terms via their PURL:
    ```shell
    robot remove -i mop_pre_ODK.owl --base-iri MOP --axioms external --preserve-structure false \
      --trim false convert -f ofn \
      -o mop_base_pre_ODK.owl  && \
    robot remove -i rxno_pre_ODK.owl --base-iri MOP --axioms external --preserve-structure false \
      --trim false convert -f ofn \
      -o mop_base_in_rxno_pre_ODK.owl
    ```
3. Use the `mop_base_in_rxno_pre_ODK.owl` file created in step 2 to build a term list containing only the MOP terms defined in this pre-odk RXNO version:
    ```shell
    robot export --input mop_base_in_rxno_pre_ODK.owl --header "ID|LABEL" --include "classes properties" --export mop_base_in_rxno_pre_ODK.csv && \
    grep '^MOP:' mop_base_in_rxno_pre_ODK.csv | \
    sed 's/,/\t# /' > mop_in_rxno_terms.txt && \
    rm mop_base_in_rxno_pre_ODK.csv
    ```
4. Use the `mop_in_rxno_terms.txt` file from step 3 to filter the `mop_base_pre_ODK.owl` from step 1 into an MOP module that contains only those classes also present in `rxno_pre_ODK.owl`. This was done to produce a slimmer diff between the pre-ODK MOP and RXNO versions.
    ```shell
    robot extract -i mop_base_pre_ODK.owl --method BOT \
      --term-file mop_in_rxno_terms.txt --copy-ontology-annotations true \
      convert -f ofn \
      -o mop_base_pre_ODK_slim.owl
    ```
5. Produce the diff showing how the MOP classes differ between RXNO and MOP before the ODK migration:
    ```shell
    robot diff --left mop_base_in_rxno_pre_ODK.owl --right mop_base_pre_ODK_slim.owl -f markdown -o mop_in_rxno_pre_ODK_diff.md
    ```

## Analysing the ROBOT diff Between Pre-ODK RXNO & MOP 
This section analyses the differences between the MOP terms defined in the pre ODK `rxno.owl` and `mop.owl` files made visible by the ROBOT markdown document created in step 5 of the previous section.

### Pipeline Artifact
#### Ontology-level annotations

**Cause:** The left side of the diff carries the metadata of the original `rxno.owl` file, while the right side carries the metadata of the original `mop.owl` file.

**Action:** Ignore. These differences are expected, because the right side are the correct metadata. 

#### Spurious `hasOBONamespace "RXNO"` on MOP terms

**Cause:** When MOP classes were defined inside `rxno.owl`, they received the `hasOBONamespace "RXNO"` annotation. This is incorrect, because MOP terms are already identified as MOP-namespace terms by their IRI (`http://purl.obolibrary.org/obo/MOP_...`). The `mop.owl` file correctly omits this wrong annotation.

**Action:** Ignore. The absence of this annotation in `mop.owl` is correct behaviour.

**Affected terms:** All MOP terms that were defined in `rxno.owl` carry this spurious annotation. It is removed for every such term in the diff. No individual listing is needed, and it can be treated as universal noise.

**Example:**
- **substitution reaction** `http://purl.obolibrary.org/obo/MOP_0000790`
  - **Removed**
    - [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"


#### External Terms Added or Removed

**Cause:** The diff compares two OWL files that each contain only the axioms relevant to MOP terms. An external term (from BFO, CHEBI, IAO, RO, etc.) declaration appears in one file but not the other depending on whether it is referenced in an axiom on a MOP term *in that specific file*. 

**Action:** Ignore. The *axiom* difference is the semantic issue (covered under *Semantic Drift* below); the *presence of the external term declaration* is a pipeline artifact associated with the OWL syntax.

**Removed external terms (18):** `BFO_0000016`, `BFO_0000162`, `CHEBI_15734`, `CHEBI_16526`, `CHEBI_17087`, `CHEBI_17478`, `CHEBI_29785`, `CHEBI_30879`, `CHEBI_33308`, `CHEBI_33338`, `CHEBI_33595`, `CHEBI_33823`, `CHEBI_35681`, `CHEBI_36587`, `CHEBI_42485`, `CHEBI_46883`, `IAO_0000030`, `RXNO_0000344`

**Added external terms (10):** `BFO_0000117`, `CHEBI_22221`, `CHEBI_33641`, `CHEBI_51100`, `CHEBI_52214`, `CHEBI_52845`, `IAO_0000116`, `RO_0002213`, `dc:creator`, `dc:date`

> **Note:** Some of these "added" terms (`BFO_0000117`, `CHEBI_22221`, `CHEBI_51100`, `CHEBI_52214`, `CHEBI_52845`, `RO_0002213`) are added because `mop.owl` uses them in axioms where `rxno.owl` used a different term or no axiom at all.


#### Classes Added Due to MOP Base Module Extraction  
Some MOP terms appear as *added* in the diff purely because of step 4 of the workflow, where the right-hand file (`mop_base_pre_ODK_slim.owl`) was built with `robot extract --method BOT`. BOT pulls in every ancestor needed to keep the extracted classes connected, so parent or grandparent classes that are present in `mop.owl` — but that were *not* part of the term list extracted from `rxno.owl` — get added to the slim module and consequently show up as additions in the diff.

This reason explains the differences on:
* alkene oxidative cleavage `http://purl.obolibrary.org/obo/MOP_0000708`
* alkenylation `http://purl.obolibrary.org/obo/MOP_0000420`
* breaking of covalent bond `http://purl.obolibrary.org/obo/MOP_0000780`
* catalysis `http://purl.obolibrary.org/obo/MOP_0000781`
* dehydrocarbylation `http://purl.obolibrary.org/obo/MOP_0001410`
* formation of covalent bond `http://purl.obolibrary.org/obo/MOP_0000779`
* formation of covalent bond with group `http://purl.obolibrary.org/obo/MOP_0000566`
* formation of covalent bond with carbon centre `http://purl.obolibrary.org/obo/MOP_0000800`
* formation of covalent bond with halogen centre `http://purl.obolibrary.org/obo/MOP_0000820`
* formation of covalent bond with silicon centre `http://purl.obolibrary.org/obo/MOP_0000810`
* formation of covalent bond with sulfur centre `http://purl.obolibrary.org/obo/MOP_0000798`
* is_catalysis_of `http://purl.obolibrary.org/obo/MOP_0000769`
* oxidative cleavage `http://purl.obolibrary.org/obo/MOP_0000707`
* sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000728`
* univalent carboacylation `http://purl.obolibrary.org/obo/MOP_0000029`

### Semantic Changes

#### Critical Severity

<a id="iri-clashes"></a>
##### IRI clashes: One IRI, two different reactions

**Cause:** Some MOP IRIs were used for *different reactions* in `rxno.owl` vs. `mop.owl`. In `rxno.owl`, the IRI carried the RXNO meaning; in `mop.owl`, the same IRI carried the canonical MOP meaning. These are not annotation drift. They are entirely different chemical processes sharing a single PURL.

**Action:** For each clash, decide:
- If the RXNO-defined term already has an equivalent MOP term under a *different* IRI → reuse the existing MOP IRI and sync annotations & logical axioms.
- If the RXNO-defined term has no equivalent in MOP → mint a new MOP IRI, copy the full definition and axioms from `rxno.owl` and add a comment that points to the PR where this was done.

**Affected terms (5):**

| IRI         | Meaning in `rxno.owl`        | Meaning in `mop.owl`     | Sub-case                                          |
|-------------|------------------------------|--------------------------|---------------------------------------------------|
| MOP_0000715 | [3+2] cycloaddition          | trimerization            | New MOP IRI must be minted                        |
| MOP_0000714 | carbonyl oxidation to alkyne | ligand exchange reaction | New MOP IRI must be minted                        |
| MOP_0000713 | carboxylation                | acetylene reduction      | MOP_0000591 is the correct ID for 'carboxylation' |
| MOP_0000716 | enolization                  | cyclotrimerization       | MOP_0000672 is the correct ID for 'enolization'   |
| MOP_0000712 | oxidation state              | olefin reduction         | New MOP IRI must be minted                        |

> **Note on diff appearance:** These clashes do NOT show up as class removals in the diff. Instead, the diff shows the `rxno.owl` annotations (label, definition, SubClassOf) as "Removed" and the `mop.owl` annotations as "Added", because ROBOT sees the same IRI with different content on each side. The linked label in the diff header uses the `rxno.owl` label simply because it was read in by ROBOT first, which can be misleading.

**Example:**
- **\[3+2\] cycloaddition** `http://purl.obolibrary.org/obo/MOP_0000715`
  - **Removed** (this is what `rxno.owl` defined at this IRI)
    - [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A cycloaddition in which one participant contributes three electrons and the other participant contributes two electrons to the transformation of reactants to products." 
      - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
    - [label](http://www.w3.org/2000/01/rdf-schema#label) "[3+2] cycloaddition"
    - SubClassOf [cycloaddition](http://purl.obolibrary.org/obo/MOP_0000562) 
  - **Added** (this is what `mop.owl` defined at this same IRI)
    - [IAO_0000116](http://purl.obolibrary.org/obo/IAO_0000116) "The GO class &quot;protein trimerization&quot; (https://purl.obolibrary.org/obo/GO_0070206) would actually be a subclass of &quot;trimerization&quot; according to the definition provided here. However, for the reason of not wanting to inject this subclassOf axiom into GO, we decided to not assert this in MOP as of yet. Further discussion is needed with the GO and/or OBO team to see how this GO class can be integrated."
    - [creator](http://purl.org/dc/elements/1.1/creator) [0000-0002-4378-6061](http://orcid.org/0000-0002-4378-6061)
    - [date](http://purl.org/dc/elements/1.1/date) "2022-02-01T09:40:48Z"^^[dateTime](http://www.w3.org/2001/XMLSchema#dateTime)
    - [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "The formation of a trimer from three molecular subunits." 
      - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://en.wiktionary.org/wiki/trimerization"^^[anyURI](http://www.w3.org/2001/XMLSchema#anyURI)
    - [label](http://www.w3.org/2000/01/rdf-schema#label) "trimerization"@en
    - [seeAlso](http://www.w3.org/2000/01/rdf-schema#seeAlso) [45](https://github.com/rsc-ontologies/rxno/issues/45)
    - [seeAlso](http://www.w3.org/2000/01/rdf-schema#seeAlso) [issuecomment-1124151946](https://github.com/rsc-ontologies/rxno/pull/46#issuecomment-1124151946)
    - SubClassOf [molecular process](http://purl.obolibrary.org/obo/MOP_0000543) 

<a id="duplicates"></a>
##### Duplicates: One reaction, two IRIs

**Cause:** The concept `alkene oxidative cleavage` was defined twice: once as `RXNO:0000344` in the RXNO namespace and once as `MOP:0000708` in the MOP namespace. In `rxno.owl`, MOP terms like `alkene ozonolysis` used the RXNO IRI as their parent; in `mop.owl`, they correctly used either the MOP IRI or its parent classes.

**Action:** The RXNO duplicate (`RXNO:0000344`) needs to be deprecated in favor of the MOP term (`MOP:0000708`). The subsumption of its children within MOP should be reviewed to check if `MOP:0000708` is the better parent, than the current use of `oxidative cleavage` & `alkene oxidation` (`MOP:0000708` parents) for some of the affected terms.

**Affected terms (6):**
- alkene ozonolysis `http://purl.obolibrary.org/obo/MOP_0000583`
- primary alkene oxidation to carboxylic acid and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000584`
- quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588`
- secondary, non-terminal alkene oxidation to aldehydes `http://purl.obolibrary.org/obo/MOP_0000586`
- secondary terminal alkene oxidation to ketone and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000587`
- tertiary alkene oxidation to carboxylic acid and ketone `http://purl.obolibrary.org/obo/MOP_0000585`

**Example:**
- **quaternary alkene oxidation to ketones** `http://purl.obolibrary.org/obo/MOP_0000588`
  - **Removed**
    - SubClassOf [RXNO_0000344](http://purl.obolibrary.org/obo/RXNO_0000344)
  - **Added**
    - SubClassOf [alkene oxidation](http://purl.obolibrary.org/obo/MOP_0000581)
    - SubClassOf [oxidative cleavage](http://purl.obolibrary.org/obo/MOP_0000707) 


<a id="mop-only-in-rxno"></a>
#### High Severity: MOP terms only defined in `rxno.owl`

**Cause:** Ten MOP terms were defined exclusively in the `rxno.owl` file and were never copied/added to `mop.owl`. They carry full definitions, labels, and logical axioms in `rxno.owl` but are entirely absent from `mop.owl`. A naive removal of MOP terms from `rxno.owl` would silently delete these classes.

**Action:** These terms must be copied into MOP before they can be cut from RXNO. Each requires verification that its parent classes and referenced external terms exist in MOP.

**Affected terms (10):**
- O-dealkylation `http://purl.obolibrary.org/obo/MOP_0000719`
- P-alkylation `http://purl.obolibrary.org/obo/MOP_0006369`
- [2+2+1] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000720`
- dealcoholative condensation `http://purl.obolibrary.org/obo/MOP_0000825`
- decarboxylation `http://purl.obolibrary.org/obo/MOP_0001713`
- decarboxylative condensation `http://purl.obolibrary.org/obo/MOP_0000722`
- deethanolative condensation `http://purl.obolibrary.org/obo/MOP_0000723`
- dienophilicity `http://purl.obolibrary.org/obo/MOP_0000826`
- enolisability `http://purl.obolibrary.org/obo/MOP_0000717`
- enolisable carbonyl compound `http://purl.obolibrary.org/obo/MOP_0000718`

> **Note:** `decarboxylative condensation` (MOP:0000722) also uses the local `rxno.obo#has_part` relation in its axioms. When copied to MOP, this must be replaced with `BFO_0000117` (has occurrent part) — see "Low Severity: `rxno.obo#has_part` replaced by `BFO_0000117`" below.

**Example:**
- **O-dealkylation** `http://purl.obolibrary.org/obo/MOP_0000719`
  - **Removed** (entire class definition absent from `mop.owl`)
    - Class: [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719)
    - [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
    - [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [id](http://www.geneontology.org/formats/oboInOwl#id) "MOP:0000719"
    - [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "Breaking of a covalent bond between an alkyl group and an oxygen atom." 
      - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:xp"
    - [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [label](http://www.w3.org/2000/01/rdf-schema#label) "O-dealkylation"
    - [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) SubClassOf [dealkylation](http://purl.obolibrary.org/obo/MOP_0001369) 

<a id="axiom-diffs"></a>
#### Medium Severity: Different logical axioms on shared MOP terms

**Cause:** Some MOP terms exist in both files and refer to the same reaction, but their SubClassOf axioms or EquivalentTo axioms differ. In several cases, `mop.owl` has a more refined hierarchy (e.g., `halogenation` is a child of `formation of covalent bond with halogen centre` in `mop.owl` but only of `formation of covalent bond with group` in `rxno.owl`). In other cases, `rxno.owl` carried reaction participant axioms that in `mop.owl` were only present as `rdfs:comment` annotations.

**Action:** Each case requires domain-expert review to decide which axiomatization is correct. Where `rxno.owl` had a more specific or additional axiom, it should be salvaged into MOP. 

**Affected terms (33):**
- N-arylation `http://purl.obolibrary.org/obo/MOP_0002411`
- [3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
- [4+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000565` (removed axiom references `dienophilicity`, which is RXNO-only — see cascade note below)
- acid catalysis `http://purl.obolibrary.org/obo/MOP_0000740`
- acylation `http://purl.obolibrary.org/obo/MOP_0000479`
- alcohol oxidation `http://purl.obolibrary.org/obo/MOP_0000572`
- aldehyde reduction `http://purl.obolibrary.org/obo/MOP_0000571`
- alkene ozonolysis `http://purl.obolibrary.org/obo/MOP_0000583` (Parent swap only — see [Duplicates: One reaction, two IRIs](#duplicates) for details.)
- allylation `http://purl.obolibrary.org/obo/MOP_0000422`
- amine oxidation `http://purl.obolibrary.org/obo/MOP_0000575`
- arylation `http://purl.obolibrary.org/obo/MOP_0000411`
- breaking of covalent bond with group `http://purl.obolibrary.org/obo/MOP_0000567`
- cyclization `http://purl.obolibrary.org/obo/MOP_0000561`
- cycloaddition `http://purl.obolibrary.org/obo/MOP_0000562`
- dealkylation `http://purl.obolibrary.org/obo/MOP_0001369`
- epoxidation `http://purl.obolibrary.org/obo/MOP_0000671`
- ester reduction `http://purl.obolibrary.org/obo/MOP_0000577`
- ester reduction to aldehyde `http://purl.obolibrary.org/obo/MOP_0000578`
- ester reduction to primary alcohol `http://purl.obolibrary.org/obo/MOP_0000579`
- formylation `http://purl.obolibrary.org/obo/MOP_0000003`
- halogenation `http://purl.obolibrary.org/obo/MOP_0000550`
- ketone reduction `http://purl.obolibrary.org/obo/MOP_0000580`
- nitration `http://purl.obolibrary.org/obo/MOP_0000556`
- organylation `http://purl.obolibrary.org/obo/MOP_0000458`
- primary alcohol oxidation to aldehyde `http://purl.obolibrary.org/obo/MOP_0000573`
- primary alkene oxidation to carboxylic acid and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000584`
- quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588`
- secondary alcohol oxidation to ketone `http://purl.obolibrary.org/obo/MOP_0000574`
- secondary terminal alkene oxidation to ketone and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000587`
- secondary, non-terminal alkene oxidation to aldehydes `http://purl.obolibrary.org/obo/MOP_0000586`
- silylation `http://purl.obolibrary.org/obo/MOP_0000339`
- sulfonylation `http://purl.obolibrary.org/obo/MOP_0000539`
- tertiary alkene oxidation to carboxylic acid and ketone `http://purl.obolibrary.org/obo/MOP_0000585`

> **Sub-pattern: reaction participant axioms as comments.** In `rxno.owl`, several terms had `SubClassOf BFO_0000057 some CHEBI:xxx` axioms. In `mop.owl`, these were only provided as `rdfs:comment "has_participant: CHEBI:xxx"` — a placeholder indicating the axiom was not yet properly added in MOP. Affected: N-arylation, arylation, formylation, nitration, dealkylation. These terms also appear under "Low: Annotation drift" because the comment is technically an annotation change, but the *loss of a logical axiom* is the substantive issue.

> **Sub-pattern: axiom removal cascading from a missing term.** The removed `SubClassOf BFO_0000055 some dienophilicity` axiom on [4+2] cycloaddition (`MOP_0000565`) references `MOP_0000826` (dienophilicity), which is only defined in `rxno.owl` (see "High Severity: MOP terms only defined in `rxno.owl`"). The axiom was *not* deliberately removed in `mop.owl`. It simply could not exist there because its object class was absent. Once `dienophilicity` is copied into MOP (resolving the High Severity item), this axiom must be restored on `MOP_0000565`. This restoration will appear as an added SubClassOf axiom in the pre- vs post-ODK MOP diff.

**Example:**
- **acylation** `http://purl.obolibrary.org/obo/MOP_0000479`
  - **Removed**
    - [comment](http://www.w3.org/2000/01/rdf-schema#comment) "has_participant: CHEBI:22221"
    - SubClassOf [organylation](http://purl.obolibrary.org/obo/MOP_0000458) 
  - **Added**
    - [comment](http://www.w3.org/2000/01/rdf-schema#comment) "you probably want &quot;carboacylation&quot;, MOP:0000028, here" 
    - SubClassOf [formation of covalent bond with carbon centre](http://purl.obolibrary.org/obo/MOP_0000800)
    - SubClassOf [BFO_0000057](http://purl.obolibrary.org/obo/BFO_0000057) some [CHEBI_22221](http://purl.obolibrary.org/obo/CHEBI_22221) 


#### Low Severity: Annotation differences and relation normalization
##### `rxno.obo#has_part` replaced by `BFO_0000117`

**Cause:** The `rxno.owl` file defined a local object property `has_part` (`http://purl.obolibrary.org/obo/rxno.obo#has_part`) and used it in SubClassOf axioms on MOP terms. The `mop.owl` file used the BFO property `has occurrent part` (`BFO_0000117`). These two relations can be considered semantically equivalent.

**Action:** Mechanical. Replace all uses of the local `has_part` with `BFO_0000117` when copying terms to MOP. No domain-expert judgement needed.

> **Note:** The [OBO Foundry principle 7](https://obofoundry.org/principles/fp-007-relations.html) requires OBO ontologies to use the Relations Ontology (RO) for relations and BFO only for upper-level classes. In a later step (post-ODK migration), `BFO_0000117` was itself replaced by its RO equivalent. See the post-ODK diff analysis.

**Affected terms (2):**
- Brønsted base catalysis `http://purl.obolibrary.org/obo/MOP_0000736`
- Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000739`

> `decarboxylative condensation` (MOP:0000722) also used `has_part`, but since that term is entirely absent from `mop.owl` (see "MOP terms only defined in RXNO"), the swap is handled as part of copying the term, not as a standalone diff entry.

**Example:**
- **Brønsted base catalysis** `http://purl.obolibrary.org/obo/MOP_0000736`
  - **Removed**
    - SubClassOf [has_part](http://purl.obolibrary.org/obo/rxno.obo#has_part) some [accepting a hydron](http://purl.obolibrary.org/obo/MOP_0000737) 
  - **Added**
    - SubClassOf [BFO_0000117](http://purl.obolibrary.org/obo/BFO_0000117) some [accepting a hydron](http://purl.obolibrary.org/obo/MOP_0000737) 


##### Different labels

**Cause:** Three MOP terms have different `rdfs:label` values in the two files. The `mop.owl` label is generally shorter and more generic (e.g., "dehydration reaction" vs. "molecular dehydration reaction").

**Action:** Domain-expert review to decide which label is more accurate.

**Affected terms (3):**
- `http://purl.obolibrary.org/obo/MOP_0000628` — "molecular dehydration reaction" → "dehydration reaction"
- `http://purl.obolibrary.org/obo/MOP_0000574` — "secondary alcohol oxidation to ketone" → "secondary alcohol oxidation"
- `http://purl.obolibrary.org/obo/MOP_0000573` — "primary alcohol oxidation to aldehyde" → "primary alcohol oxidation"

**Example:**
- molecular dehydration reaction `http://purl.obolibrary.org/obo/MOP_0000628`
  - Removed
    - [label](http://www.w3.org/2000/01/rdf-schema#label) "molecular dehydration reaction"
  - Added
    - [label](http://www.w3.org/2000/01/rdf-schema#label) "dehydration reaction"

<a id="metadata-changes"></a>
### Metadata Changes (no logical impact)

##### Annotation differences

**Cause:** The two files carry different annotations on otherwise logically identical terms. The most common pattern is that `mop.owl` has richer annotations than `rxno.owl` (definitions added, dbxref URIs updated from `AU:xxx` shorthand to full DOI URLs, synonyms added). One term (`allylic rearrangement`) shows the reverse pattern: `rxno.owl` had an extra `hasDbXref` that is missing in `mop.owl`.

**Action:** Sync annotations by taking the superset from both files, preferring the full DOI URI format from `mop.owl`. No domain-expert judgement needed for dbxref format changes; definition text changes should be reviewed for accuracy.

**Terms where `mop.owl` has richer annotations (27):**
- Brønsted acid catalysis `http://purl.obolibrary.org/obo/MOP_0000730`
- Brønsted base catalysis `http://purl.obolibrary.org/obo/MOP_0000736`
- Lewis acid catalysis `http://purl.obolibrary.org/obo/MOP_0000734`
- Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000739`
- Lewis base catalysis `http://purl.obolibrary.org/obo/MOP_0000735`
- [3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
- accepting a hydron `http://purl.obolibrary.org/obo/MOP_0000737`
- accepting an electron pair in Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000733`
- acylation `http://purl.obolibrary.org/obo/MOP_0000479`
- base catalysis `http://purl.obolibrary.org/obo/MOP_0000731`
- condensation reaction `http://purl.obolibrary.org/obo/MOP_0000627`
- cyclization `http://purl.obolibrary.org/obo/MOP_0000561`
- cycloaddition `http://purl.obolibrary.org/obo/MOP_0000562`
- cycloelimination `http://purl.obolibrary.org/obo/MOP_0000564` (CAVE: synonym typo in MOP for `retrocycloaddition`)
- electron attachment `http://purl.obolibrary.org/obo/MOP_0000570`
- electron pair donation in Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000738`
- elimination reaction `http://purl.obolibrary.org/obo/MOP_0000656`
- hydron donation `http://purl.obolibrary.org/obo/MOP_0000732`
- methylation `http://purl.obolibrary.org/obo/MOP_0000364`
- oxidation `http://purl.obolibrary.org/obo/MOP_0000568`
- reduction `http://purl.obolibrary.org/obo/MOP_0000569`
- solvolysis `http://purl.obolibrary.org/obo/MOP_0000618`
- N-arylation `http://purl.obolibrary.org/obo/MOP_0002411`
- arylation `http://purl.obolibrary.org/obo/MOP_0000411`
- formylation `http://purl.obolibrary.org/obo/MOP_0000003`
- nitration `http://purl.obolibrary.org/obo/MOP_0000556`
- dealkylation `http://purl.obolibrary.org/obo/MOP_0001369`

> The last 5 terms (N-arylation through dealkylation) gained `rdfs:comment "has_participant: CHEBI:xxx"` in `mop.owl` where `rxno.owl` had a real `BFO_0000057 some CHEBI:XXX` axiom. They are listed here because the comment *is* an annotation, but cross-reference "Medium Severity: Different logical axioms on shared MOP terms" for the substantive issue.

**Term where `rxno.owl` had richer annotations (1):**
- allylic rearrangement `http://purl.obolibrary.org/obo/MOP_0000792`

**Example:**
- [3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
  - Added
    - [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A sigmatropic rearrangement where the intermediate consists of two three-electron fragments." 
      - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"


#### Alternative ID annotations

**Cause:** Four terms have `hasAlternativeId` annotations that exist in one file but not the other. These are bookkeeping records of deprecated predecessor IDs.

**Action:** Take the union of all alternative IDs. No logical impact.

**Affected terms (4):**
- N-sulfonylation `http://purl.obolibrary.org/obo/MOP_0002524` — `hasAlternativeId "MOP:0002539"` only in `mop.owl`
- O-sulfonylation `http://purl.obolibrary.org/obo/MOP_0003524` — `hasAlternativeId "MOP:0003539"` only in `mop.owl`
- condensation reaction `http://purl.obolibrary.org/obo/MOP_0000627` — `hasAlternativeId "RXNO:0000315"` only in `rxno.owl`
- cycloaddition `http://purl.obolibrary.org/obo/MOP_0000562` — `hasAlternativeId "RXNO:0000314"` only in `rxno.owl`

## Using ROBOT to identify the differences between MOP before and after the ODK Migration
In this step we produce a diff between the pre-ODK MOP version and the post-ODK version to make sure the ODK migration did not produce any unaccounted changes of MOP. 
The MOP term overlap analysed in the previous section was partially addressed during the MOP migration by copying the MOP terms only defined in RXNO over to MOP, resolving the IRI clashes and some of the differences in SubclassOf axioms or annotations together with Colin Batchelor.

### 1. Download the post-ODK MOP release file:
To compare the pre-ODK MOP version we need to download the first release made after the ODK migration. We're using the base release to exclude imported terms.
```shell
curl -o mop_base_post_ODK.owl https://raw.githubusercontent.com/rsc-ontology/mop/refs/tags/v2026-10-26/mop-base.owl
```

### 2. Make diff to see the changes after the ODK migration
```shell
robot diff --left mop_base_pre_ODK.owl --right mop_base_post_ODK.owl -f markdown -o mop_ODK_diff.md
```

## Analysing ROBOT diff comparing MOP before and after the ODK Migration

### Pipeline Artifact

#### External terms added or removed

**Cause:** The diff compares two OWL files that each contain only the axioms relevant to MOP terms. An external term (from BFO, CHEBI, IAO, RO, etc.) declaration appears in one file but not the other depending on whether it is referenced in an axiom on a MOP term *in that specific file*. 

**Action:** Ignore. The *axiom* difference is the semantic issue (covered under **Semantic Drift** below); the *presence of the external term declaration* is a pipeline artifact associated with the OWL syntax.

**Removed external terms (18):** `BFO_0000057`, `BFO_0000077`, `BFO_0000078`, `BFO_0000117`, `BFO_0000132`, `BFO_0000162`, `CHEBI_52845`, `IAO_0000116`, `RO_0000057`, `RO_0000091`, `RO_0004008`, `RO_0004009`, `http://purl.org/dc/elements/1.1/creator`, `http://purl.org/dc/elements/1.1/date`, `http://purl.org/dc/elements/1.1/description`, `http://purl.org/dc/elements/1.1/title`, `hasOBOFormatVersion`, `http://purl.obolibrary.org/obo/rxno.obo#has_part`, 

**Added external terms (25):** `BFO_0000016`, `BFO_0000050`, `BFO_0000051`, `CHEBI_15734`, `CHEBI_16526`, `CHEBI_17087`, `CHEBI_17478`, `CHEBI_29785`, `CHEBI_30879`, `CHEBI_33308`, `CHEBI_33338`, `CHEBI_33575`, `CHEBI_33595`, `CHEBI_35681`, `CHEBI_36587`, `CHEBI_42485`, `CHEBI_46883`, `IAO_0000030`, `IAO_0000233`, `http://purl.org/dc/terms/contributor`, `http://purl.org/dc/terms/creator`, `http://purl.org/dc/terms/date`, `http://purl.org/dc/terms/description`, `http://purl.org/dc/terms/title`, `http://purl.org/dc/elements/1.1/type`


#### Deleting of Inherited SubClassOf Axioms
**Cause:**
Because we compare release versions, the file generated automatically by ODK from the MOP editor file differs from the pre‑ODK one, owing to the [ROBOT reduce](https://robot.obolibrary.org/reduce) step in the ODK release build pipeline. For example if an axiom is asserted on an MOP term in the editor file (e.g., `'has primary output' some macromolecule` on `'chain polymerisation'`) but the same axiom is already present on the parent of this term (e.g., `polymerisation`), the reduce step will delete this axiom from the child, as it is already inherited by the parent (see 'chain polymerisation' in the MOP release versus the editor file.) 

**Action:**
Delete these redundant axioms from the terms in the MOP editor file.

**Affected terms:**
  * chain polymerisation `http://purl.obolibrary.org/obo/MOP_0000634`
  * condensation polymerisation `http://purl.obolibrary.org/obo/MOP_0000632`
  * condensative chain polymerisation `http://purl.obolibrary.org/obo/MOP_0000645`
  * ionic copolymerisation `http://purl.obolibrary.org/obo/MOP_0000647`
  * living copolymerisation `http://purl.obolibrary.org/obo/MOP_0000649`
  * polycondensation `http://purl.obolibrary.org/obo/MOP_0000644`
  * radical copolymerisation `http://purl.obolibrary.org/obo/MOP_0000648`
  * ring-opening copolymerisation `http://purl.obolibrary.org/obo/MOP_0000688`


### Semantic Changes

#### Replacing BFO with RO Relations
**Cause:**
In the axiomatization of MOP terms, BFO relations, such as `BFO_0000117 # has occurrent part`, were used that are not supported by the BFO classes only version used in the OBO Foundry. According to the [OBO Foundry principle 7](https://obofoundry.org/principles/fp-007-relations.html) relations defined in the Relations Ontology (RO) should be used whenever possible, which is the case for such high level BFO relations. This relation swap accounts for the majority of the changes in the analysed diff.

**Action:**
In the ODK migration these BFO relations were replaced with suitable RO equivalents in [PR#3](https://github.com/rsc-ontology/mop/issues/4) (see also: https://github.com/rsc-ontology/mop/issues/4 & https://github.com/rsc-ontology/mop/issues/7).

**Affected terms:**
All MOP terms that have a SubClassOf axiom change in which the following relation replacements happen:
* `[BFO_0000117](http://purl.obolibrary.org/obo/BFO_0000117)` → `[BFO_0000051](http://purl.obolibrary.org/obo/BFO_0000051)` (Although BFO_0000051 was initially a BFO relation it has since been adopted under this IRI in RO)
  * 3-endo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000750`
  * 3-endo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000751`
  * 3-exo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000749`
  * 3-exo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000778`
  * 3-exo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000748`
  * 4-endo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000754`
  * 4-endo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000753`
  * 4-exo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000762`
  * 4-exo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000741`
  * 4-exo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000752`
  * 5-endo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000758`
  * 5-endo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000745`
  * 5-endo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000757`
  * 5-exo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000756`
  * 5-exo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000742`
  * 5-exo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000755`
  * 6-endo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000770`
  * 6-endo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000746`
  * 6-endo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000759`
  * 6-exo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000763`
  * 6-exo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000743`
  * 6-exo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000765`
  * 7-endo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000761`
  * 7-endo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000747`
  * 7-endo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000760`
  * 7-exo-dig cyclization `http://purl.obolibrary.org/obo/MOP_0000764`
  * 7-exo-tet cyclization `http://purl.obolibrary.org/obo/MOP_0000744`
  * 7-exo-trig cyclization `http://purl.obolibrary.org/obo/MOP_0000766`
  * Brønsted base catalysis `http://purl.obolibrary.org/obo/MOP_0000736`
  * Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000739`
  * chain polymerisation `http://purl.obolibrary.org/obo/MOP_0000634`
  * condensation polymerisation `http://purl.obolibrary.org/obo/MOP_0000632`
  * condensative chain polymerisation `http://purl.obolibrary.org/obo/MOP_0000645`
  * cyanosilylation `http://purl.obolibrary.org/obo/MOP_0000678`
  * electron pair transfer `http://purl.obolibrary.org/obo/MOP_0000679`
  * polyaddition `http://purl.obolibrary.org/obo/MOP_0000643`
  * polycondensation `http://purl.obolibrary.org/obo/MOP_0000644`
  * ring-opening metathesis polymerisation `http://purl.obolibrary.org/obo/MOP_0000685`
  * transmetallation `http://purl.obolibrary.org/obo/MOP_0000677`

* `[BFO_0000132](http://purl.obolibrary.org/obo/BFO_0000132)` → `[BFO_0000050](http://purl.obolibrary.org/obo/BFO_0000050) (Although BFO_0000050 was initially a BFO relation it has since been adopted under this IRI in RO)
  * chain initiation `http://purl.obolibrary.org/obo/MOP_0000640`

* `[BFO_0000057](http://purl.obolibrary.org/obo/BFO_0000057)` → `[RO_0000057](http://purl.obolibrary.org/obo/RO_0000057)` 
  * N-deacetylation `http://purl.obolibrary.org/obo/MOP_0000703`
  * O-deacetylation `http://purl.obolibrary.org/obo/MOP_0000704`
  * acylation `http://purl.obolibrary.org/obo/MOP_0000479`
  * amination `http://purl.obolibrary.org/obo/MOP_0000650`
  * azidation `http://purl.obolibrary.org/obo/MOP_0000598`
  * borylation `http://purl.obolibrary.org/obo/MOP_0000000`
  * carboacylation `http://purl.obolibrary.org/obo/MOP_0000028`
  * cyanation `http://purl.obolibrary.org/obo/MOP_0000597`
  * deamination `http://purl.obolibrary.org/obo/MOP_0000651`
  * deuteron transfer `http://purl.obolibrary.org/obo/MOP_0000683`
  * hydron transfer `http://purl.obolibrary.org/obo/MOP_0000682`
  * hydroxylation `http://purl.obolibrary.org/obo/MOP_0000673`
  * isocyanatation `http://purl.obolibrary.org/obo/MOP_0000600`
  * isocyanation `http://purl.obolibrary.org/obo/MOP_0000599`
  * isoselenocyanatation `http://purl.obolibrary.org/obo/MOP_0000602`
  * isotellurocyanatation `http://purl.obolibrary.org/obo/MOP_0000603`
  * isothiocyanatation `http://purl.obolibrary.org/obo/MOP_0000601`
  * ligand exchange reaction `http://purl.obolibrary.org/obo/MOP_0000714`
  * proton transfer `http://purl.obolibrary.org/obo/MOP_0000680`
  * pseudohalogenation `http://purl.obolibrary.org/obo/MOP_0000596`
  * sulfonylation `http://purl.obolibrary.org/obo/MOP_0000539`

* `[BFO_0000077](http://purl.obolibrary.org/obo/BFO_0000077)` → `[RO_0004008](http://purl.obolibrary.org/obo/RO_0004008)`
  * alternating copolymerisation `http://purl.obolibrary.org/obo/MOP_0000693`
  * block copolymerisation `http://purl.obolibrary.org/obo/MOP_0000699`
  * chain polymerisation `http://purl.obolibrary.org/obo/MOP_0000634`
  * condensation polymerisation `http://purl.obolibrary.org/obo/MOP_0000632`
  * copolymerisation `http://purl.obolibrary.org/obo/MOP_0000631`
  * cyclization `http://purl.obolibrary.org/obo/MOP_0000561`
  * diblock copolymerisation `http://purl.obolibrary.org/obo/MOP_0000700`
  * enolization `http://purl.obolibrary.org/obo/MOP_0000672`
  * epoxidation `http://purl.obolibrary.org/obo/MOP_0000671`
  * graft copolymerisation `http://purl.obolibrary.org/obo/MOP_0000694`
  * graft polymerisation `http://purl.obolibrary.org/obo/MOP_0000702`
  * homopolymerisation `http://purl.obolibrary.org/obo/MOP_0000633`
  * ionic copolymerisation `http://purl.obolibrary.org/obo/MOP_0000647`
  * living copolymerisation `http://purl.obolibrary.org/obo/MOP_0000649`
  * nitrile reduction `http://purl.obolibrary.org/obo/MOP_0000711`
  * periodic copolymerisation `http://purl.obolibrary.org/obo/MOP_0000695`
  * polymerisation `http://purl.obolibrary.org/obo/MOP_0000629`
  * radical copolymerisation `http://purl.obolibrary.org/obo/MOP_0000648`
  * radical formation `http://purl.obolibrary.org/obo/MOP_0000786`
  * random copolymerisation `http://purl.obolibrary.org/obo/MOP_0000696`
  * ring-opening copolymerisation `http://purl.obolibrary.org/obo/MOP_0000688`
  * statistical copolymerisation `http://purl.obolibrary.org/obo/MOP_0000697`
  * stereospecific copolymerisation `http://purl.obolibrary.org/obo/MOP_0000698`
  * triblock copolymerisation `http://purl.obolibrary.org/obo/MOP_0000701`

* `[BFO_0000078](http://purl.obolibrary.org/obo/BFO_0000078)` → `[RO_0004009](http://purl.obolibrary.org/obo/RO_0004009)`
  * enolization `http://purl.obolibrary.org/obo/MOP_0000672`
  * nitrile reduction `http://purl.obolibrary.org/obo/MOP_0000711`
  * olefin reduction `http://purl.obolibrary.org/obo/MOP_0000712`

* `[BFO_0000162](http://purl.obolibrary.org/obo/BFO_0000162)` → `[RO_0000091](http://purl.obolibrary.org/obo/RO_0000091)`
  * enolisable carbonyl compound `http://purl.obolibrary.org/obo/MOP_0000718`


#### Moving MOP terms from pre-ODK RXNO to MOP
The MOP terms that were only defined in the pre-ODK `rxno.owl`file (see [High Severity: MOP terms only defined in `rxno.owl`](#mop-only-in-rxno)) were copied to MOP during the ODK migration. Since the diff compares only the pre-ODK with the post ODK MOP, these terms appear as added classes in the diff.

**Affected Terms:**
* O-dealkylation `http://purl.obolibrary.org/obo/MOP_0000719`
* P-alkylation `http://purl.obolibrary.org/obo/MOP_0006369`
* [2+2+1] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000720`
* dealcoholative condensation `http://purl.obolibrary.org/obo/MOP_0000825`
* decarboxylation `http://purl.obolibrary.org/obo/MOP_0001713`
* decarboxylative condensation `http://purl.obolibrary.org/obo/MOP_0000722`
* deethanolative condensation `http://purl.obolibrary.org/obo/MOP_0000723`
* dienophilicity `http://purl.obolibrary.org/obo/MOP_0000826`
* enolisability `http://purl.obolibrary.org/obo/MOP_0000717`
* enolisable carbonyl compound `http://purl.obolibrary.org/obo/MOP_0000718`

#### Resolving IRI clashes
As described in the [IRI clashes: One IRI, two different reactions](#iri-clashes) section, Some MOP IRIs were used for *different reactions* in `rxno.owl` vs. `mop.owl`. These appear as added classes in the diff, if the term was only defined in the pre-ODK RXNO release and thus a new IRI had to be minted for them in MOP. If the reaction was already defined in MOP, any relevant axioms that were only defined in pre-ODK RXNO under the wrong IRI were copied over to MOP, and thus appear as added in the diff. Also, comments were added on all the involved IRIs for provenance recording. The textual definition change on `enolization` was done as a deliberate improvement during this step to make it Aristotelian.

**Affected Terms:**
* previously only defined in RXNO
  * [3+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0006371`
  * carbonyl oxidation to alkyne `http://purl.obolibrary.org/obo/MOP_0006370`
  * oxidation state `http://purl.obolibrary.org/obo/MOP_0005561`
* already present in MOP
  * carboxylation `http://purl.obolibrary.org/obo/MOP_0000591`
  * enolization `http://purl.obolibrary.org/obo/MOP_0000672`
* comments added on pre-existing MOP terms
  * acetylene reduction `http://purl.obolibrary.org/obo/MOP_0000713` 
  * cyclotrimerization `http://purl.obolibrary.org/obo/MOP_0000716`
  * ligand exchange reaction `http://purl.obolibrary.org/obo/MOP_0000714`
  * olefin reduction `http://purl.obolibrary.org/obo/MOP_0000712`
  * trimerization `http://purl.obolibrary.org/obo/MOP_0000715`


#### Resolving `alkene oxidative cleavage` duplicate
As described in [Duplicates: One reaction, two IRIs](#duplicates), `alkene oxidative cleavage` was defined in the RXNO **and** MOP namespace. The RXNO term needed to be deprecated in favor of the MOP term. The more precise textual definition provided on the RXNO version was used instead of the previous MOP one. The MOP terms that were subsumed in RXNO under `alkene oxidative cleavage` but under its parents in MOP (for more detail see [Duplicates: One reaction, two IRIs](#duplicates)) were not changed in the ODK migration. Instead, [this issue](https://github.com/rsc-ontology/mop/issues/27) was opened to review them.

**Affected Terms:**
* alkene oxidative cleavage `http://purl.obolibrary.org/obo/MOP_0000708`


#### Resolving Logical Axiom differences
As described in the [Medium Severity: Different logical axioms on shared MOP terms](#axiom-diffs) section, there were differences regarding the logical axioms  asserted on MOP terms defined in RXNO and MOP. These were resolved and reviewed by Colin Batchelor.

**Affected Terms:**
* SubClassOf expressions added, because they were were only defined in pre-ODK RXNO (including deletion of rdfs:comment placeholder)
  * [4+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000565`
  * alcohol oxidation `http://purl.obolibrary.org/obo/MOP_0000572`
  * aldehyde reduction `http://purl.obolibrary.org/obo/MOP_0000571`
  * arylation `http://purl.obolibrary.org/obo/MOP_0000411`
  * ester reduction `http://purl.obolibrary.org/obo/MOP_0000577`
  * ester reduction to aldehyde `http://purl.obolibrary.org/obo/MOP_0000578`
  * ester reduction to primary alcohol `http://purl.obolibrary.org/obo/MOP_0000579`
  * formylation `http://purl.obolibrary.org/obo/MOP_0000003`
  * ketone reduction `http://purl.obolibrary.org/obo/MOP_0000580`
  * nitration `http://purl.obolibrary.org/obo/MOP_0000556`
  * primary alcohol oxidation to aldehyde `http://purl.obolibrary.org/obo/MOP_0000573`
  * primary alkene oxidation to carboxylic acid and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000584`
  * quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588`
  * secondary alcohol oxidation to ketone `http://purl.obolibrary.org/obo/MOP_0000574`
  * secondary terminal alkene oxidation to ketone and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000587`
  * secondary, non-terminal alkene oxidation to aldehydes `http://purl.obolibrary.org/obo/MOP_0000586`
  * tertiary alkene oxidation to carboxylic acid and ketone `http://purl.obolibrary.org/obo/MOP_0000585`
* Change of parent class (including removal of grandparent classes or dropping of equivalentTo axioms)
  * amine oxidation `http://purl.obolibrary.org/obo/MOP_0000575` 
  * carboacylation `http://purl.obolibrary.org/obo/MOP_0000028`
  * epoxidation `http://purl.obolibrary.org/obo/MOP_0000671`
* Swapping of more general CHEBI term ('cyclic organic group - CHEBI_52845' → 'cyclic compound - CHEBI_33595.') 
  * cyclization `http://purl.obolibrary.org/obo/MOP_0000561`


#### Dropping Grandparent Assertion on 'carboacylation'
During the review of MOP terms for the ODK migration it became apparent that `carboacylation` (`http://purl.obolibrary.org/obo/MOP_0000028`) had an additional SubClassOf axiom that subsumed it additionally under its grandparent `formation of covalent bond with carbon centre`. This redundant axiom was dropped, as it is already inherited from the parent `acylation`.


### Metadata Changes

#### Ontology Annotation Changes
The ontology annotations of MOP were updated in the course of the ODK migration. This included switching from using DC Elements predicates to DC Terms ones for `title` and `description` (see https://github.com/rsc-ontology/mop/pull/2), using a full title instead of just the MOP acronym (see https://github.com/rsc-ontology/mop/pull/25), adding Colin Batchelor as creator and the NFDI4Chem team as contributors and replacing the outdated versioning info in form of a `dc:date` and `hasOBOFormatVersion`annotation with the `owl:versionInfo` autogenerated by ODK (see **TODO:add PR link when https://github.com/rsc-ontology/mop/tree/Fix-ODK-migration-errors is merged**). The fact that we are also comparing base releases (versions that only declare external terms but not further define them), explains also the additional DC Elements `type` annotation. It is added automatically by ODK for any base release, whereas it was not added for the pre-ODK MOP version, because we manually created this base release with ROBOT in Step 2 of the previous MOP vs RXNO comparison.

#### Resolving Annotation Differences
As described in the [Metadata Changes (no logical impact)](#metadata-changes) section above, several MOP terms had different annotations. Those that were only provided or better in the pre-ODK RXNO, were added or changed.

**Affected Terms:**
* Annotations copied over from pre-ODK RXNO
    * condensation reaction `http://purl.obolibrary.org/obo/MOP_0000627`
    * dehydration reaction `http://purl.obolibrary.org/obo/MOP_0000628`
* DC Elements to DC Terms predicate swap
  * cyclotrimerization `http://purl.obolibrary.org/obo/MOP_0000716`
* rdfs:label changes due to review by Colin Batchelor (see review Gsheet table)
  * primary alcohol oxidation to aldehyde `http://purl.obolibrary.org/obo/MOP_0000573` 
  * secondary alcohol oxidation to ketone `http://purl.obolibrary.org/obo/MOP_0000574`
* additional rdfs:comments for reaction participants
  * sulfonylation `http://purl.obolibrary.org/obo/MOP_0000539`

#### Changed Annotations
On some term the annotations were improved in the course of the ODK migration, by using better predicates or sources.
**Affected Terms:**
  * trimerization `http://purl.obolibrary.org/obo/MOP_0000715`