# Working with the catalogues

Let's have a go at loading in some data! We'll start with the catalogues, as we don't need `pyread_eagle` to load these in - just the python module `h5py`.

## Loading in catalogues

If you remember, the catalogues are split across many files. To load in a dataset from them, we must load it in from each file and append it to a numpy array in a loop. Here's how I would go about loading in `Group_M_Crit200` from the `FOF` table:

```python
# Import modules. This basic example requires only numpy and h5py
import numpy as np
import h5py as h5


# It's a good idea to create variables for the simulation details at the top
sim = 'L0025N0376'      # Initial conditions
model = 'REFERENCE'     # Model
tag = '019_z001p004'    # Snapshot identifier, or 'tag' as people often call it


# Establish the table we want to load in from, and what we want to load
table = 'FOF'
quantity = 'Group_M_Crit200'

# Remember the catalogues have filenames like 'eagle_subfind_tab_019_z001p004.0.hdf5'
# That number at the end is what we want to loop over
# Let's establish the majority of the file path as a string to start with
subfindfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/' + 'groups_'+tag+'/eagle_subfind_tab_' + tag + '.'

# I like to loop over an integer called file_ind in a while loop, incrementing it each time a file is loaded
file_ind = 0    # Initialise the file index

while True:

    try:
        # Use h5py to initialise the catalogue file
        with h5.File(subfindfile+str(file_ind)+'.hdf5', 'r') as f:
            
            # h5py lets you access the contents like a dictionary. Load in the '/FOF/Group_M_Crit200' dataset
            data = f['/'+table+'/%s'%(quantity)]                        

            if file_ind == 0:
                # If this is the first file, convert the dataset into a numpy array
                data_arr = np.array(data)
            else:
                # If this isn't the first file, add the loaded data to the array
                data_arr = np.append(data_arr,np.array(data),axis=0)

    # When we run out of files to load in, an OS exception will be raised.
    # Catch it to finish loading in files and break from the loop.
    except OSError:
        print('Run out of files after loading ',file_ind)
        break

    # Increment the file index
    file_ind += 1
        
# Print the loaded data!
print(data_arr)
```

This code is totally general - you could set `table` to `'Subhalo'` instead and load something in from the SUBFIND catalogue. Try running it for yourself!

A couple of notes:

- Remember to run this code on one of the cluster machines and **not** on `external1/2`.

- We're working with the `Ref-L0025N0376` simulation here. It's always best to prototype and test your code on the smaller simulation volumes first, before going for the big `Ref-L0100N1504` box. The 100 Mpc volume is 16 times larger than the 25 Mpc volumes, so things will take at least 16 times as long to run - likely even longer as things probably won't scale perfectly!

- I recommend using `h5py` in conjunction with a `with` statement, as shown above, for speed.

- You need to do that step of converting the loaded data into a numpy array. `data` here is an HDF5 object, which can be sliced and indexed, but can't be fully manipulated as a regular numpy array.


## Unit conversions

If you successfully ran the code above, you'll notice that the data we loaded doesn't look like it corresponds to halo mass. We need to convert the units - right now we're in h-less comoving GADGET units. Conveniently, all the conversion factors we need can be loaded as attributes from our hdf5 file, as I discussed in the previous section. Lets modify the above example so it does unit corrections automatically:

