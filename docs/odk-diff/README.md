# Systematic Comparison of MOP classes defined in RXNO and MOP before the ODK migration
[author](http://purl.org/dc/terms/creator): [Philip Strömert](https://orcid.org/0000-0002-1595-3213) | [created on](http://purl.org/dc/terms/created): 01.10.2026

## Background
Before the migration to an Ontology Development Kit (ODK) based workflow, the two ontologies RXNO and MOP where maintained within the same GitHub repository: https://github.com/rsc-ontology/rxno. Their release workflow was such that edits were directly made to the release files `rxno.owl` respectively `mop.owl`. So these two files where the only source of truth/origin for both ontologies, compared to an ODK based workflow, where release files are being generated automatically from a separate editor OWl file and/or different component files, such as OWL files and/or ROBOT TSV files.

In 2021, as a result of the [Ontologies4Chem: the landscape of ontologies in chemistry](https://doi.org/10.1515/pac-2021-2007) overview paper, the NFDI4Chem team started to improve both ontologies by filing issues and pull requests in this single RXNO repository. One of the first suggestions was to properly import MOP in RXNO using `owl:import` instead of (re)defining the MOP classes within the `rxno.owl` file (see [issue #27](https://github.com/rsc-ontology/rxno/issues/27)). 

While trying to resolve this issues in 2022, it became apparent that there were differences regarding the MOP classes defined in the `rxno.owl` and `mop.owl` file. This lead to a state in which :
* some MOP classes were exactly the same in both files,
* some MOP classes had only been defined in `rxno.owl` and were thus missing in `mop.owl`,
* and some MOP classes with the same PURL in both files
  * represent the same reaction but have different axioms or annotations
  * represent different reactions in MOP and RXNO and thus have clashing PURLs

Hence, to resolve this issue and to allow a proper import of MOP within RXNO in an ODK based repository, a systematic analysis was needed to identify: which MOP classes can be safely cut from RXNO and copied to MOP, which need to be split into two classes to avoid PURL clashes, and which need to be vetted regarding their differing axiomatization. The following sections describe how this systematic analysis was conducted.

## Creating and Archival Release of RXNO and MOP
Before the migration to ODK, both ontologies were only published directly in the code base of the GitHub repository. No GitHub release were ever created. Thus, we created a pre-ODK release and tag to allow pointing to these versions of RXNO and MOP, see: https://github.com/rsc-ontology/rxno/tree/pre-odk-release

## Using ROBOT to identify the differences between the MOP terms defined in the pre ODK `rxno.owl` and `mop.owl` files. 
@TODO: Add a paragraph about the initial workflow using ROBOT to convert RXNO and MOP to tsv first and then load into MS EXCEL to reconcile, with links to now outdated PR and files and such

### Using [ROBOT diff](https://robot.obolibrary.org/diff) to Produce a Systematic Diff 
1. Download the pre-ODK RXNO and MOP release files:
    ```shell
    curl -o rxno_pre_ODK.owl https://raw.githubusercontent.com/rsc-ontology/rxno/refs/tags/pre-odk-release/rxno.owl && \
    curl -o mop_pre_ODK.owl https://raw.githubusercontent.com/rsc-ontology/rxno/refs/tags/pre-odk-release/mop.owl
    ```
2. Use the files downloaded in step 1 in ROBOT, to produce a ontologies that only contain the MOP classes and only references all external ontology terms via their PURL:
    ```shell
    robot remove -i mop_pre_ODK.owl --base-iri MOP --axioms external --preserve-structure false \
      --trim false convert -f ofn \
      -o mop_base_pre_ODK.owl  && \
    robot remove -i rxno_pre_ODK.owl --base-iri MOP --axioms external --preserve-structure false \
      --trim false convert -f ofn \
      -o mop_base_in_rxno_pre_ODK.owl
    ```
3. Use the `mop_base_in_rxno_pre_ODK.owl` file created in step 2 to transform it into a term list that only contains the MOP terms defined in this RXNO version:
    ```shell
    robot export --input mop_base_in_rxno_pre_ODK.owl --header "ID|LABEL" --include "classes properties" --export mop_base_in_rxno_pre_ODK.csv && \
    grep '^MOP:' mop_base_in_rxno_pre_ODK.csv | \
    sed 's/,/\t# /' > mop_in_rxno_terms.txt && \
    rm mop_base_in_rxno_pre_ODK.csv
    ```
4. Use the `mop_in_rxno_terms.txt` file from step 3 to filter the `mop_base_pre_ODK.owl` from step 1 into an MOP module that contains only those classes that were also present in `rxno_pre_ODK.owl`. We do this as a preparation to provide a slimmer diff between the pre ODK MOP and RXNO versions, which only focuses on the MOP terms defined in the pre ODK RXNO. 
    ```shell
    robot extract -i mop_base_pre_ODK.owl --method BOT \
      --term-file mop_in_rxno_terms.txt --copy-ontology-annotations true \
      convert -f ofn \
      -o mop_base_pre_ODK_slim.owl
    ```
5. Make diff to see the changes of MOP classes in RXNO and MOP before ODK migration
    ```shell
    robot diff --left mop_base_in_rxno_pre_ODK.owl --right mop_base_pre_ODK_slim.owl -f markdown -o mop_in_rxno_pre_ODK_diff.md
    ```
## Analysing the ROBOT diff
This section analyses the differences between the MOP terms defined in the pre ODK `rxno.owl` and `mop.owl` files made visible by the ROBOT derived markdown document. In there, the deletion of the ontology annotations are expected, sine we are comparing the MOP part wrongly defined in the `rxno.owl` file against the `mop.owl` file. The remaining differences are  systematic and can be grouped as follows: 

### 1. Deletion of external terms
External terms, such as [RXNO_0000344](http://purl.obolibrary.org/obo/RXNO_0000344), [BFO_0000016](http://purl.obolibrary.org/obo/BFO_0000016) or [CHEBI_15734](http://purl.obolibrary.org/obo/CHEBI_15734), were deleted because they were not used in any axioms within the `mop.owl` file on those MOP terms that were also wrongly defined in the `rxno.owl` file.

### 2. Addition of external terms
External terms, such as [BFO_0000117](http://purl.obolibrary.org/obo/BFO_0000117) or [CHEBI_22221](http://purl.obolibrary.org/obo/CHEBI_22221), were added in axioms within the `mop.owl` file on those MOP terms that were also wrongly defined in the `rxno.owl` file.

### 3. Deletion of "RXNO" Namespace annotation
The namespace annotation `RXNO` is not present in the `mop.owl` file because it does not make any sense, since the MOP terms already are declared to be within the MOP namespace via their base IRI `http://purl.obolibrary.org/obo/MOP_`. Hence, this type of diff can savely be ignored. Here is an example copied from the ROBOT diff:
#### EXAMPLE 
substitution reaction `http://purl.obolibrary.org/obo/MOP_0000790`
##### Removed
- [substitution reaction](http://purl.obolibrary.org/obo/MOP_0000790) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- 
### 4. Deletion of `http://purl.obolibrary.org/obo/rxno.obo#has_part`
In the `rxno.owl` file, the relation `has_part` (http://purl.obolibrary.org/obo/rxno.obo#has_part) was defined and used in SubClassOf axioms of MOP terms (`Brønsted base catalysis` & `Lewis adduct formation`), whereas in the `mop.owl` file the equivalent BFO relation (http://purl.obolibrary.org/obo/BFO_0000117) was used in these SubClassOf axioms. This reason explains the differences on:

* decarboxylative condensation `http://purl.obolibrary.org/obo/MOP_0000722`
* Brønsted base catalysis `http://purl.obolibrary.org/obo/MOP_0000736`
* Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000739`

Here is an example copied from the ROBOT diff:
#### EXAMPLE 
Brønsted base catalysis `http://purl.obolibrary.org/obo/MOP_0000736`
##### Removed
- [Brønsted base catalysis](http://purl.obolibrary.org/obo/MOP_0000736) SubClassOf [has_part](http://purl.obolibrary.org/obo/rxno.obo#has_part) some [accepting a hydron](http://purl.obolibrary.org/obo/MOP_0000737)
##### Added
- [Brønsted base catalysis](http://purl.obolibrary.org/obo/MOP_0000736) SubClassOf [BFO_0000117](http://purl.obolibrary.org/obo/BFO_0000117) some [accepting a hydron](http://purl.obolibrary.org/obo/MOP_0000737)

### 5. Alternaive IDs
Two MOP terms (`http://purl.obolibrary.org/obo/MOP_0002524` & `http://purl.obolibrary.org/obo/MOP_0003524`) have and additional `hasAlternativeId`(http://www.geneontology.org/formats/oboInOwl#hasAlternativeId) annotation axiom that points to the ID of the deprecated successor MOP term and that was only present in the `mop.owl` file. While two terms (`http://purl.obolibrary.org/obo/MOP_0000627` & `http://purl.obolibrary.org/obo/MOP_0000562`) had an alternative ID annotation in `rxno.owl` that was not present in the `mop.owl`.

### 6. Removed MOP Terms
Some MOP terms where only defined in the `rxno.owl` file and are thus showing up in the diff as being removed from the `mop.owl` file (see example below). This reason explains the differences on:

* O-dealkylation `http://purl.obolibrary.org/obo/MOP_0000719`
* P-alkylation `http://purl.obolibrary.org/obo/MOP_0006369`
* [2+2+1] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000720`
* [4+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000565`
* alcohol oxidation `http://purl.obolibrary.org/obo/MOP_0000572`
* aldehyde reduction `http://purl.obolibrary.org/obo/MOP_0000571`
* alkene oxidation `http://purl.obolibrary.org/obo/MOP_0000581
* dealkylation `http://purl.obolibrary.org/obo/MOP_0001369`
* decarboxylative condensation `http://purl.obolibrary.org/obo/MOP_0000722`
* deethanolative condensation `http://purl.obolibrary.org/obo/MOP_0000723`
* dealcoholative condensation `http://purl.obolibrary.org/obo/MOP_0000825`
* decarboxylation `http://purl.obolibrary.org/obo/MOP_0001713`
* dienophilicity `http://purl.obolibrary.org/obo/MOP_0000826`
* enolisability `http://purl.obolibrary.org/obo/MOP_0000717`
* enolisable carbonyl compound `http://purl.obolibrary.org/obo/MOP_0000718`

#### EXAMPLE 
O-dealkylation `http://purl.obolibrary.org/obo/MOP_0000719`
##### Removed
- Class: [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719)
- [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [id](http://www.geneontology.org/formats/oboInOwl#id) "MOP:0000719"
- [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "Breaking of a covalent bond between an alkyl group and an oxygen atom." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:xp"
- [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) [label](http://www.w3.org/2000/01/rdf-schema#label) "O-dealkylation"
- [O-dealkylation](http://purl.obolibrary.org/obo/MOP_0000719) SubClassOf [dealkylation](http://purl.obolibrary.org/obo/MOP_0001369) 

### 7. IRI Clashes 
Some MOP terms in the `rxno.owl` files are using the same IRI as another MOP term in the `mop.owl` file. These IRI clashes show up in the diff not via the removal of the class itself, but via the removal and new addition of required annotations, such as rdfs:label and definition (IAO:0000115). Also, the linked class reference in the diff uses the wrong label for the term from the `mop.owl` (see `[3+2] cycloaddition` example below). These terms need to be discussed with RXNO/MOP maintainer and domain expert Colin Batchelor. This reason explains the differences on:

* [3+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000715`
* carbonyl oxidation to alkyne `http://purl.obolibrary.org/obo/MOP_0000714`
* carboxylation `http://purl.obolibrary.org/obo/MOP_0000713`
* enolisation `http://purl.obolibrary.org/obo/MOP_0000716`
* oxidation state `http://purl.obolibrary.org/obo/MOP_0000712`

#### Example
[3+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000715`
##### Removed
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A cycloaddition in which one participant contributes three electrons and the other participant contributes two electrons to the transformation of reactants to products." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [label](http://www.w3.org/2000/01/rdf-schema#label) "[3+2] cycloaddition"
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) SubClassOf [cycloaddition](http://purl.obolibrary.org/obo/MOP_0000562) 
##### Added
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [IAO_0000116](http://purl.obolibrary.org/obo/IAO_0000116) "The GO class &quot;protein trimerization&quot; (https://purl.obolibrary.org/obo/GO_0070206) would actually be a subclass of &quot;trimerization&quot; according to the definition provided here. However, for the reason of not wanting to inject this subclassOf axiom into GO, we decided to not assert this in MOP as of yet. Further discussion is needed with the GO and/or OBO team to see how this GO class can be integrated."
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [creator](http://purl.org/dc/elements/1.1/creator) [0000-0002-4378-6061](http://orcid.org/0000-0002-4378-6061)
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [date](http://purl.org/dc/elements/1.1/date) "2022-02-01T09:40:48Z"^^[dateTime](http://www.w3.org/2001/XMLSchema#dateTime)
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "The formation of a trimer from three molecular subunits." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://en.wiktionary.org/wiki/trimerization"^^[anyURI](http://www.w3.org/2001/XMLSchema#anyURI)
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [label](http://www.w3.org/2000/01/rdf-schema#label) "trimerization"@en
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [seeAlso](http://www.w3.org/2000/01/rdf-schema#seeAlso) [45](https://github.com/rsc-ontologies/rxno/issues/45)
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) [seeAlso](http://www.w3.org/2000/01/rdf-schema#seeAlso) [issuecomment-1124151946](https://github.com/rsc-ontologies/rxno/pull/46#issuecomment-1124151946)
- [[3+2] cycloaddition](http://purl.obolibrary.org/obo/MOP_0000715) SubClassOf [molecular process](http://purl.obolibrary.org/obo/MOP_0000543) 

### 8. Missing Annotations on MOP terms in `rxno.owl` 
Some MOP terms in the `rxno.owl` file have less or less specific annotations (see `[3,3]-sigmatropic rearrangement` example below). This reason explains the differences on:

* [3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
* accepting a hydron `http://purl.obolibrary.org/obo/MOP_0000737`
* accepting an electron pair in Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000733`
* acylation `http://purl.obolibrary.org/obo/MOP_0000479`
* base catalysis `http://purl.obolibrary.org/obo/MOP_0000731`
* condensation reaction `http://purl.obolibrary.org/obo/MOP_0000627`
* cyclization `http://purl.obolibrary.org/obo/MOP_0000561`
* cycloaddition `http://purl.obolibrary.org/obo/MOP_0000562`
* cycloelimination `http://purl.obolibrary.org/obo/MOP_0000564`
* electron attachment `http://purl.obolibrary.org/obo/MOP_0000570`
* electron pair donation in Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000738`
* elimination reaction `http://purl.obolibrary.org/obo/MOP_0000656`
* hydron donation `http://purl.obolibrary.org/obo/MOP_0000732`
* methylation `http://purl.obolibrary.org/obo/MOP_0000364`
* oxidation `http://purl.obolibrary.org/obo/MOP_0000568`
* reduction `http://purl.obolibrary.org/obo/MOP_0000569`
* solvolysis `http://purl.obolibrary.org/obo/MOP_0000618`

#### Example
[3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
##### Added
- [[3,3]-sigmatropic rearrangement](http://purl.obolibrary.org/obo/MOP_0000721) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A sigmatropic rearrangement where the intermediate consists of two three-electron fragments." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"

### 9. Different labels on MOP terms in `mop.owl` 
Some MOP terms in the `rxno.owl` file different labels than in the `mop.owl` file (see `molecular dehydration reaction` example below). This reason explains the differences on:

* molecular dehydration reaction `http://purl.obolibrary.org/obo/MOP_0000628`
* secondary alcohol oxidation to ketone `http://purl.obolibrary.org/obo/MOP_0000574`

#### molecular dehydration reaction `http://purl.obolibrary.org/obo/MOP_0000628`
##### Removed
- [molecular dehydration reaction](http://purl.obolibrary.org/obo/MOP_0000628) [label](http://www.w3.org/2000/01/rdf-schema#label) "molecular dehydration reaction"
##### Added
- [molecular dehydration reaction](http://purl.obolibrary.org/obo/MOP_0000628) [label](http://www.w3.org/2000/01/rdf-schema#label) "dehydration reaction" 

### 10. Different SubclassOf Axioms of MOP terms in `rxno.owl` 
Some MOP terms in the `rxno.owl` file have a different SubclassOf axioms than in the `mop.owl` file (see `acylation` example below). Sometimes there are differences in the parent class itself and sometimes the SubclassOf expressions were either only declared in `rxno.owl` or in `mop.owl`. This reason explains the differences on:

* [3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
* acid catalysis `http://purl.obolibrary.org/obo/MOP_0000740`
* acylation `http://purl.obolibrary.org/obo/MOP_0000479`
* allylation `http://purl.obolibrary.org/obo/MOP_0000422`
* amine oxidation `http://purl.obolibrary.org/obo/MOP_0000575`
* arylation `http://purl.obolibrary.org/obo/MOP_0000411`
* breaking of covalent bond with group `http://purl.obolibrary.org/obo/MOP_0000567`
* cyclization `http://purl.obolibrary.org/obo/MOP_0000561`
* cycloaddition `http://purl.obolibrary.org/obo/MOP_0000562`
* epoxidation `http://purl.obolibrary.org/obo/MOP_0000671`
* ester reduction `http://purl.obolibrary.org/obo/MOP_0000577`
* ester reduction to aldehyde `http://purl.obolibrary.org/obo/MOP_0000578`
* ester reduction to primary alcohol `http://purl.obolibrary.org/obo/MOP_0000579`
* formylation `http://purl.obolibrary.org/obo/MOP_0000003`
* ketone reduction `http://purl.obolibrary.org/obo/MOP_0000580`
* halogenation `http://purl.obolibrary.org/obo/MOP_0000550`
* N-arylation `http://purl.obolibrary.org/obo/MOP_0002411`
* nitration `http://purl.obolibrary.org/obo/MOP_0000556`
* organylation `http://purl.obolibrary.org/obo/MOP_0000458`
* primary alcohol oxidation to aldehyde `http://purl.obolibrary.org/obo/MOP_0000573`
* primary alkene oxidation to carboxylic acid and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000584`
* quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588
* secondary alcohol oxidation to ketone `http://purl.obolibrary.org/obo/MOP_0000574`
* secondary, non-terminal alkene oxidation to aldehydes `http://purl.obolibrary.org/obo/MOP_0000586`
* silylation `http://purl.obolibrary.org/obo/MOP_0000339`
* sulfonylation `http://purl.obolibrary.org/obo/MOP_0000539`
* tertiary alkene oxidation to carboxylic acid and ketone `http://purl.obolibrary.org/obo/MOP_0000585`

#### acylation `http://purl.obolibrary.org/obo/MOP_0000479`
##### Removed
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) [comment](http://www.w3.org/2000/01/rdf-schema#comment) "has_participant: CHEBI:22221"
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) SubClassOf [organylation](http://purl.obolibrary.org/obo/MOP_0000458)
##### Added
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) [comment](http://www.w3.org/2000/01/rdf-schema#comment) "you probably want &quot;carboacylation&quot;, MOP:0000028, here"
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) SubClassOf [formation of covalent bond with carbon centre](http://purl.obolibrary.org/obo/MOP_0000800)
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) SubClassOf [BFO_0000057](http://purl.obolibrary.org/obo/BFO_0000057) some [CHEBI_22221](http://purl.obolibrary.org/obo/CHEBI_22221) 


### 11. Added Classes due to MOP Base Module Extraction  
Some MOP terms in the `mop.owl` file got added, due to being the parent or grandparent of one of those terms that was defined in both files. This addition came from Step 4. of building the MOP module that was used in the comparison. This reason explains the differences on:
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

### 12. Dublicate Terms in RXNO and MOP
The term `alkene oxidative cleavage` was defined in `rxno.owl`with http://purl.obolibrary.org/obo/RXNO_0000344 and in `mop.owl` with http://purl.obolibrary.org/obo/MOP_0000708. The RXNO duplicate was used as parent for MOP terms defined in the `rxno.owl`, whereas the MOP duplicate or another MOP term was used as parent for these terms in `mop.owl` (see `alkene ozonolysis`example below). This reason explains the differences on:

* alkene ozonolysis `http://purl.obolibrary.org/obo/MOP_0000583`
* quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588`
* secondary, non-terminal alkene oxidation to aldehydes `http://purl.obolibrary.org/obo/MOP_0000586`
* secondary terminal alkene oxidation to ketone and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000587`
* tertiary alkene oxidation to carboxylic acid and ketone `http://purl.obolibrary.org/obo/MOP_0000585`

#### alkene ozonolysis `http://purl.obolibrary.org/obo/MOP_0000583`
##### Removed
- [alkene ozonolysis](http://purl.obolibrary.org/obo/MOP_0000583) SubClassOf [RXNO_0000344](http://purl.obolibrary.org/obo/RXNO_0000344)
##### Added
- [alkene ozonolysis](http://purl.obolibrary.org/obo/MOP_0000583) SubClassOf [alkene oxidative cleavage](http://purl.obolibrary.org/obo/MOP_0000708) 

### 13. Pseudodiff
For `allylic rearrangement` ROBOT calculated a diff, although there are no changes at all, as can be seen below:
#### allylic rearrangement `http://purl.obolibrary.org/obo/MOP_0000792`
##### Removed
- [allylic rearrangement](http://purl.obolibrary.org/obo/MOP_0000792) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [allylic rearrangement](http://purl.obolibrary.org/obo/MOP_0000792) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A substitution reaction in an allylic system with concomitant migration of the allyl double bond." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0002-4077-4719"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:db"
##### Added
- [allylic rearrangement](http://purl.obolibrary.org/obo/MOP_0000792) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A substitution reaction in an allylic system with concomitant migration of the allyl double bond." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:db"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:cc" 

## Using ROBOT to produce a diff comparing MOP before and after the ODK Migration
In this step we produce a diff between the pre-ODK MOP version and the post-ODK version to make sure the ODK migration did not produce any unaccounted changes of MOP. 
The MOP term overlap analysed in the previous section was partially addressed during the MOP migration by copying the MOP terms only defined in RXNO over to MOP, resolving the IRI clashes and some of the differences in SubclassOf axioms or annotations together with Colin Batchelor. 
To produce a minimal diff between the pre- and post-ODK MOP version, we decided to merge only those MOP terms defined in the pre-ODK base RXNO with the pre ODK base version of MOP that were not previously defined in the `mop.owl`. Hence, the diff will show changes for those terms defined in both OWL files that are due to the reasons listed in the previous step.

### 1. Get MOP Terms only defined in pre-ODK RXNO
We used the terms identified in the previous step `6. Removed MOP Terms` to manually produce the `mop_only_in_rxno_terms.txt` file, which we then used to build the `mop_only_in_rxno_terms.owl` module that contains MOP terms only defined in RXNO, via ROBOT extract like this:
```shell
robot extract -i mop_base_in_rxno_pre_ODK.owl --method BOT \
  --term-file mop_only_in_rxno_terms.txt --copy-ontology-annotations false \
  -o mop_only_in_rxno.owl
```

### 2. Merge Pre ODK MOP versions
We used ROBOT to merge the `mop_only_in_rxno_terms.owl` module with the pre ODK base release of MOP (`mop_base_pre_ODK.owl`) that we created earlier.
```shell
robot merge -i mop_base_pre_ODK.owl -i mop_only_in_rxno_base.owl \
  --include-annotations true -o mop_base_pre_ODK_complete.owl
```

### 3. Download the post-ODK MOP release file:
To compare the pre-ODK MOP version we need to download the first release made after the ODK migration. We're using the base release to exclude imported terms.
```shell
curl -o mop_base_post_ODK.owl https://raw.githubusercontent.com/rsc-ontology/mop/refs/heads/main/mop-base.owl
```

### 4. Make diff to see the changes after the ODK migration
```shell
robot diff --left mop_base_pre_ODK_complete.owl --right mop_base_post_ODK.owl -f markdown -o mop_ODK_diff.md
```

## Analysing ROBOT diff comparing MOP before and after the ODK Migration

### Ontology annotation diff
The DC Elements predicates were swapped with the newer and OBO preferred DCTerms equivalents. The `owl:versionIRI` and `dce:type` annotations were added automatically by the ODK in the release artifact build process.

### Swapping of BFO relations
Since the OBO Foundry ontologies are to use the Relations Ontology RO for their relations and only BFO classes for the upper level, we swapped all BFO object properties for their RO equivalents in https://github.com/rsc-ontology/mop/issues/3. This accounts for the SubclassOf changes on:
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
* BFO_0000050 `http://purl.obolibrary.org/obo/BFO_0000050`
* 