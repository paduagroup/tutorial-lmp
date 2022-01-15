# tutorial-lmp

This tutorial gives step-by-step instructions to setup a system and run a molecular dynamics simulation with the CL&P (fixed-charge) and the CL&Pol (polarizable) force fields for ionic liquids, using the LAMMPS code.

It involves the following steps:

1. Make sure the necessary packages are installed: Python, VMD, Packmol, and LAMMPS compiled with the DRUDE package. 
2. Download the CL&P force field (non-polarizable version), the `fftool` script (to create input files and simulation box) and the `polarizer` tools (to generate the CL&Pol polarizable model).
3. Create input files and an initial simulation box with the non-polarizable force field using `fftool` and `packmol`.
4. Run a test trajectory.
5. Add explicit polarization terms using the `polarizer` tool.
6. Run an equilibration trajectory.
7. Run a production trajectory.
8. Calculate structural and dynamic quantities from the trajectory.

Molecular simulation using force fields requires the specification of **a lot of information** concerning the system (coordinates, topology), the force field model and the simulation conditions. 

**GO SLOWLY. INSPECT INPUT AND OUTPUT FILES. TRY TO UNDERSTAND EACH STEP.**


## 1  Make sure software is installed

If you are following this tutorial on a machine of the CBP center at ENS de Lyon, the packages needed are installed. If you are logging to the CBP machines from outside, then it is pertinent that you install VMD in your computer because visualization may be slow over an internet connection.

Python 3 is necessary to run the tools we develop.

VMD is a molecular visualization program:
    
        vmd
    
Packmol is a program that packs molecules in simulation boxes:
    
        packmol
    
Finally, LAMMPS is the molecular dynamics code we use in this tutorial. On the CBP LAMMPS is installed in `/projects/RFCT2022/software/lammps`:


        export LMP=/projects/RFCT2022/software/lammps
        $LMP/bin/lmp -h
    
and check that the binary was compiled with the DRUDE package (or USER-DRUDE in older versions).


## 2  Download CL&P, fftool and CL&Pol