```python
import numpy as np
import h5py as h5

sim = 'L0025N0376'
model = 'REFERENCE'
tag = '019_z001p004'

table = 'FOF'
quantity = 'Group_M_Crit200'

subfindfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/' + 'groups_'+tag+'/eagle_subfind_tab_' + tag + '.'

file_ind = 0

while True:

    try:
        with h5.File(subfindfile+str(file_ind)+'.hdf5', 'r') as f:
            
            data = f['/'+table+'/%s'%(quantity)]                        

            if file_ind == 0:

                # We can grab all our conversion factors from the first file...
                h_scale_exponent = data.attrs['h-scale-exponent']
                a_scale_exponent = data.attrs['aexp-scale-exponent']
                cgs_conversion_factor = data.attrs['CGSConversionFactor']

                # ... and get the expansion factor and little h from the header
                h = f['Header'].attrs['HubbleParam']
                a = f['Header'].attrs['ExpansionFactor']

                print('Loading ',quantity)
                print('h exponent = ',h_scale_exponent)
                print('a exponent = ',a_scale_exponent)
                print('cgs conversion factor = ',cgs_conversion_factor)
                print('h = ',h)
                print('a = ',a)

                data_arr = np.array(data)

            else:
                data_arr = np.append(data_arr,np.array(data),axis=0)

    except OSError:
        print('Run out of files after loading ',file_ind)
        break

    file_ind += 1
        
# Multiply by h and a, raised to the correct powers, to get the answer in physical GADGET units
M200_gadgetunits = data_arr * np.power(h,h_scale_exponent) * np.power(a,a_scale_exponent)

# Multiply that by the cgs conversion factor to get the answer in grams
M200_cgs = M200_gadgetunits * cgs_conversion_factor

# Alternatively, recall that the unit mass is 10^10 solar masses, so to get M200 in solar masses
M200_sol = M200_gadgetunits * 1e10

print(M200_gadgetunits)
print(M200_cgs)
print(M200_sol)
```
Since we're at redshift 1, you'll see in the output that the expansion factor `a` is about 0.5, however since we're loading in a mass, no correction was needed for expansion factor and `a_scale_exponent` was 0. We did, however, need to multiply by `h^-1` to get a physical mass.

## A function for easy catalogue reading

Clearly, it would be a total pain to do all the above every time we wanted to load something from the catalogues. Since all the appropriate conversion factors can be easily obtained, however, it's dead easy to wrap this up in a function that automates the process. It would look something like this:

```python
def catalogue_read(table,quantity,
                    sim = 'L0025N0376',
                    model = 'REFERENCE',
                    tag = '028_z000p000',
                    data_location = '/hpcdata0/simulations/EAGLE/',
                    phys_units=True,
                    cgs_units=False,
                    verbose=False):

    # Do a quick check to make sure a valid table has been specified
    assert table in ['FOF','Subhalo'],'table must be either FOF or Subhalo'

    subfindfile = data_location + sim + '/' + model + '/data/' + 'groups_'+tag+'/eagle_subfind_tab_' + tag + '.'

    file_ind = 0
    while True:

        try:
            with h5.File(subfindfile+str(file_ind)+'.hdf5', 'r') as f:
                                    
                data = f['/'+table+'/'+quantity]

                if file_ind == 0:

                    h_scale_exponent = data.attrs['h-scale-exponent']
                    a_scale_exponent = data.attrs['aexp-scale-exponent']
                    cgs_conversion_factor = data.attrs['CGSConversionFactor']
                    h = f['Header'].attrs['HubbleParam']
                    a = f['Header'].attrs['ExpansionFactor']

                    # Let's only print this stuff out if the user wants it!
                    if verbose:
                        print('Loading ',quantity)
                        print('h exponent = ',h_scale_exponent)
                        print('a exponent = ',a_scale_exponent)
                        print('cgs conversion factor = ',cgs_conversion_factor)
                        print('h = ',h)
                        print('a = ',a)

                    # Lets be a bit more cautious and make sure we cast our HDF5 dataset into an array of the correct type
                    dt = data.dtype
                    data_arr = np.array(data,dtype=dt)
            
                else:
                    data_arr = np.append(data_arr,np.array(data,dtype=dt),axis=0)

        except OSError:
            print('Run out of files after loading ',file_ind)
            break

        file_ind += 1
    
    # If the data we're loading is integer-type, no corrections will be needed
    if np.issubdtype(dt,np.integer):
        return data_arr

    # Otherwise, do unit corrections
    else:

        if phys_units:
            data_arr *= np.power(h,h_scale_exponent) * np.power(a,a_scale_exponent)

        if cgs_units:

            # cgs numbers can be huge and overflow np.float32
            # Recast the data to float64 to be safe
            data_arr = np.array(data_arr,dtype=np.float64)

            data_arr *= cgs_conversion_factor

        return data_arr
```

A few things to point out here:

- As you can see, the user need only specify the table (`FOF` or `Subhalo`) and the quantity they want to load, and make sure they specify the correct simulation details. I've set the present-day Ref-L0025N0376 snapshot as the default.

- By default, the data will be loaded in physical GADGET units. `cgs_units=True` can be set to return the answer in CGS.

