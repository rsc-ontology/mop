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

## Using ROBOT to identify the relevant MOP classes
@TODO: Add a paragraph about the initial workflow using ROBOT to convert RXNO and MOP to tsv first and then load into MS EXCEL to reconcile, with links to now outdated PR and files and such

### Make MOP Base files 
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
