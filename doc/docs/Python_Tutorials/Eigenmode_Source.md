---
# Eigenmode Source
---

This tutorial demonstrates using the [`EigenModeSource`](../Python_User_Interface.md#eigenmodesource) to launch a *single* eigenmode propagating in just one direction. Examples are provided for two kinds of eigenmodes in lossless, dielectric media: (1) localized (i.e., guided) and (2) non-localized (i.e., radiative planewave).

[TOC]

Index-Guided Modes in a Ridge Waveguide
---------------------------------------

The first structure, shown in the schematic below, is a 2d ridge waveguide with ε=12, width $a$=1 μm, and out-of-plane electric field E<sub>z</sub>. The dispersion relation ω(k) for index-guided modes with *even* mirror symmetry in the y-direction is computed using [MPB](https://mpb.readthedocs.io/en/latest/) and shown as blue lines. The light cone which denotes radiative modes is the section in solid green. Using this waveguide configuration, we will investigate two different frequency regimes: (1) single mode (normalized frequency of 0.15) and (2) multi mode (normalized frequency of 0.35), both shown as dotted horizontal lines in the figures. We will use the eigenmode source to excite a specific mode in each case &mdash; labeled **A** and **B** in the band diagram &mdash; and compare the results to using a constant-amplitude source for straight and rotated waveguides. Finally, we will demonstrate that a single monitor plane in the y-direction is sufficient for computing the total Poynting flux in a waveguide with any arbitrary orientation.

<center>
![](../images/eigenmode_source.png)
</center>

The simulation script is in [examples/oblique-source.py](https://github.com/NanoComp/meep/blob/master/python/examples/oblique-source.py). The notebook is [examples/oblique-source.ipynb](https://nbviewer.jupyter.org/github/NanoComp/meep/blob/master/python/examples/oblique-source.ipynb).

The simulation consists of two separate parts: (1) computing the flux and (2) plotting the field profile. The field profile is generated by setting the flag `compute_flux=False`. For the single-mode case, a constant-amplitude current source (`eig_src=False`) excites both the waveguide mode and radiating fields in both directions (i.e., forwards and backwards). This is shown in the main inset of the first of two figures above. The `EigenModeSource` excites only the forward-going waveguide mode **A** as shown in the smaller inset. Exciting this mode requires setting `eig_src=True`, `fsrc=0.15`, `kx=0.4`, and `bnum=1`. Note that `EigenModeSource` is a line centered at the origin extending the length of the entire cell. The constant-amplitude source is a line that is slightly larger than the waveguide width. The parameter `rot_angle` specifies the rotation angle of the waveguide axis and is initially 0° (i.e., straight or horizontal orientation). This enables `eig_parity` to include `EVEN_Y` in addition to `ODD_Z` and the simulation to include an overall mirror symmetry plane in the y-direction.

For the multi-mode case, a constant-amplitude current source excites a superposition of the two waveguide modes in addition to the radiating field. This is shown in the main inset of the second figure above. The `EigenModeSource` excites only a given mode: **A** (`fsrc=0.35`, `kx=0.4`, `bnum=2`) or **B** (`fsrc=0.35`, `kx=1.2`, `bnum=1`) as shown in the smaller insets.

```py
import meep as mp
import numpy as np
import matplotlib.pyplot as plt

resolution = 50 # pixels/μm

cell_size = mp.Vector3(14,14,0)

pml_layers = [mp.PML(thickness=2)]

# rotation angle (in degrees) of waveguide, counter clockwise (CCW) around z-axis
rot_angle = np.radians(20)

geometry = [mp.Block(center=mp.Vector3(),
                     size=mp.Vector3(mp.inf,1,mp.inf),
                     e1=mp.Vector3(1).rotate(mp.Vector3(z=1), rot_angle),
                     e2=mp.Vector3(y=1).rotate(mp.Vector3(z=1), rot_angle),
                     material=mp.Medium(epsilon=12))]

fsrc = 0.15 # frequency of eigenmode or constant-amplitude source
kx = 0.4    # initial guess for wavevector in x-direction of eigenmode
bnum = 1    # band number of eigenmode

compute_flux = True # compute flux (True) or plot the field profile (False)

eig_src = True # eigenmode (True) or constant-amplitude (False) source

if eig_src:
    sources = [mp.EigenModeSource(src=mp.GaussianSource(fsrc,fwidth=0.2*fsrc) if compute_flux else mp.ContinuousSource(fsrc),
                                  center=mp.Vector3(),
                                  size=mp.Vector3(y=14),
                                  direction=mp.AUTOMATIC if rot_angle == 0 else mp.NO_DIRECTION,
                                  eig_kpoint=mp.Vector3(kx).rotate(mp.Vector3(z=1), rot_angle),
                                  eig_band=bnum,
                                  eig_parity=mp.EVEN_Y+mp.ODD_Z if rot_angle == 0 else mp.ODD_Z,
                                  eig_match_freq=True)]
else:
    sources = [mp.Source(src=mp.GaussianSource(fsrc,fwidth=0.2*fsrc) if compute_flux else mp.ContinuousSource(fsrc),
                         center=mp.Vector3(),
                         size=mp.Vector3(y=2),
                         component=mp.Ez)]

sim = mp.Simulation(cell_size=cell_size,
                    resolution=resolution,
                    boundary_layers=pml_layers,
                    sources=sources,
                    geometry=geometry,
                    symmetries=[mp.Mirror(mp.Y)] if rot_angle == 0 else [])

if compute_flux:
    tran = sim.add_flux(fsrc, 0, 1, mp.FluxRegion(center=mp.Vector3(x=5), size=mp.Vector3(y=14)))
    sim.run(until_after_sources=50)
    print("flux:, {:.6f}".format(mp.get_fluxes(tran)[0]))
else:
    sim.run(until=100)
    nonpml_vol = mp.Volume(center=mp.Vector3(), size=mp.Vector3(10,10,0))
    eps_data = sim.get_array(vol=nonpml_vol, component=mp.Dielectric)
    ez_data = sim.get_array(vol=nonpml_vol, component=mp.Ez)
    plt.figure()
    plt.imshow(np.flipud(np.transpose(eps_data)), interpolation='spline36', cmap='binary')
    plt.imshow(np.flipud(np.transpose(ez_data)), interpolation='spline36', cmap='RdBu', alpha=0.9)
    plt.axis('off')
    plt.show()
```

Note that in the eigenmode source, `direction` must be set to `NO_DIRECTION` for a non-zero `eig_kpoint` which specifies the waveguide axis.

Additionally, we can demonstrate the eigenmode source for a rotated waveguide. The results are shown in the two figures below for the single- and multi-mode case. There is one subtlety: for mode **A** in the multi-mode case, the `bnum` parameter is set to 3 rather than 2. This is because a non-zero rotation angle breaks the symmetry in the y-direction which therefore precludes the use of `EVEN_Y` in `eig_parity`. Without any parity specified for the y-direction, the second band corresponds to *odd* modes. This is why we must select the third band which contains even modes. An oblique waveguide also leads to a breakdown in the [PML](../Perfectly_Matched_Layer.md). A simple workaround for mitigating the PML artifacts is to increase its length which is why the `thickness` has been doubled from 1 to 2.

<center>
![](../images/oblique_source_singlemode.png)
</center>

There are numerical dispersion artifacts due to the FDTD spatial and temporal discretizations which create negligible backward-propagating waves by the eigenmode current source, carrying approximately 10<sup>-5</sup> of the power of the desired forward-propagating mode. These artifacts can be seen as residues in the field profiles.

<center>
![](../images/oblique_source_multimode.png)
</center>

Finally, we demonstrate that the total power in a waveguide with *arbitrary* orientation can be computed by a single flux plane oriented along the y direction: thanks to Poynting's theorem, the flux through any plane crossing a lossless waveguide is the same, regardless of whether the plane is oriented perpendicular to the waveguide. Furthermore, the eigenmode source is normalized in such a way as to produce the same power regardless of the waveguide orientation — in consequence, the flux values for mode **A** of the single-mode case for rotation angles of 0°, 20°, and 40° are 77.000266, 76.879339, and 76.817850, within 0.2% (discretization error) of one another.

Planewaves in Homogeneous Media
-------------------------------

The eigenmode source can also be used to launch planewaves in homogeneous media. The dispersion relation for a planewave is ω=|$\vec{k}$|/$n$ where ω is the angular frequency of the planewave and $\vec{k}$ its wavevector; $n$ is the refractive index of the homogeneous medium. This example demonstrates launching planewaves in a uniform medium with $n$ of 1.5 at three rotation angles: 0°, 20°, and 40°. Bloch-periodic boundaries via the `k_point` are used and specified by the wavevector $\vec{k}$. PML boundaries are used only along the x-direction.

The simulation script is in [examples/oblique-planewave.py](https://github.com/NanoComp/meep/blob/master/python/examples/oblique-planewave.py). The notebook is in [examples/oblique-planewave.ipynb](https://nbviewer.jupyter.org/github/NanoComp/meep/blob/master/python/examples/oblique-planewave.ipynb).

```py
import meep as mp
import numpy as np
import matplotlib.pyplot as plt

resolution = 50 # pixels/μm

cell_size = mp.Vector3(14,10,0)

pml_layers = [mp.PML(thickness=2,direction=mp.X)]

# rotation angle (in degrees) of planewave, counter clockwise (CCW) around z-axis
rot_angle = np.radians(0)

fsrc = 1.0 # frequency of planewave (wavelength = 1/fsrc)

n = 1.5  # refractive index of homogeneous material
default_material = mp.Medium(index=n)

k_point = mp.Vector3(fsrc*n).rotate(mp.Vector3(z=1), rot_angle)

sources = [mp.EigenModeSource(src=mp.ContinuousSource(fsrc),
                              center=mp.Vector3(),
                              size=mp.Vector3(y=10),
                              direction=mp.AUTOMATIC if rot_angle == 0 else mp.NO_DIRECTION,
                              eig_kpoint=k_point,
                              eig_band=1,
                              eig_parity=mp.EVEN_Y+mp.ODD_Z if rot_angle == 0 else mp.ODD_Z,
                              eig_match_freq=True)]

sim = mp.Simulation(cell_size=cell_size,
                    resolution=resolution,
                    boundary_layers=pml_layers,
                    sources=sources,
                    k_point=k_point,
                    default_material=default_material,
                    symmetries=[mp.Mirror(mp.Y)] if rot_angle == 0 else [])

sim.run(until=100)

nonpml_vol = mp.Volume(center=mp.Vector3(), size=mp.Vector3(10,10,0))
ez_data = sim.get_array(vol=nonpml_vol, component=mp.Ez)

plt.figure()
plt.imshow(np.flipud(np.transpose(np.real(ez_data))), interpolation='spline36', cmap='RdBu')
plt.axis('off')
plt.show()
```

Note that this example involves a `ContinuousSource` for the time profile. For a pulsed source, the oblique planewave is incident at a given angle for only a *single* frequency component of the source. This is a fundamental feature of FDTD simulations and not of Meep per se. Thus, to simulate an incident planewave at multiple angles for a given frequency ω, you will need to do separate simulations involving different values of $\vec{k}$ (`k_point`) since each set of ($\vec{k}$,ω) specifying the Bloch-periodic boundaries and the frequency of the source will produce a different angle of the planewave. For more details, refer to Section 4.5 ("Efficient Frequency-Angle Coverage") in [Chapter 4](https://arxiv.org/abs/1301.5366) ("Electromagnetic Wave Source Conditions") of [Advances in FDTD Computational Electrodynamics: Photonics and Nanotechnology](https://www.amazon.com/Advances-FDTD-Computational-Electrodynamics-Nanotechnology/dp/1608071707).

Shown below are the steady-state field profiles generated by the planewave for the three rotation angles. Residues of the backward-propagating waves due to the discretization are slightly visible.

<center>
![](../images/eigenmode_planewave.png)
</center>