- Note that I now obtain the correct data type from the HDF5 dataset and make sure to cast the data into an array of the correct type. This is important for loading quantities that are integers (such as `GroupNumber`), as simply using `np.array()` will convert these into floats.

- When converting to cgs units, I recast the data to `np.float64` to be safe. Asking for the mass of a DM halo in grams will return an extremely large number which will overflow a 32-bit float and return `inf`, so it's essential to do this.

With this function in hand, it's now easy to load whatever you like from the catalogues. For example, the following script will produce the same output as we got in the previous section:

```python
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '019_z001p004'

M200_gadgetunits = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag)
M200_cgs = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag,cgs_units=True)
M200_sol = M200_gadgetunits * 1e10

print(M200_gadgetunits)
print(M200_cgs)
print(M200_sol)
```

Put this function somewhere safe, as we'll be using it throughout the rest of this guide. If you have a directory in which you're saving and running these example scripts, create a file containing only this function (and the relevant import statements at the top) as `catalogue_reading.py`. That way, we can access it in other scripts in the same directory with ``` from catalogue_reading import *```.

## Constructing samples of galaxies/subhaloes

The primary appeal of cosmological simulations is that they allow us to study relatively large samples of galaxies in cosmologically representative volumes. You'll therefore almost always be utilising the catalogues to construct samples of haloes/galaxies that fit certain criteria, which you then explore further with the catalogues or particle data. This will often involve matching between the `FOF` and `Subhalo` tables. In this section we'll look at a couple of examples of how to do this.

### Central galaxies of haloes in a given mass range

Often, you'll likely want to take haloes within a certain mass range and investigate the properties of their central galaxies. To perform a "mass cut" in the catalogues, you can do something like this:

```python
import numpy as np
from catalogue_reading import *

# Let's stick with the z=0 Ref-L0025N0376 snapshot
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

# Load in the halo masses from the FOF catalogue
M200 = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag) * 1e10

# The group numbers are simply an ascending sequence from 1 to N_groups
groupnumbers = np.arange(len(M200)) + 1

print(len(M200),' haloes in catalogue')
print(M200)
print(groupnumbers)

mass_selection = np.where(M200>np.power(10.,11.5))[0]

M200 = M200[mass_selection]
groupnumbers = groupnumbers[mass_selection]    

print(len(groupnumbers),' satisfy mass selection')
```
Here we've loaded in the halo masses of every FOF group in the snapshot, initialised what their group numbers are, and used `np.where` to mask these arrays, returning a sample of haloes where `M200` is greater than 10^11.5 solar masses. If you run this code, you'll see that of the 30161 groups in the snapshot, only 83 are more massive than this threshold; the overwhelming majority of the groups are far lower-mass. In fact, a large fraction of the groups are 'empty' and contain no subhaloes (bound structures) - this happens because FOF is a relatively simple linking algorithm, which will still assign unbound 'fuzz' particles to a group, even if that group contains only that one particle. Such systems have `NumOfSubhalos=0` in the `FOF` table.

Now we have our halo sample identified in the `FOF` table, we can get the properties of its central galaxy from the `Subhalo` table. Say we wanted the maximum circular velocity (V_max) of the central subhaloes in each group:
```python
# FirstSubhaloID tells us where the central galaxy of each FOF halo is in the Subhalo table
first_subhalo = catalogue_read('FOF','FirstSubhaloID',sim=sim,model=model,tag=tag)

# Load in the maximum circular velocities of all subhaloes in the snapshot (units: km/s)
vmax = catalogue_read('Subhalo','Vmax',sim=sim,model=model,tag=tag)

print(len(vmax),' subhaloes in catalogue')

# Cut down the subhalo catalogue to just the centrals
vmax = vmax[first_subhalo]

# Now this array corresponds exactly to the FOF catalogue, and they have the same length
print(len(vmax),' of them are centrals')

# We can therefore apply the same mass selection 
vmax = vmax[mass_selection]

print(vmax)
```
Alternatively, we could look for every subhalo with a `SubGroupNumber` of 0 to find all the centrals:
```python
subgroupnumbers = catalogue_read('Subhalo','SubGroupNumber',sim=sim,model=model,tag=tag)

central = np.where(subgroupnumber==0)[0]

vmax = catalogue_read('Subhalo','Vmax',sim=sim,model=model,tag=tag)[central][mass_selection]

print(vmax)
```
would be equivalent. However, this requires an extra `np.where` operation, which can be rather slow, so it's best to use the `FirstSubhaloID`.

