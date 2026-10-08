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

### Using ROBOT diff to produce a systematic diff
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
      --term-file mop_in_rxno_terms.txt --cop\
      convert -f ofn \
      -o mop_base_pre_ODK_slim.owl
    ```
5. Make diff to see the changes of MOP classes in RXNO and MOP before ODK migration
    ```shell
    robot diff --left mop_base_in_rxno_pre_ODK.owl --right mop_base_pre_ODK_slim.owl -f markdown -o mop_in_rxno_pre_ODK_diff.md
    ```
## Analysing the ROBOT diff showing the differences between the MOP terms defined in the pre ODK `rxno.owl` and `mop.owl` files. 
The deletion of the ontology annotations in the diff are expected, sine we are comparing the MOP part wrongly defined in the `rxno.owl` file against the `mop.owl` file.
### Reoccuring differences
1. External terms, such as [RXNO_0000344](http://purl.obolibrary.org/obo/RXNO_0000344), [BFO_0000016](http://purl.obolibrary.org/obo/BFO_0000016) or [CHEBI_15734](http://purl.obolibrary.org/obo/CHEBI_15734), were deleted because they were not used in any axioms within the `mop.owl` file. 
2. External terms, such as [BFO_0000117](http://purl.obolibrary.org/obo/BFO_0000117) or [CHEBI_22221](http://purl.obolibrary.org/obo/CHEBI_22221), were added in axioms within the `mop.owl` file on MOP terms wrongly defined in the `rxno.owl` file.
3. The annotation `[hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"` is not present in the `mop.owl` file because it doesn't make sense, since they have the MOP base IRI `http://purl.obolibrary.org/obo/MOP_` and thus are part of the MOP namespace. This happened on all MOP terms defined wrongly in the `rxno.owl` file (see `Brønsted base catalysis` example below).
4. In the `rxno.owl` file the relation `has_part` (http://purl.obolibrary.org/obo/rxno.obo#has_part) was defined and used in SubClassOf axioms, whereas in the `mop.owl` file the equivalent BFO relation (http://purl.obolibrary.org/obo/BFO_0000117) was used in these SubClassOf axioms (see `Brønsted base catalysis` example below). This reason explains the differences on:
    * Brønsted base catalysis `http://purl.obolibrary.org/obo/MOP_0000736`
    * Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000739`
5. Some MOP terms in the `rxno.owl` file had SubClassOf axioms defined that where not asserted on the same terms in the `mop.owl` file (see `N-arylation` example below). Yet, sometimes the terms in the `mop.owl` file would have a rdfs:comment indicating this missing axiomatization. This reason explains the differences on:
    * N-arylation `http://purl.obolibrary.org/obo/MOP_0002411`
    * arylation `http://purl.obolibrary.org/obo/MOP_0000411`
    * ester reduction `http://purl.obolibrary.org/obo/MOP_0000577`
    * ester reduction to aldehyde `http://purl.obolibrary.org/obo/MOP_0000578`
    * ester reduction to primary alcohol `http://purl.obolibrary.org/obo/MOP_0000579`
6. Some MOP terms would have and additional `hasAlternativeId`(http://www.geneontology.org/formats/oboInOwl#hasAlternativeId) annotation axiom that points to the ID of the deprecated successor MOP term and that was only present in the `mop.owl` file. This reason explains the differences on:
   * N-sulfonylation `http://purl.obolibrary.org/obo/MOP_0002524`
   * O-sulfonylation `http://purl.obolibrary.org/obo/MOP_0003524`
7. Some MOP terms where only defined in the `rxno.owl` file and are thus showing up in the diff as being removed from the `mop.owl` file (see `alkene oxidation to 1,2-diol` example below). This reason explains the differences on:
   * O-dealkylation `http://purl.obolibrary.org/obo/MOP_0000719`
   * P-alkylation `http://purl.obolibrary.org/obo/MOP_0006369`
   * [2+2+1] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000720`
   * [4+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000565`
   * alcohol oxidation `http://purl.obolibrary.org/obo/MOP_0000572`
   * aldehyde reduction `http://purl.obolibrary.org/obo/MOP_0000571`
   * alkene oxidation `http://purl.obolibrary.org/obo/MOP_0000581`
   * alkene oxidation to 1,2-diol `http://purl.obolibrary.org/obo/MOP_0000582`
   * dealcoholative condensation `http://purl.obolibrary.org/obo/MOP_0000825`
   * decarboxylation `http://purl.obolibrary.org/obo/MOP_0001713`
   * dienophilicity `http://purl.obolibrary.org/obo/MOP_0000826`
   * enolisability `http://purl.obolibrary.org/obo/MOP_0000717`
   * enolisable carbonyl compound `http://purl.obolibrary.org/obo/MOP_0000718`
8. Some MOP terms in the `rxno.owl` files are using the same IRI as another MOP term in the `mop.owl` file. These IRI clashes show up in the diff not via the removal of the class itself, but via the removal and new addition of required annotations, such as rdfs:label and definition (IAO:0000115). Also, the linked class reference in the diff uses the wrong label for the term from the `mop.owl` (see `[3+2] cycloaddition` example below). These terms need to be discussed with RXNO/MOP maintainer and domain expert Colin Batchelor. This reason explains the differences on:
    * [3+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000715`
    * carbonyl oxidation to alkyne `http://purl.obolibrary.org/obo/MOP_0000714`
    * carboxylation `http://purl.obolibrary.org/obo/MOP_0000713`
    * enolisation `http://purl.obolibrary.org/obo/MOP_0000716`
    * oxidation state `http://purl.obolibrary.org/obo/MOP_0000712`
9. Some MOP terms in the `rxno.owl` file have fewer or less specific annotations and/or logical  axioms, and/or a less specific parent than in the `mop.owl` file (see `[3,3]-sigmatropic rearrangement` example below). This reason explains the differences on:
    * [3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
    * accepting a hydron `http://purl.obolibrary.org/obo/MOP_0000737`
    * accepting an electron pair in Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000733`
    * acid catalysis `http://purl.obolibrary.org/obo/MOP_0000740`
    * base catalysis `http://purl.obolibrary.org/obo/MOP_0000731`
    * breaking of covalent bond with group `http://purl.obolibrary.org/obo/MOP_0000567`
    * cycloelimination `http://purl.obolibrary.org/obo/MOP_0000564`
    * dealkylation `http://purl.obolibrary.org/obo/MOP_0001369`
    * decarboxylative condensation `http://purl.obolibrary.org/obo/MOP_0000722`
    * deethanolative condensation `http://purl.obolibrary.org/obo/MOP_0000723`
    * electron attachment `http://purl.obolibrary.org/obo/MOP_0000570`
    * electron pair donation in Lewis adduct formation `http://purl.obolibrary.org/obo/MOP_0000738`
    * elimination reaction `http://purl.obolibrary.org/obo/MOP_0000656`
    * formation of covalent bond with group `http://purl.obolibrary.org/obo/MOP_0000566`
    * hydron donation `http://purl.obolibrary.org/obo/MOP_0000732`
    * methylation `http://purl.obolibrary.org/obo/MOP_0000364`
    * oxidation `http://purl.obolibrary.org/obo/MOP_0000568`
    * reduction `http://purl.obolibrary.org/obo/MOP_0000569`
    * solvolysis `http://purl.obolibrary.org/obo/MOP_0000618`
    * sulfonylation `http://purl.obolibrary.org/obo/MOP_0000539`
10. Some MOP terms in the `rxno.owl` file have more or different annotations and/or SubClassOf axioms than in the `mop.owl` file (see `condensation reaction` example below). This reason explains the differences on:
    * condensation reaction `http://purl.obolibrary.org/obo/MOP_0000627`
    * cyclization `http://purl.obolibrary.org/obo/MOP_0000561`
    * cycloaddition `http://purl.obolibrary.org/obo/MOP_0000562`
    * epoxidation `http://purl.obolibrary.org/obo/MOP_0000671`
    * ester reduction `http://purl.obolibrary.org/obo/MOP_0000577`
    * ester reduction to aldehyde `http://purl.obolibrary.org/obo/MOP_0000578`
    * ester reduction to primary alcohol `http://purl.obolibrary.org/obo/MOP_0000579`
    * formylation `http://purl.obolibrary.org/obo/MOP_0000003`
    * ketone reduction `http://purl.obolibrary.org/obo/MOP_0000580`
    * molecular dehydration reaction `http://purl.obolibrary.org/obo/MOP_0000628`
    * nitration `http://purl.obolibrary.org/obo/MOP_0000556`
    * primary alcohol oxidation to aldehyde `http://purl.obolibrary.org/obo/MOP_0000573`
    * primary alkene oxidation to carboxylic acid and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000584`
    * quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588`
    * secondary alcohol oxidation to ketone `http://purl.obolibrary.org/obo/MOP_0000574`
    * secondary terminal alkene oxidation to ketone and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000587`
    * secondary, non-terminal alkene oxidation to aldehydes `http://purl.obolibrary.org/obo/MOP_0000586`
    * tertiary alkene oxidation to carboxylic acid and ketone `http://purl.obolibrary.org/obo/MOP_0000585`
11. Some MOP terms in the `rxno.owl` file have a different parent than in the `mop.owl` file (see `acylation` example below). This reason explains the differences on:
    * acylation `http://purl.obolibrary.org/obo/MOP_0000479`
    * allylation `http://purl.obolibrary.org/obo/MOP_0000422`
    * amine oxidation `http://purl.obolibrary.org/obo/MOP_0000575`
    * cycloaddition `http://purl.obolibrary.org/obo/MOP_0000562`
    * epoxidation `http://purl.obolibrary.org/obo/MOP_0000671`
    * formylation `http://purl.obolibrary.org/obo/MOP_0000003`
    * halogenation `http://purl.obolibrary.org/obo/MOP_0000550`
    * nitration `http://purl.obolibrary.org/obo/MOP_0000556`
    * organylation `http://purl.obolibrary.org/obo/MOP_0000458`
    * primary alkene oxidation to carboxylic acid and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000584`
    * quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588`
    * silylation `http://purl.obolibrary.org/obo/MOP_0000339`
    * sulfonylation `http://purl.obolibrary.org/obo/MOP_0000539`
12. Some MOP terms in the `mop.owl` file got added, due to being the parent of one of those terms that was defined in both files. This addition came from Step 4. of building the MOP module that was used in the comparison. This reason explains the differences on:
    * alkene oxidative cleavage `http://purl.obolibrary.org/obo/MOP_0000708`
    * alkenylation `http://purl.obolibrary.org/obo/MOP_0000420`
    * breaking of covalent bond `http://purl.obolibrary.org/obo/MOP_0000780`
    * catalysis `http://purl.obolibrary.org/obo/MOP_0000781`
    * dehydrocarbylation `http://purl.obolibrary.org/obo/MOP_0001410`
    * formation of covalent bond `http://purl.obolibrary.org/obo/MOP_0000779`
    * formation of covalent bond with carbon centre `http://purl.obolibrary.org/obo/MOP_0000800`
    * formation of covalent bond with halogen centre `http://purl.obolibrary.org/obo/MOP_0000820`
    * formation of covalent bond with silicon centre `http://purl.obolibrary.org/obo/MOP_0000810`
    * formation of covalent bond with sulfur centre `http://purl.obolibrary.org/obo/MOP_0000798`
    * is_catalysis_of `http://purl.obolibrary.org/obo/MOP_0000769`
    * oxidative cleavage `http://purl.obolibrary.org/obo/MOP_0000707`
    * sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000728`
    * univalent carboacylation `http://purl.obolibrary.org/obo/MOP_0000029`
13. The term `alkene oxidative cleavage` was defined in `rxno.owl`with http://purl.obolibrary.org/obo/RXNO_0000344 and in `mop.owl` with http://purl.obolibrary.org/obo/MOP_0000708. The RXNO duplicate was used as parent for MOP terms defined in the `rxno.owl`, whereas the MOP duplicate or another MOP term was used as parent for these terms in `mop.owl` (see `alkene ozonolysis`example below). This reason explains the differences on:
    * alkene ozonolysis `http://purl.obolibrary.org/obo/MOP_0000583`
    * quaternary alkene oxidation to ketones `http://purl.obolibrary.org/obo/MOP_0000588`
    * secondary, non-terminal alkene oxidation to aldehydes `http://purl.obolibrary.org/obo/MOP_0000586`
    * secondary terminal alkene oxidation to ketone and carbon dioxide `http://purl.obolibrary.org/obo/MOP_0000587`
    * tertiary alkene oxidation to carboxylic acid and ketone `http://purl.obolibrary.org/obo/MOP_0000585`
14. For some terms ROBOT calculated a diff, although there are no changes at all (see `allylic rearrangement`example below). This reason explains the differences on:
    * allylic rearrangement `http://purl.obolibrary.org/obo/MOP_0000792`

### Diff examples:
#### Brønsted base catalysis `http://purl.obolibrary.org/obo/MOP_0000736`
##### Removed
- [Brønsted base catalysis](http://purl.obolibrary.org/obo/MOP_0000736) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [Brønsted base catalysis](http://purl.obolibrary.org/obo/MOP_0000736) SubClassOf [has_part](http://purl.obolibrary.org/obo/rxno.obo#has_part) some [accepting a hydron](http://purl.obolibrary.org/obo/MOP_0000737)
##### Added
- [Brønsted base catalysis](http://purl.obolibrary.org/obo/MOP_0000736) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "Catalysis where the catalyst is a Br&oslash;nsted base." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
- [Brønsted base catalysis](http://purl.obolibrary.org/obo/MOP_0000736) SubClassOf [BFO_0000117](http://purl.obolibrary.org/obo/BFO_0000117) some [accepting a hydron](http://purl.obolibrary.org/obo/MOP_0000737)

#### N-arylation `http://purl.obolibrary.org/obo/MOP_0002411`
##### Removed
- [N-arylation](http://purl.obolibrary.org/obo/MOP_0002411) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [N-arylation](http://purl.obolibrary.org/obo/MOP_0002411) SubClassOf [BFO_0000057](http://purl.obolibrary.org/obo/BFO_0000057) some [CHEBI_33338](http://purl.obolibrary.org/obo/CHEBI_33338)
##### Added
- [N-arylation](http://purl.obolibrary.org/obo/MOP_0002411) [comment](http://www.w3.org/2000/01/rdf-schema#comment) "has_participant: CHEBI:33338" 

#### [3+2] cycloaddition `http://purl.obolibrary.org/obo/MOP_0000715`
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

#### [3,3]-sigmatropic rearrangement `http://purl.obolibrary.org/obo/MOP_0000721`
##### Removed
- [[3,3]-sigmatropic rearrangement](http://purl.obolibrary.org/obo/MOP_0000721) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [[3,3]-sigmatropic rearrangement](http://purl.obolibrary.org/obo/MOP_0000721) SubClassOf [molecular process](http://purl.obolibrary.org/obo/MOP_0000543)
##### Added
- [[3,3]-sigmatropic rearrangement](http://purl.obolibrary.org/obo/MOP_0000721) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A sigmatropic rearrangement where the intermediate consists of two three-electron fragments." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
- [[3,3]-sigmatropic rearrangement](http://purl.obolibrary.org/obo/MOP_0000721) SubClassOf [sigmatropic rearrangement](http://purl.obolibrary.org/obo/MOP_0000728) 

#### acylation `http://purl.obolibrary.org/obo/MOP_0000479`
##### Removed
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) [comment](http://www.w3.org/2000/01/rdf-schema#comment) "has_participant: CHEBI:22221"
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) SubClassOf [organylation](http://purl.obolibrary.org/obo/MOP_0000458)
##### Added
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) [comment](http://www.w3.org/2000/01/rdf-schema#comment) "you probably want &quot;carboacylation&quot;, MOP:0000028, here"
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) SubClassOf [formation of covalent bond with carbon centre](http://purl.obolibrary.org/obo/MOP_0000800)
- [acylation](http://purl.obolibrary.org/obo/MOP_0000479) SubClassOf [BFO_0000057](http://purl.obolibrary.org/obo/BFO_0000057) some [CHEBI_22221](http://purl.obolibrary.org/obo/CHEBI_22221) 

#### alkene oxidation to 1,2-diol `http://purl.obolibrary.org/obo/MOP_0000582`
##### Removed
- [alkene oxidation to 1,2-diol](http://purl.obolibrary.org/obo/MOP_0000582) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO" 

#### alkene ozonolysis `http://purl.obolibrary.org/obo/MOP_0000583`
##### Removed
- [alkene ozonolysis](http://purl.obolibrary.org/obo/MOP_0000583) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [alkene ozonolysis](http://purl.obolibrary.org/obo/MOP_0000583) SubClassOf [RXNO_0000344](http://purl.obolibrary.org/obo/RXNO_0000344)
##### Added
- [alkene ozonolysis](http://purl.obolibrary.org/obo/MOP_0000583) SubClassOf [alkene oxidative cleavage](http://purl.obolibrary.org/obo/MOP_0000708)

### allylic rearrangement `http://purl.obolibrary.org/obo/MOP_0000792`
#### Removed
- [allylic rearrangement](http://purl.obolibrary.org/obo/MOP_0000792) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [allylic rearrangement](http://purl.obolibrary.org/obo/MOP_0000792) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A substitution reaction in an allylic system with concomitant migration of the allyl double bond." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0002-4077-4719"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:db"
#### Added
- [allylic rearrangement](http://purl.obolibrary.org/obo/MOP_0000792) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A substitution reaction in an allylic system with concomitant migration of the allyl double bond." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://orcid.org/0000-0001-5985-7429"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:db"
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "RSC:cc"

### condensation reaction `http://purl.obolibrary.org/obo/MOP_0000627`
#### Removed
- [condensation reaction](http://purl.obolibrary.org/obo/MOP_0000627) [hasAlternativeId](http://www.geneontology.org/formats/oboInOwl#hasAlternativeId) "RXNO:0000315"
- [condensation reaction](http://purl.obolibrary.org/obo/MOP_0000627) [hasOBONamespace](http://www.geneontology.org/formats/oboInOwl#hasOBONamespace) "RXNO"
- [condensation reaction](http://purl.obolibrary.org/obo/MOP_0000627) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A reaction in which two or more reactants or remote reactive sites within the same molecular entity yield a single main product with accompanying formation of a small molecule." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "AU:C01238"
#### Added
- [condensation reaction](http://purl.obolibrary.org/obo/MOP_0000627) [IAO_0000115](http://purl.obolibrary.org/obo/IAO_0000115) "A reaction in which two or more reactants or remote reactive sites within the same molecular entity yield a single main product with accompanying formation of a small molecule." 
  - [hasDbXref](http://www.geneontology.org/formats/oboInOwl#hasDbXref) "https://doi.org/10.1351/goldbook.C01238" 