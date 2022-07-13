# Run the calculations

[Go back to TOC](../../../../../../README.md)

Detailed information on the files here are given in [this file](../../../README.md). 

## What is this
  
 We use CP2K v9.1 as is currect on the CCR as of June 2022
 
 Here we conduct the time-overlap calculations for adamantane molecule using BEEF functional

## The procedure

### 1. Edit es_diag_temp.inp 

 You can start by copying the md input from step 1 and then:

  1.1. Make sure you include the following printing statements, you can customize part of it
  but we need to be able to produce molden files for the workflow to work correctly.

    &PRINT
    &MULLIKEN OFF
      &END
      &HIRSHFELD OFF
      &END
      &MO ON
        FILENAME adamantane
        EIGENVECTORS F
        EIGENVALUES F
        NDIGITS 8
      &END
      &PDOS
        APPEND T
        COMPONENTS T
        FILENAME adamantane
      &END
      &MO_MOLDEN ON
        FILENAME
        NDIGITS 8
        GTO_KIND SPHERICAL
      &END
      &MO_CUBES
        NHOMO 1
        NLUMO 1
        STRIDE 3 3 3
        WRITE_CUBE T
      &END
    &END PRINT

  1.2. In the `SCF` section, we need to enable calculations based on the matrix diagonalization
  so that we could get the orbitals. In the MD step we used the OT approach.

      &DIAGONALIZATION ON
      &END DIAGONALIZATION
      &MIXING
        ALPHA 0.3
        METHOD BROYDEN_MIXING
        NBROYDEN 8
      &END MIXING

  1.3. Very important: need to add unoccupied orbitals. In the MD step, we didn't need it.

      ADDED_MOS 30

  1.4. The runtype is now Energy, not the MD

      &GLOBAL
        PROJECT adamantane
        RUN_TYPE Energy
        PRINT_LEVEL Medium
      &END GLOBAL


### 2. Edit submit_template.slm 

  2.1. Choose the queue to use. Either Valhalla:

      #SBATCH --partition=valhalla  --qos=valhalla
      #SBATCH --clusters=faculty

  or general compute nodes

      #SBATCH --partition=general-compute  --qos=general-compute
      #SBATCH --clusters=ub-hpc

  or scavenger nodes

      #SBATCH --partition=scavenger  --qos=scavenger
      #SBATCH --clusters=faculty


  2.2. Choose sufficient amount of memory and time. Sometimes having not enough memory will kill the job.

      #SBATCH --time=24:00:00
      #SBATCH --mem=80000


  2.3. Choose a suitable parallelization. In this example, we use 16 cores. 
       CP2K likes the parallelization over n^2 CPUS.

      #SBATCH --nodes=1
      #SBATCH --ntasks-per-node=16
      #SBATCH --cpus-per-task=1

  2.4. Choose a proper OpenMP number of threads per core. Most nodes on UB CCR have 1 thread per CPU core.

      export OMP_NUM_THREADS=1

  
### 3. Edit run_template.py

  3.1. Use the correct number of CPUs - this should be consistent with the 
    number indicated in 2.3. So, in this example, we use 16

      params['nprocs'] = 16

  If you use Valhalla nodes, please note that they have only 12 number of cores. In this case, you can use 9 cores or 12 but please set the `NGIRDS` in the `&MGRID` section devisible by the number of processors

      &MGRID
        CUTOFF 300
        NGRIDS 12
      &END 


  3.2. Define the active space of orbitals for which the time-overlaps will be computed
     The index of the HOMO should be determined for each system. For instance, the output of the
     MD calculations in step1 will contain this information. In this example, the number is 28.
     We include 20 occupied and 20 unoccupied orbitals:

      # Lowest and highest orbital, Here HOMO is 28
      params['lowest_orbital'] = 28-20
      params['highest_orbital'] = 28+20

   The number of the unoccupied orbitals included should not be larger than the number of added MOs.
   See setup in 1.3.

   3.3. Make sure you indicate it is not xTB, if you are doing DFT or TD-DFT

      # extended tight-binding calculation type
      params['isxTB'] = False

   3.4. Make sure to provide the correct path to a correct cp2k executable (correct version). 
   As of now, the CP2K v9.1 is installed on the CCR at:

      params['cp2k_exe'] = '/projects/academic/cyberwksp21/Software/cp2k-intel/cp2k-9.1/exe/Linux-x86-64-intelx/cp2k.psmp'

   **Update**: For using CP2K v9.1 ion UB CCR compiled with GNU compilers, you can use one of these executables

      # With ELPA 
      params['cp2k_exe'] = '/projects/academic/cyberwksp21/Software/cp2k-gnu/cp2k-9.1/exe/local/cp2k.psmp'
      # Without ELPA
      params['cp2k_exe'] = '/projects/academic/cyberwksp21/Software/cp2k-gnu/cp2k-9.1-no-elpa/exe/local/cp2k.psmp'

   Before running the calculations, make sure you have loaded all the depndencies required for loading CP2K in the submit file. For versions that are made with Intel compielrs

      module load intel-mpi/2020.2
      module load intel/20.2

   and for GNU-based compilers

      source /projects/academic/cyberwksp21/Software/cp2k-intel/avx512-dependencies/cp2k-9.1-avx512/tools/toolchain/build/setup_gcc
      source /projects/academic/cyberwksp21/Software/cp2k-intel/avx512-dependencies/cp2k-9.1-avx512/tools/toolchain/install/setup
      module load mkl/2020.2
 
   3.5. Make sure you provide the correct name of the trajectory file - this is the file generated by step1:

      params['trajectory_xyz_filename'] = path + '/../adamantane-pos-1.xyz'
 

### 4. Edit distribute_jobs.py  

  This is the easiest file to edit - you just define what part of the trajectory should be re-computed
  and my how many chunks. In this example, we handle first 100 steps and distribute them over 10 jobs

      istep = 0
      fstep = 100
      njobs = 10


### 5. Submit multiple jobs as

  By running the command in the terminal

     python distribute_jobs.py

 This Python script will create multiple folders, with the content of each one
 being automatically set up to run the calculations for a chunk of the trajectory

 Note: make sure you have Libra activated before you run this Python script

     module load jupyter
     conda activate libra