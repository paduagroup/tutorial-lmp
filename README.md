# tutorial-lmp

Running a molecular simulation with the CL&Pol polarizable force field for ionic liquids in LAMMPS.

## Steps

1. Make sure the necessary packages are installed: Python, VMD, Packmol and LAMMPS compiled with the DRUDE package. 
2. Download the CL&P force field (non-polarizable version), the `fftool` script (to create input files and simulation box) and the `polarizer` tools (to generate the CL&Pol force field).
3. Create input files and an initial simulation box with the non-polarizable force field using `fftool` and `packmol`.
4. Add explicit polarization terms using the `polarizer` tool.
5. Run an equilibration trajectory.
6. Run a production trajectory.
7. Do post-treatment calculations of structural and dynamic quantities from the trajectory.

**GO SLOWLY. INSPECT INPUT AND OUTPUT FILES. TRY TO UNDERSTAND EACH STEP.**


## 1. Make sure software is installed

If you are following this tutorial on a machine of the CBP center at ENS de Lyon, the packages needed are installed. If you are logging to the CBP machines from outside, then it is pertinent that you install VMD in your computer because visualization may be slow over an internet connection.

Python 3 is necessary to run the tools we develop.

VMD is a molecular visualization program:
    
    vmd
    
Packmol is a program that packs molecules in simulation boxes.
    
    packmol
    
Finally, LAMMPS is the molecular dynamics code we use in this tutorial. On the CBP LAMMPS is installed in `/projects/RFCT2022/software/lammps`.

    /projects/RFCT2022/software/lammps/bin/lmp -h
    
and check that the binary was compiled with the DRUDE package (or USER-DRUDE in older versions).


## 2. Download CL&P, fftool and CL&Pol

These tools are available on [github.com/paduagroup](https://github.com/paduagroup)

        mkdir sim
        cd sim
        git clone https://github.com/paduagroup/clanp
        git clone https://github.com/paduagroup/fftool
        git clone https://github.com/paduagroup/clanpol
        
You can check that `fftool` runs and **learn about the command-line options**:

        ~/sim/fftool/fftool -h


## 3. Create an initial simulation box

Let's try an ionic liquids composed of the butylmethylimidazolium cation and the bistriflamide anion, `[C4C1im][Ntf2]`

        mkdir c4c1im_ntf2
        cd c4c1im_ntf2

Copy the CL&P data base and molecule specification files for the ions:

        cp ~/sim/clandp/il.ff .
        cp ~/sim/clandp/c4c1im.zmat .
        cp ~/sim/clandp/ntf2.zmat .
        
Study these files. The molecule specifications files contain atomic coordinates in `xyz` or `z-matrix` format and point to a file `.ff` which is a database of force field parameters.


### 3.1 Create a test system with one molecule or ion

Use `fftool` to create a system with **one ion** in a cubic box of 30 Å:

        ~/sim/fftool/fftool 1 c4c1im.zmat -b 30

**Verify the electrostatic charge** of the molecule or ion. If it is not an integer then the attribution of atom types from the force field is not correct.

Inspect the `pack.inp` file, which is an input for `packmol`.

Use `packmol` to generate the atomic coordinates placing the ions in the box:

        packmol < pack.inp

Have a look at the box:

        vmd simbox.xyz

Run `fftool` again with '-l' to generate the LAMMPS in put files:

        ~/sim/fftool/fftool 1 c4c1im.zmat 1 -b 30
        
Study the `in.lmp` and `data.lmp` files. These are LAMMPS input files.

### 3.2 Study the `data.lmp` file

The `data.lmp` file describes the system to be simulated and is usually a large file not to be edited by hand. It includes atomic coordinates, electrostatic charges and molecular topology (every bond, angle, torsion). LAMMPS is a very atomistic code, working at the level of the atoms with molecules thinly defined by just an index.

**Verify the number of bonds**: if this is not the number expected for the molecule, then the topology is wrong (maybe the initial geometry is not correct, with some bonds significantly different from the equilibrium distances in the force field file `il.ff`).

Look at the `Atoms` section which lists: atom index, molecule index, atom type, charge, x, y, z.

        Atoms

              1       1    1   0.150000  3.973990e+00  8.929037e+00  3.934037e+00  # NA c4c1im+
              2       1    2  -0.110000  3.751917e+00  7.657070e+00  3.685020e+00  # CR c4c1im+
              3       1    1   0.150000  4.051689e+00  6.936962e+00  4.743701e+00  # NA c4c1im+
              ...

### 3.3 Study the `in.lmp` file

The `in.lmp` file contains LAMMPS commands to perform the simulation, so it is meant to be read and modified by a human. In our examples it also contains certain informations about the force field, namely the Lennard-Jones parameters. LAMMPS has a **huge number** of commands, witrh which you can get acquainted at [docs.lammps.org/Commands.html](https://docs.lammps.org/Commands.html). LAMMPS is a very general code that can perform a variety of calculations during the run itself, so it contains many commands to evaluate quantities or to modify the system.

Two important classes of LAMMPS commands are `fix` and `compute` commands:

- a `fix` is executed in the main time-step iteration loop of molecular dynamics, at specified intervals; a `fix` is refered to in other commands by prepending `f_`;
- a `compute` declares a calculation but does not execute it: it has to be called from a `fix` by prepending `c_`.
    
The `variable` command declares numerical or formula variables that can be used in computes and fixes, by prepending `v_`.

The `thermo` command can be seen as a special instance of a `fix` that prints information on thermodynamic quantities during the simulation (energies, temperature, pressure, density, etc.) and can also print values of computes, fixes and variables.

The `dump` command outputs configurations to a trajectory file.

The integrators of the equations of motion, a central piece in molecular dynamics, is a `fix`. LAMMPS has several integrators including `fix nve`, `fix nvt` to couple the system with a thermostat, or `fix npt` to maintain pressure by changing the box dimensions.


### 3.4 Repeat for each component

Repeat 3.1 for each component, verifying the charge and the number of bonds.


### 3.5 Create a box for a larger system

SInce we don't know beforehand the size of the box, let's give a density of 2.5 mol/L:

        ~/sim/fftool/fftool 200 c4c1im.zmat 200 ntf2.zmat -r2.5
        packmol < pack.inp
        
If `packmol` takes more than about 30 seconds, the density is probably too high. Try with a smaller value.

Ru `fftool` again with `-l` to generate the input files: 

        ~/sim/fftool/fftool 200 c4c1im.zmat 200 ntf2.zmat -r2.5 -l

**GOOD! YOU ACCOMPLISHED THE INITIAL STEPS**


## Run an equilibration trajectory





