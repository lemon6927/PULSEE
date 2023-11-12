# PULSEE (**P**rogram for the Sim**ul**ation of Nuclear **S**pin **E**nsemble **E**volution)

Developers: Jiwoo S., Ilija N., Lucas B., Stephen C., Alex A. (Brown University), Davide Candoli (Università di Bologna)

PULSEE is an open-source software for the simulation of typical nuclear quadrupole/magnetic resonance experiments on a solid-state sample, describing the dynamics of nuclear spins in condensed matter under the effect of external magnetic fields and reproducing the traditional results observed in laboratory.

<!-- TOC -->
* [PULSEE (**P**rogram for the Sim**ul**ation of Nuclear **S**pin **E**nsemble **E**volution)](#pulsee-program-for-the-simulation-of-nuclear-spin-ensemble-evolution)
  * [Physics Background](#physics-background)
  * [Example](#example)
    * [Unit standard of the software](#unit-standard-of-the-software)
  * [Installation](#installation)
    * [Creating a Conda Environment with Required Packages](#creating-a-conda-environment-with-required-packages)
    * [Editable Installation of PULSEE](#editable-installation-of-pulsee)
    * [Notes About Requirements](#notes-about-requirements)
  * [Software](#software)
    * [Modules of the software](#modules-of-the-software)
    * [Further examples](#further-examples)
      * [Pure Zeeman experiment](#pure-zeeman-experiment)
      * [Perturbed Zeeman experiment](#perturbed-zeeman-experiment)
      * [Pure NQR experiment](#pure-nqr-experiment)
  * [Acknowledgements](#acknowledgements)
<!-- TOC -->

## Physics Background

Each atomic nucleus in a single crystal is endowed with an intrinsic angular momentum, named spin, and a corresponding intrinsic magnetic moment. The interaction between this magnetic moment and any applied magnetic field provides a means for the manipulation of nuclear spins and the study of nuclear interactions in a material.

The basic assumption of the simulations implemented with this program is that the nuclei belonging to the sample under study are identical and independent, so that the set of all their spins can be treated as an ideal *statistical ensemble*. As such, the whole state of the many-spin system can be described with a *mixed density matrix* of dimensions 2s+1, where s is the quantum numer of a single spin.

Let us consider such a system under the effect of a static magnetic field: the degeneracy of spin states is broken, and they arrange into equally spaced energy levels, with an energy separation determined by the strength of the field and the gyromagnetic ratio of the nuclear spin.

In order to induce transitions between these states, a pulsed electromagnetic wave must have a frequency tuned with the resonance frequency of the system. This is the concept at the basis of the nuclear magnetic resonance (NMR) technique.

In nuclear quadrupole resonance (NQR), the degeneracy of spin states is broken by the interaction between the electric quadrupole moment of the nucleus and the electric field gradient (EFG) generated by surrounding electrons. The applied electromagnetic pulses are able to induce an appreciable change in the state of the system only if their frequency is in resonance with a frequency of transition.

In general, experimental systems are characterized by an intermediate configuration where both a magnetic field and an EFG are present. The corresponding interactions must be taken into account simultaneously in order to determine the energy spectrum.

After the application of a given pulse sequence, the system's magnetization typically develops a non-zero component along the direction of the coil employed for the generation of the pulse. This component changes with time in a manner dependent on the energy spectrum of the nuclei, eventually relaxing to zero due to the unavoidable dephasing of different spins. This relaxation is denoted by time T<sub>2</sub> (*coherence time*).

The time dependence of the magnetization is measured through the acquisition of the current it induces in the coil. After performing the Fourier analysis of this signal, one can extract information about the system, according to the position of the peaks of the spectrum, their shape, and their sign.

## Example

One can set up a simulation as follows. Import PULSEE's simulation module: 
```
from pulsee import simulation as sim
```

Define the system parameters as given by the function `nuclear_system_setup`'s
documentation. For example, to include a Zeeman interaction and a quadrupolar
interaction on an $s = 5/2$ system with $\gamma/ 2\pi = 1$, 
```
spin_par = {'quantum number' : 5/2,
            'gamma/2pi' : 1.}

zeem_par = {'field magnitude' : 10.,
            'theta_z' : 0,
            'phi_z' : 0}

quad_par = {'coupling constant' : 0.,
            'asymmetry parameter' : 0.,
            'alpha_q' : 0,
            'beta_q' : 0,
            'gamma_q' : 0}
```

Now specify the initial state via a density matrix; in this case we choose 
the $m = 5/2$ state given by the following density matrix:
```
initial_state = np.zeros((6, 6))
initial_state[0, 0] = 1
```

We can then call `nuclear_system_setup` to produce the corresponding spin 
operators (as `spin`), the Hamiltonian representing each of the specified 
interactions (treated as an unperturbed Hamiltonian in the sense of perturbation
theory), and the initial state's density matrix (in this case the same as
specified above):
```
spin, h_unperturbed, dm_initial = nuclear_system_setup(spin_par, quad_par, zeem_par, \
                                                        initial_state=initial_state)
```

We may now specify a pulse sequence to apply as a Pandas `DataFrame`: 
```
mode = pd.DataFrame([(10., 1., 0., np.pi/2, 0, 2 * np.pi)], 
                    columns=['frequency', 'amplitude', 'phase', 'theta_p', 'phi_p', 'pulse_time'])
```

Finally, we can evolve this state with 
```
from qutip import mesolve

dm_evolved = evolve(spin, h_unperturbed, dm_initial, mesolve,
                                mode=mode)
```
where we have chosen QuTiP's `mesolve` solver and a pulse time of $2\pi$ seconds.
We can alternatively use `mesolve` using a string:
```
dm_evolved = evolve(spin, h_unperturbed, dm_initial, 'mesolve',
                                mode=mode, picture='IP')
```
Other compatible solvers include the `magnus` magnus expansion solver that 
can be imported from `pulsee.simulation` (which may also be specified as a 
string `'magnus'`) or any solver function with signature `(Qobj, Qobj, ndarray,
**kwargs) -> qutip.solver.Result`.

To obtain the FID signal of the spins one can run 
```
t, FID = FID_signal(spin, h_unperturbed, dm_evolved, acquisition_time=50, 
         T2=10, reference_frequency=0, n_points=10)
```
where `acquisition_time` denotes the amount of time we are capturing the FID signal, `T2` is the (empirically determined) T2 decay time `reference_frequency` is the frequency of rotation of the capturing reference frame, and `n_points` is the number of samples per microsecond that are measured within the acquisition time. 

Passing a list of values produces a decay envelope equivalent to the product 
of exponential decay envelopes with the corresponding T2 values: 
```
t, FID = FID_signal(spin, h_unperturbed, dm_evolved, acquisition_time=50, 
         T2=[5, 10, 15], reference_frequency=0, n_points=10)
```

Alternatively, PULSEE allows one to specify a custom decay function 
```
t, FID = FID_signal(spin, h_unperturbed, dm_evolved, acquisition_time=50, 
         T2=lambda t: np.exp(-t/50), reference_frequency=0, n_points=10)
```
and follows the same protocol as above when given a list of functions 
```
t, FID = FID_signal(spin, h_unperturbed, dm_evolved, acquisition_time=50, 
T2=[lambda t: np.exp(-(t/50)**1.5), lambda t: np.exp(-(t/50)**2)], reference_frequency=0, n_points=10)
```

To obtain the spectrum one can simply run 
```
f, ft = fourier_transform_signal(FID, t)
```

### Unit standard of the software

In order to save processing power and lighten calculations, a suitable choice of units of measure has been taken.

**Remark:** In the whole software, energies and frequencies are considered quantities with the same physical dimensions. The identification between them is a consequence of setting the Planck constant h equal to 1.

The standard units employed in the software are listed below.

| physical quantity  | unit  |
|--------------------|-------|
| gyromagnetic ratio | MHz/T |
| magnetic field     | T     |
| energy/frequency   | MHz   |
| temperature        | K     |
| time               | us    |

Angles do not have a standard unit: they are measured in radians when they are passed directly to the software's functions, while they are measured in degrees when they are inserted in the GUI.

## Installation

In the current development phase, we recommend git cloning the repository into your local machine and following the 
later instructions of installing the local project in "editable" mode, so that any changes are reflected immediately &
conveniently.

For most users (using Windows or Mac), we recommend using the popular Anaconda distribution which the developers have mainly used and are most familiar with.
If you are not familiar with Anaconda, there are many resources online and you can start [here](https://docs.anaconda.com/free/anaconda/):

For users using Linux, more experienced with package management, or prefer to not use Anaconda, they can skip to the 'Installation of PULSEE' section.

### Creating a Conda Environment with Required Packages

We assume the user has successfully installed Anaconda and have a cloned PULSEE repository in their local machine.
This section is mainly written in language of Windows, but Mac users should
be able to follow along with their corresponding language.

Run 'Anaconda Prompt' as administrator. Navigate to your local PULSEE directory by running
```shell
cd {path_to_PULSEE}
```
where `path_to_PULSEE` is the path to your local PULSEE directory.

Instead of working in the base environment, we will create a new & clean Conda environment with just the required packages ([as the recommended standard practice](https://conda.io/projects/conda/en/latest/user-guide/getting-started.html#managing-envs)).\
Run this command to create a new Conda environment with name `pulsee_env` which installs all the packages specified in the `conda-requirements.yml` file.
(Click [here](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html) for details on conda environment management)
```shell
conda env create -f conda-requirements.yml
```
Verify that this environment was successfully installed by running `conda env list` and checking that there's an environment listed as `pulsee_env` along with the path to its location.

Activate this new environment by running
```shell
conda activate pulsee_env
```
Running `conda list` will let you see all the packages installed in this environment.
Verify that the version of Python is 3.11 or higher.

Now we will install PULSEE in our `pulsee_env` environment by adding a pointer to your local repository of PULSEE.
We do this by running the following command while inside the PULSEE directory (where `setup.py` is located):
```shell
pip install -e .
```
We now have a working environment with PULSEE and all its dependencies!

Any changes to your local PULSEE directory will be reflected immediately, so make sure to keep your local PULSEE directory up to date for any recent changes.

Finally, we can make sure that we will be able to use this new Conda environment in Jupyter by making sure
to install the kernel of our new environment by running:
```shell
python -m ipykernel install --user --name=pulsee_env
```
Now when we open Jupyter, we should be able to choose the pulsee_env kernel (instead of the default `python3`).

### Editable Installation of PULSEE
(For users not using Conda)

Once all the required packages have been installed with `requirements.txt`, git clone the PULSEE repository to your local machine.
Then do an editable installation by running the following command in the PULSEE directory (where `setup.py` is located):
```shell
pip install -e .
```

### Notes About Requirements

All dependencies are summarized in the `conda-requirements.yml` and `requirements.txt` files. 

PULSEE has been mainly developed in Python 3.11, and a Python version of 3.11 is required.

The operative systems where it has been tested and executed are
* Ubuntu
* Windows 10 (Through the Anaconda distribution)

The software makes wide use of many of the standard Python modules (namely `numpy`, `scipy`, `pandas`, `matplotlib`) for its general purposes.

The QuTip package is required -> https://qutip.org/.
We downgrade cython to version 0.29 in our requirements, since QuTip only supports up to cython 0.29.

Tests have been carried out using the `pytest` framework and the `hypothesis` module.

`pytest` -> https://docs.pytest.org/en/stable/

`hypothesis` -> https://hypothesis.readthedocs.io/en/latest/

The GUI has been implemented with the tools provided by the Python library `kivy`.

`kivy` -> https://kivy.org/#home

In order to run the GUI, it is required the additional installation of the module `kivy.`

## Software

### Modules of the software

PULSEE's modules are described in detail in the wiki page of the repository of GitHub which hosts the project:

https://github.com/DavideCandoli/NQR-NMRSimulationSoftware/wiki
[deprecated]

Below, the content and usage of these modules is reported briefly:

* `operators`

  Along with `many_body`, this module is considered a toolkit for the simulations of generic quantum systems. `operators` contains the definitions of classes and functions representing the basic mathematical objects employed in the treatment of a quantum system. `operators` focuses on the properties of a single system, while the functions in `many_body` build systems made up of several particles.

  The classes and subclasses defined in `operators` belong to the following inheritance tree:
  
  * `Operator`
    * `DensityMatrix(Operator)`
    * `Observable(Operator)`
    
  Class `Operator` defines the properties of a generic linear application acting in the Hilbert space of a finite-dimensional quantum system. Its main attribute is `matrix`, a square array of complex numbers which gives the matrix representation of the operator in a certain basis. The methods of this class implement the basic algebraic operations and other common actions involving operators, such as the change of basis.
  
  Class `DensityMatrix` characterizes the operators which represent the state (pure or mixed) of the quantum system. It is defined by three fundamental properties:
  1. Hermicity
  2. Unit trace
  3. Positivity

  Class `Observable` characterizes the hermitian operators which represent the physical properties of the system.
  
  Other functions defined in this module perform:
  * the calculation of the first few terms of the Magnus expansion of a time-dependent Hamiltonian, which enter the approximate formula for the evolution operator 
  * the calculation of the canonical density matrix representing the thermal equilibrium state of a system at the specified temperature


* `many_body`

  This module defines two functions which allow us to pass an operator from a single particle Hilbert space to a many particle space and vice-versa.
  
  * `tensor_product`
  
    Takes two operators of arbitrary dimensions and returns their tensor product.
    
  * `ptrace_subspace`
  
    Takes an operator acting on the Hilbert space of a many-particle system and extracts its partial trace over the specified subspace.


* `nuclear_spin`

  In this module, the objects defined in `operators` are employed to construct the class which represents the spin of an atomic nucleus or a system of nuclei.
  
  Class `NuclearSpin` is characterized by a quantum number, a gyromagnetic ratio, and a set of methods that return the spherical and cartesian components of the spin vector operator.
  
  The subclass `ManySpins` contains the `NuclearSpin` objects representing the individual spins in an ensemble of nuclei, as well as a method for the calculation of the operators of the total spin.


* `hamiltonians`

  This file is dedicated to the definitions of the functions which compute the terms of the Hamiltonian of a nuclear spin system in a NMR/NQR experiment.
  
  * `h_zeeman`
    
    Builds up the Hamiltonian of the interaction between the spin and an external static magnetic field, after its magnitude and direction have been given.

  * `h_quadrupole`
  
    Given the coupling constant of the interaction, the asymmetry of the EFG, and the direction of its principal axes, `h_quadrupole` constructs the Hamiltonian of the interaction between the nuclear electric quadrupole momentum and the EFG.
    
    In its scope, `h_quadrupole` calls the three functions `v0_EFG`,`v1_EFG`, `v2_EFG` in order to compute the spherical components of the EFG tensor, which enters the expression of the quadrupole Hamiltonian.

  * `h_single_mode_pulse`
    
    Returns the Hamiltonian of interaction of a nuclear spin with a linearly polarized electromagnetic wave, once the properties of the wave and the time of evaluation have been passed.
    
    This function is iteratively called by `h_multiple_mode_pulse`, which returns the Hamiltonian of interaction between a nuclear spin system and a superposition of pulses.
    
  * `h_changed_picture`
    
    In turn, `h_multiple_mode_pulse` is called inside `h_changed_picture`, which, in the given instant of time, evaluates the full Hamiltonian of the system (comprised of the Zeeman, quadrupole and time-dependent contributions) and returns the same Hamiltonian expressed in a different dynamical picture. This passage is required by the implementation of the evolution of the system, which is described later.


* `simulation`

  This module provides the function definitions that implement the various tasks of the simulation. Simple simulations, similar to the following examples, will largely rely on the functions in this module. More sophisticated simulations are feasible, but they require a deeper understanding of the program as development of a complex quantum system relies on manipulation of the other modules. 

  The order in which the definitions in `simulation` appear suggests the ideal progression of the various steps of the simulation.

  1. `nuclear_system_setup`
  
     Constructs the system under study: creates the objects associated with the nuclear spin system, the unperturbed Hamiltonian (Zeeman + quadrupole contributions), and the initial state.      
  
  2. `power_absorption_spectrum`
  
     Computes the power absorbed by the system from a pulse described by the theoretical formula derived from Fermi's golden rule.
     
  3. `evolve`
     
     Evolves the state of the system under the action of a given electromagnetic pulse or the stationary Hamiltonian. Evolution is carried out in one of the three methods: 
     - Direct diagonalization for time independent Hamiltonians.
     - Average Hamiltonian Theory: approximating the evolution operator with the first terms of the Magnus expansion. The user can specify in which dynamical picture to evolve the system.
     - QuTiP's evolution with mesolve.

  4. `FID_signal`
  
     Simulates the free induction decay signal generated by the transverse magnetization of the system after the application of a pulse.
    
  5. `fourier_transform_signal`
  
     Performs the Fourier analysis of a signal (in this context, the FID).
    
  6. `fourier_phase_shift`
  
     Computes the phase shift to be applied to the FID in order to correct the shape of the Fourier spectrum of the system and recover the typical absorptive/dispersive lorentzian shapes at a given peak.
    
  In addition to these main functions, this module contains the functions for the plot and visualization of the results.
  

* `quantum_computing`
  This module contains implementations of fundamental components of quantum 
  circuits, including several relevant quantum gates and `Qubit` objects that
  gates may act on. In principle, this allows the user to construct elementary quantum
  circuits and extract relevant information such as the density matrix of the compound
  qubit state. 
  
* `PULSEE_CMP_GUI` and `PULSEE_CHEM_GUI`

  Graphical user interfaces of the program specialized for condensed matter 
  physics and chemistry respectively. The execution of the following command from the terminal
  
  `$ python PULSEE_CMP_GUI.py` 
  
  `$ python PULSEE_CHEM_GUI.py`
  
  launches the application. This has been developed using the tools provided by the library Kivy.
  
  In the application, most of the features of the software are available (power absorption spectra and a few others are not included). An in-depth description of the GUI can be found in the related page of the wiki:
  
  https://github.com/DavideCandoli/PULSEE/wiki/GUI

### Further examples

Any simulation can be implemented using only the functions defined in the module
`simulation`. Therefore, the imports required by a generic simulation code are
the following: 

```
from pulsee.simulation import *
```

#### Pure Zeeman experiment

The simplest experiment one can simulate is the case of pure NMR, where a static magnetic field (conventionally directed along z) is applied to a nucleus where the quadrupolar interaction is negligible.

Take for instance a spin 1 nucleus: the set-up of the system is carried out passing to `nuclear_system_setup` the following parameters:
```
spin_par = {'quantum number' : 1.,
            'gamma/2pi' : 1.}
    
zeem_par = {'field magnitude' : 1.,
            'theta_z' : 0.,
            'phi_z' : 0.}
                
spin, h_unperturbed, dm_0 = nuclear_system_setup(spin_par=spin_par, quad_par=None, zeem_par=zeem_par, initial_state='canonical', temperature=1e-4)

plot_real_part_density_matrix(dm_0)
```
where the system has been initially set at thermal equilibrium at a temperature of 10<sup>-4</sup> K.
![Pure Zeeman - Initial_State](Figures_README/Pure_Zeeman_Initial_State.png)

Then, the power absorption spectrum can be simulated running the functions
```
f, p = power_absorption_spectrum(spin, h_unperturbed, normalized=True)

plot_power_absorption_spectrum(f, p)
```
![Pure Zeeman - Power_Absorption](Figures_README/Pure_Zeeman_Power_Absorption.png)

In order to apply a 90° pulse to the system, which rotates the magnetization from the z-axis to the x-y plane, we shall design a pulse in resonance with the system such that the product

gyromagnetic ratio x pulse field magnitude x pulse time

is equal to pi/2. Setting a pulse made up of the single linearly polarized mode
```
mode = pd.DataFrame([(2 * np.pi, 0.2, 0., np.pi/2, 0.)], 
                     columns=['frequency', 'amplitude', 'phase', 'theta_p', 'phi_p'])
```
the pulse time should be equal to `1/(4 * 0.1)` in order to produce a 90° rotation. Indeed, the effective amplitude of the wave is 0.05 T: the linearly polarized mode splits into two rotating waves, only one of which is in resonance with the system.

Then, the state of the system is evolved and plotted with the following calls:
```
dm_evolved = evolve(spin, h_unperturbed, dm_0, solver=magnus, \
                    mode=mode, pulse_time=1 / (4 * 0.1), \
                    picture = 'IP')
    
plot_real_part_density_matrix(dm_evolved)
```
![Pure Zeeman - Evolved State](Figures_README/Pure_Zeeman_Evolved_State.png)


The evolved density matrix can be employed to generate the FID signal of the system as follows:
```
t, fid = FID_signal(spin, h_unperturbed, dm_evolved, acquisition_time=100, T2=10)

plot_real_part_FID_signal(t, fid)
```
![Pure Zeeman - FID Signal](Figures_README/Pure_Zeeman_FID_Signal.png)

**Remark:** In order to acquire a true reproduction of the continuous FID signal, one needs to compare the largest frequency in its Fourier spectrum with the frequency of sampling of the signal. Indeed, Nyquist theorem states that the latter must be at least twice the former to ensure a correct sampling. See the documentation for FID_signal to learn how to set the number of sampling points.

The Fourier analysis of the FID signal produces the NMR spectrum:
```
f, ft = fourier_transform_signal(fid, t)
    
plot_fourier_transform(f, ft)
```
![Pure Zeeman - NMR Spectrum](Figures_README/Pure_Zeeman_NMR_Spectrum.png)

#### Perturbed Zeeman experiment

When the quadrupolar interaction is non-negligible, but still very small compared to the interaction with the magnetic field, one is in the so-called *perturbed Zeeman* regime.

An experiment with these conditions can be easily simulated following the same steps described in the pure Zeeman case with the only difference being a non-zero quadrupolar coupling constant:

```
quad_par = {'coupling constant' : 0.2,
            'asymmetry parameter' : 0.,
            'alpha_q' : 0.,
            'beta_q' : 0.,
            'gamma_q' : 0.}
            
spin, h_unperturbed, dm_0 = nuclear_system_setup(spin_par=spin_par, quad_par=quad_par, zeem_par=zeem_par, initial_state='canonical', temperature=1e-4)
```

The presence of this perturbation leads eventually to a spectrum with two resonance peaks.
![Perturbed Zeeman - NMR Spectrum](Figures_README/Perturbed_Zeeman_NMR_Spectrum.png)

As one can see, the real and imaginary parts of the spectrum at each peak don't fit the conventional absorptive/dispersive lorentzian shapes, which would be a nice feature to be visualized. By means of the function `fourier_phase_shift`, one can obtain the phase for the correction of the shape of the spectrum at a specified peak (the simultaneous correction at both peaks is impossible):
```
phi = fourier_phase_shift(f, ft, peak_frequency=0.82, int_domain_width=0.2)

plot_fourier_transform(f, np.exp(1j*phi)*ft)
```
![Perturbed Zeeman - Corrected NMR Spectrum](Figures_README/Perturbed_Zeeman_Corrected_NMR_Spectrum.png)

#### Pure NQR experiment

Another important type of experiments is that of *pure NQR*, where the only term of the unperturbed Hamiltonian is the quadrupolar interaction. The pure NQR of spin 3/2 nuclei can be simulated changing the parameters in the previous two examples as
```
spin_par = {'quantum number' : 3/2,
            'gamma/2pi' : 1.}
    
quad_par = {'coupling constant' : 2.,
            'asymmetry parameter' : 0.,
            'alpha_q' : 0.,
            'beta_q' : 0.,
            'gamma_q' : 0.}
            
spin, h_unperturbed, dm_0 = nuclear_system_setup(spin_par=spin_par, quad_par=quad_par, zeem_par=None, initial_state='canonical', temperature=1e-4)
```
where we have set the coupling constant of the quadrupole interaction to 2 MHz.

In such a configuration, the pulse set up in the previous example turns to be in resonance with the new system as well, so that it can be left unaltered.

The initial state is
![Pure_NQR - Initial_State](Figures_README/Pure_NQR_Initial_State.png)

while the evolved one is  
![Pure_NQR - Evolved_State](Figures_README/Pure_NQR_Evolved_State.png)

In this case, the frequencies of transition of the system have same modulus but opposite sign, namely 1 and -1 MHz. This means that both the rotating waves that make up the linearly polarized pulse are able to induce transitions. In order to visualize properly both the positive and negative resonance lines in the spectrum, the functions for the analysis of the FID must be run with the following parameters:
```
f, ft, ft_n = legacy_fourier_transform_signal(t, fid, 0.5, 1.5, opposite_frequency=True)

plot_fourier_transform(f, ft, ft_n)
```
![Pure_NQR - NMR_Spectrum](Figures_README/Pure_NQR_NMR_Spectrum.png)

## Acknowledgements

The program presented above was made possible thanks to professors Samuele Sanna (Università di Bologna) and Vesna Mitrovic (Brown University), who have been a great help in the interpretation of the physics simulated by the software.

Further contributions to the development of the program have been the advices given by professor Enrico Giampieri (Università di Bologna).