When it comes to working with particles, you'll most often want to load in the `CentreOfPotential` (COP) for the central subhalo, as this is the best descriptor of "where the galaxy/halo is". It's defined as the location of the most tightly-bound particle in the subhalo. This is done in the same way as above, but here we'll combine the steps into one line:
```python
COP = catalogue_read('Subhalo','CentreOfPotential',sim=sim,model=model,tag=tag)[first_subhalo,:][mass_selection,:]

print(COP)
print(np.shape(COP))
```
Again, we've loaded the COP of all subhaloes, reduced the sample down to just centrals, then applied our mass selection. Note the indexing - COP is a shape (N,3) array, as it includes x, y and z co-ordinates. The shape of the output array is printed above - you should find that it's (83,3).

### Galaxies in a given stellar mass range and their host haloes

What if we want to go the other way, and take galaxies of a certain stellar mass and find their host haloes? To do this, we can make use of the `GroupNumber`, stored for every subhalo.
```python
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

# Get the FOF group number for every subhalo
# These will not be unique as many subhaloes will share the same parent halo
groupnumbers = catalogue_read('Subhalo','GroupNumber',sim=sim,model=model,tag=tag)

# Get the stellar masses. We do this in a spherical aperture of 30 proper kpc
# This dataset in the Subhalo catalogue includes the mass of all particle types in a (N,6) array
# We want stars - PartType4 - so note the indexing at the end
# Just like M200, we multiply by 1e10 to get the answer in solar masses
Mstar_30kpc = catalogue_read('Subhalo','ApertureMeasurements/Mass/030kpc',sim=sim,model=model,tag=tag)[:,4] * 1e10

# Make a stellar mass selection using np.where
stellar_mass_selection = np.where(Mstar_30kpc>np.power(10.,9.5))[0]

# Cut down the samples
Mstar_30kpc = Mstar_30kpc[stellar_mass_selection]
groupnumbers = groupnumbers[stellar_mass_selection]

# Load in the halo masses from the whole FOF catalogue
M200 = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag) * 1e10

# Remember that GroupNumber just corresponds to the position in the FOF catalogue
# The numbers run from 1 to N, so we need to subtract 1 to get the index
M200 = M200[groupnumbers-1]

print(M200)
print(len(M200),' galaxies satisfy this mass cut')
```
As you can see, we can simply use the `GroupNumber` from the `Subhalo` table to index the `FOF` table and find the parent haloes.

**A very common mistake is to forget that `GroupNumber` runs from 1 to N, and not from 0 to N-1. When using `GroupNumber` as an _index_ for the `FOF` table, you must remember to subtract 1 from it as above. You only need to get your index wrong by 1 to completely screw up your results!**

Note that to get the stellar mass, we could have done
```python
catalogue_read('Subhalo','MassType',sim=sim,model=model,tag=tag)[:,4]
```
instead, and obtained the mass of all stars bound to the subhalo. Here's a good explanation from the EAGLE team (specifically from [McAlpine et al. (2016)](https://arxiv.org/abs/1510.01320) for why you should use the aperture-based method instead:

_The stripping of satellite galaxies as they orbit within a halo generates a significant mass loss at large radii. The resulting diffuse light (and any diffuse star formation) is extremely difficult to observe and is not commonly included in observational galaxy catalogues. Furthermore, the total galaxy stellar masses and star formation rates can depend  strongly  on the  precise  assignment  of  particles  to  the  main subhalo within each FOF group by the SUBFIND algorithm, which can lead to spurious total mass evolution. For these reasons, studies published by the EAGLE team use aperture masses and star formation rates, typically in an aperture of 30 pkpc.  As discussed by Schaye et al. (2015), this corresponds roughly to an R_80 Petrosian aperture and is hence particularly well-suited to comparison with observations. We recommend the use of aperture values when available._

I'll give further examples of how to use these techniques on the "Common EAGLE tasks" page. For now, let's move on to working with particles!