These tools are available on [github.com/paduagroup](https://github.com/paduagroup)

        mkdir sim
        cd sim
        git clone https://github.com/paduagroup/clanp
        git clone https://github.com/paduagroup/fftool
        git clone https://github.com/paduagroup/clanpol
        
You can check that `fftool` runs and **learn about the command-line options**:

        ~/sim/fftool/fftool -h


## 3  Create a simulation box

Let's try one of the most studied ionic liquids, composed of the butylmethylimidazolium cation and the bistriflamide anion, [C4C1im][Ntf2]:

        mkdir c4c1im_ntf2
        cd c4c1im_ntf2

copy the CL&P  parameter database and molecule specification files for the ions:

        cp ~/sim/clandp/il.ff .
        cp ~/sim/clandp/c4c1im.zmat .
        cp ~/sim/clandp/ntf2.zmat .
        
Study these files. The molecule specification files contain atomic coordinates in `xyz` or `z-matrix` format and point to a file `.ff` which is the database of force-field parameters.


### 3.1 Create a test system with one molecule or ion

Use `fftool` to create a system with **one molecule** in a cubic box of 30 Å side:

        ~/sim/fftool/fftool 1 c4c1im.zmat -b 30

**Verify the electrostatic charge** of the molecule or ion (if it is not an integer then the attribution of atom types from the force field is not correct).

Inspect the `pack.inp` file, which contains instructions for `packmol` to place the molecules in the simulation box.

Use `packmol` to generate the atomic coordinates placing the ions in the box:

        packmol < pack.inp

Display the molecules in the box:

        vmd simbox.xyz

Run `fftool` again repeating the previous command line with `-l` to generate the LAMMPS input files:

        ~/sim/fftool/fftool 1 c4c1im.zmat 1 -b 30 -l
        

### 3.2 Look at the `data.lmp` file

The `data.lmp` file describes the system to be simulated and is usually a large file, not meant to be edited by hand. It includes atomic coordinates, electrostatic charges and molecular topology (every atom, bond, angle, torsion in the system). LAMMPS is a very atomistic code, working at the level of the atoms, with molecules thinly defined by just an index.

**Verify the number of bonds**: if this is not the number expected for the molecule, then the topology is wrong (maybe the initial geometry is not correct, with some bonds significantly different from the equilibrium distances in the force field file `il.ff`).

> **Question:** What is the general rule to determine the number of bonds in a molecule given the number of atoms and cycles? (It's easy.) And for the number of angles (between 3 atoms connected 1-2-3) and dihedrals (between 4 atoms connected 1-2-3-4)?

Look at the `Atoms` section which lists: atom index, molecule index, atom type, charge, x, y, z.

        Atoms

              1       1    1   0.150000  3.973990e+00  8.929037e+00  3.934037e+00  # NA c4c1im+
              2       1    2  -0.110000  3.751917e+00  7.657070e+00  3.685020e+00  # CR c4c1im+
              3       1    1   0.150000  4.051689e+00  6.936962e+00  4.743701e+00  # NA c4c1im+
              ...


### 3.3  Study the `in.lmp` file

The `in.lmp` file contains LAMMPS commands to perform the simulation, so it is meant to be read and modified by a person.

In our examples it also contains certain informations about the force field, namely:

- the Lennard-Jones parameters for the different atom types,
- the cutoff distance for the non-bonded interactions,
- the long-range part of the electrostatic interactions (`kspace`).

LAMMPS has a **very large set of commands**, which you can get acquainted with at [docs.lammps.org/Commands.html](https://docs.lammps.org/Commands.html). LAMMPS is a generalist code that can model molecules and materials, and perform sophisticated calculations during the run itself, so it provides a wide array of commands to evaluate quantities or to modify the system.

Two important classes of LAMMPS commands are `fix` and `compute`:

- a `fix` is executed in the main time-step iteration loop of molecular dynamics, at specified intervals; a `fix` is referred to in other commands by prepending `f_`;
- a `compute` declares a calculation but does not execute it: it has to be called from a `fix` by prepending `c_`.
    
The `variable` command declares numerical, formula or other types of variables that can be used in other commands by prepending `v_`.

The `thermo` command prints information on thermodynamic quantities during the simulation (energies, temperature, pressure, density, etc.) and can also print values of computes, fixes and variables (it acts like a `fix`).

The `dump` command writes configurations to a trajectory file.

The integrator of the equations of motion, a central piece in molecular dynamics, is a `fix`. LAMMPS has a choice of integrators including `fix nve`, `fix nvt` to couple the system with a thermostat, or `fix npt` to maintain pressure by changing the box dimensions.

Every command used in a simulation should be fully understood. This is a long learning process...


### 3.4 Repeat for each component

Repeat 3.1 for each component, verifying the charge and the number of bonds.


### 3.5 Create a box for a larger system

Let us build a box with 200 ion pairs with a confortable box size so that molecules do not overlap and are not too far either. Since we don't know beforehand the size of the box, let's give a density of 2.5 mol/L:

        ~/sim/fftool/fftool 200 c4c1im.zmat 200 ntf2.zmat -r2.5
        packmol < pack.inp
        
If `packmol` takes more than about 30 seconds, the density is probably too high. Try a smaller value.

Visualize with `vmd`.

Run `fftool` again with `-l` to generate the input files: 

        ~/sim/fftool/fftool 200 c4c1im.zmat 200 ntf2.zmat -r2.5 -l

**GOOD! YOU JUST SET UP A SYSTEM!**


## 4  Run a test trajectory

Run a few steps to make sure the system does not blow up.

Edit the `in.lmp` to run a few steps in NPT:

        ...
        thermo 200
        ...
        dump TRAJ all custom 200 dump.lammpstrj ...
        ...
        run 20000

Run on 16 cores:

        mpirun -np 16 $LMP/bin/lmp -in in.lmp > out.lmp &

Follow the progress with (`Ctrl-C` to exit)

        tail -f out.lmp

Once LAMMPS finishes, note the performance in ns/day. See in which tasks it spends most of the computation time (pair interactions, k-space electrostatics, ...).

Visualize the trajectory:

        vmd dump.lammpstrj
        vmd> pbc box -center origin
        vmd> pbc wrap -center origin -compound fragment -all

Plot density (column 12), temperature (column 9), energies, etc:

        gnuplot
        gnuplot> plot 'log.lammps' u 1:12 w lp

**GREAT! YOU JUST RAN A LAMMPS SIMULATION!**

At this point one can perform sufficiently long equilibration (5 ns) and production runs (20 ns) with the CL&P force field, which may be quite useful work to study ionic liquid systems. However, fixed-charge force fields have some issues for ionic fluids: they tend to result in too slow dynamics, so the liquids are too viscous when compared to experiment. Structural quantities, such as radial distribution functions (related to the structure factor measured by X-rays or neutrons), are usually well predicted.


## 5 Add explicit polarization

Adding explicit polarization terms involves, in our case, adding Drude particles to atoms (except H) so that induced dipoles appear as a response to the local electrostatic field. These terms are added to a non-polarizable system prepared as in section 3.

