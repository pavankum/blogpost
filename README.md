# blogpost draft
## Open Source, Enterprise-level Data Pipeline 

At Openforcefield we generate a lot of Quantum Mechanics (QM) data as well as Molecular Mechanics (MM), predominantly for small molecules. A few data oriented tasks include
- training the valence parameters in the force field
- benchmarking the force field
- building bespoke parameters based on fragment torsions
- building machine learning models to predict charges
- building machine learned potentials
and many more analyses to answer scientific questions related to virtual sites, polarizability, etc.

In the context of openforcefield work, datasets can be broadly classified into three calculation types:
- Single point energies, with or without hessians (termed as BasicDataset)
- Optimized geometries of conformers (OptimizationDataset)
- 1D and 2D torsion potential scans (TorsionDriveDataset)

The QC* suite of packages (QCElemental, QCEngine, QCFractal, QCPortal, QCSchema), from our great friends at Molecular Software Sciences Institute (MolSSI), courtesy of Lori Burns, Benjamin Pritchard and team, is the foundation on which our data infrastructure stands. In creating our data management pipeline early work has been done by Daniel Smith, and current improvements are spearheaded by David Dotson and team. QCSchema is the standard we use to specify our calculation’s common quantum chemistry method. QCFractal is the job submission and queue management package, which uses QCEngine as the underlying interface to various QM and MM packages. Psi4, a widely used, community-driven, highly efficient open source electronic structure package, is the workhorse for our QM data generation. QCEngine supports a lot of other electronic structure packages (GAMESS, NWChem, etc.), analytical corrections (DFTD, gCP), semiempirical methods (XTB, Mopac), MM and AI potentials (GAFF, Openforcefields, ANI) as well. The long list of supported programs can be found in the QCEngine manual (section 5). The data generated with the help of these software tools is hosted on a public repository, known as Quantum Chemistry Archive (QCA), again a big thanks to MolSSI team. 

Lot of software development effort has gone into streamlining and automating most of the tasks, which can be broadly classified as 
1. Dataset preparation
2. Setup server and compute on an HPC cluster
3. Data access post computation

QCA-dataset-submission repository on the openff github, is a culmination of years of effort into standardizing and simplifying this whole process.

# Dataset preparation
First step involves creating a set of molecules you want to generate data on, you can use any of the available cheminformatics packages, and the input can be either 3D geometries in SDF format, or simple text based 2D representations such as smiles would work. Our datasets adhere to an evolving protocol of standards created by Trevor Gokey (https://github.com/openforcefield/qca-dataset-submission/blob/master/STANDARDS.md), keeping in mind the needs of training data for fitting. We have started with a compute-every-property approach when resources were abundant, but noticed the rising burden on storage infrastructure which made us move to a compute-and-store-necessary-properties policy. As an example, if we store the bond indices on a million molecules dataset where we intend to use only energies and forces this would fill up storage in 100s of gigabytes with data that will never be used. Such choices on property calculations have to be evaluated in advance so as to avoid choking up the compute and server infrastructure. After those initial decisions, Openff-QCSubmit, led by Josh Horton and team, can ingest the molecules information, quantum chemistry specification, additional metadata, and create dataset blobs that go as input to QCFractal. In our pipeline these submissions are merged after a pull request (PR) has been made and reviewed, but it is not necessary for other users in practice. We use Github actions a lot and one of them is to validate a dataset and catch any mistakes that had slid during the preparation stage. This automated step alleviates human error both in preparation and PR review. 

# Computation on an HPC cluster
The beauty of QCFractal is that it immensely simplifies job parallelization as well as mapping to already existing calculations in case a duplicate submission is made, which remediates the necessity to redo the same calculation. The mapping goes from Dataset collections → Molecules → Database records. All you need to do is either set up a long standing server instance on the main node or create an instance for a single job and back up the database before shutting the server down. After the server starts we can submit job managers using the cluster management backend, either SLURM, or PBS, or Docker pods. We sincerely thank the patience of our HPC admins on the following clusters for allowing us to use as many idle resources as possible,
- HPC3 and Greenplanet clusters at UCI
- Lilac cluster at MSKCC 
- TSCC at UCSD
- Pacific research platform (PRP)
- Sherlock at Stanford
- GWDG at MPI Gottingen
and also we thank our superusers who set up and monitor the job managers from time to time.

Job managers can be spun up with different node configurations (cores/memory) depending on the program specification and molecule sizes. For example, a CCSD calculation needs more resources than a hybrid DFT calculation. 

One caveat of using academic clusters is that we have to run a lot of jobs on pre-emptible queues to make use of idle nodes, thus when a job is suddenly stopped it returns an errored state. We have built an errorcycling script set up on our Github actions that periodically goes through each dataset under compute and restarts those failed calculations. This can also be run manually if the github actions configuration runs out of memory for large datasets. This bot also prints relevant error codes for the failed runs and it would help in debugging persistent issues. After the dataset finishes all calculations then it is archived automatically. Some datasets may never achieve 100% complete but attains an acceptable error state (say 99% complete), then a decision is made to move it to end-of-life manually after reviewing the remaining errors. Some of these error handling tasks may be incorporated into the next release of QCFractal to do it server side reducing the burden on downstream users.

As of now QCArchive stands at 97+ Million molecules, 103+ Million different calculations, and 200+ dataset collections. This is a great achievement considering the devops battles one has to endure to maintain the database with far less downtime and we sincerely appreciate Benjamin Pritchard for keeping the light on single-handedly. 


A dataset lifecycle is shown in figure 1.
![image](https://user-images.githubusercontent.com/16142894/168649582-18fc4e7e-58d4-4573-8c57-a1c4c6bfdc18.png)


Figure 1: David Dotson’s illustration of a dataset’s lifecycle

# Data retrieval 
Dataset retrieval is again facilitated by Openff-qcsubmit and there are a lot of post-processing functions that would help filter out the necessary molecule sets. Single point calculations are still a little bit difficult to download faster and evolving infrastructure changes would smoothen it further. 

Working example of creating a dataset of biaryls with different substituents is shown in the notebooks for anyone interested with a sample case of creating a biaryls torsiondrive dataset.
