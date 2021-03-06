# MUSL

MU(LTI)SL(ICE) is a MATLAB class for running multislice electron diffraction simulations of cubic crystals that has been benchmarked against experiments (see the following [Ultramicroscopy paper](https://doi.org/10.1016/j.ultramic.2021.113211) and [Dissertation](https://repository.asu.edu/items/62749)). The code can be used to get the intensities of reflections in a diffraction pattern as well as the resulting real space image. It is also capable of producing intensity versus depth plots. 

## Usage

Example scripts can be found in the [Examples](Examples/) folder; we include some samples here for thoroughness.

To use the class, we must first initial a MUSL object as is done below for a 200 keV electron beam incident on a 200 nm thick slab of Si(001):

````matlab
% lattice constants along each dimension [angstroms]
objLatticeInfo.Lattice = [5.431; 5.431; 5.431]; 
% locations of the atoms in units of the lattice constants.
% we assume we can treat the unit cell as a single layer.
objLatticeInfo.Positions = [0,0,0;              
                            0.5,0.5,0;         
                            0.25,0.25,0;
                            0.75,0.75,0;
                            0.5,0,0;
                            0,0.5,0;
                            0.25,0.75,0;
                            0.75,0.25,0 ];

% atomic number of element at each point of the Positions matrix
objLatticeInfo.AtomicNum = repmat(14,size(objLatticeInfo.Positions,1),1);
objLatticeInfo.BeamEnergy = 200*10^3;   %accelerating potential [Volts]
objLatticeInfo.crystalThickness = 2000; %crystal thickness [angstroms]

AngleX = 0;     %Tilt of the sample along X [mrad]
AngleY = 0;     %Tilt of the sample along Y [mrad]

objMUSL = MUSL(objLatticeInfo);
````

There a many options that can be called during initialization to customize the subsequent runs with the object. For example, the type of Bravais Lattice of the material can be specified using name value pairs

````matlab
objMUSL = MUSL(objLatticeInfo,'BravaisLattice','diamond');
````

as can the temperature of the material, the rotation of the crystal axes relative to the tilt axes of the sample holder, and even whether or not the run will occur or not the run will be on a GPU among other settings:

````matlab
objMUSL = MUSL(objLatticeInfo,...
                'Temperature',293,...
                'BravaisLattice','diamond',...
                'RotationCrystal',-45,...
                'partKMax',0.0133,...
                'partKExtent',3,...
                'NumUnitCells',8,...
                'NumPixels',512,...
                'UseGPU',true);
````

### Simulations

The simulation that records a single depth value can be run using the syntax:

````matlab
[intVal,reflList] = objMUSL.intensity(AngleX,AngleY);

% Table with the reflections and intensities
intensityData = table(reflList,permute(intVal,[1 3 2]));
intensityData.Properties.VariableNames = {'relList' 'intensity'};
````
which can be transformed into a table with the intensities and the corresponding Miller indices.

The image can also be recovered using 

````matlab
im = abs(fftshift(ifft2(objMUSL.Wave)));
````

For the intensity at each step up until the specfied thickness, the argument `depth` is used with the `intensity` method

````matlab
[intVal,reflList] = objMUSL.intensity(AngleX,AngleY,'depth');
````

For the previously mentioned [Ultramicroscopy paper](https://doi.org/10.1016/j.ultramic.2021.113211) and [Dissertation](https://repository.asu.edu/items/62749), intensities at across a grid of sample orientations were simulated 

````matlab
AngleX = linspace(-2.75,3.24,70);     %Tilt of the sample along X [mrad]
AngleY = linspace(10.75,13.72,35);    %Tilt of the sample along Y [mrad]

[X,Y] = meshgrid(AngleX,AngleY);

%timer to check progress of mutlslice step
tStart = tic;

for ii=1:numel(X)
    if(mod(ii,100)==0) %progress displayed
        tElapsed = toc(tStart);
        fprintf('On linear index %d of %d. Elapsed time: %0.1f minutes \n',...
            ii,numel(X),tElapsed/60)
    end
    [intVal{ii},~] = objMUSL.intensity(X(ii),Y(ii));
end
````

### Plotting

The `MUSL` class has built in plotting functions for the above cell output that are suited for non-vector images (`plotMap`) and vector image formats (`plotMapEPS`):

````matlab
fontSize = 12;
figure('Position', [800 100 500 300])
objMUSL.plotMap(X,Y,intVal,[0,0],fontSize)
colorbar
````
![exampleIntensityMap000](images/intensityMapExample000.png )

````matlab
figure('Position', [800 100 500 300])
objMUSL.plotMap(X,Y,intVal,[2,-2],fontSize)
colorbar
````
![exampleIntensityMap220](images/intensityMapExample220.png )

There is also a built in plotting function for the intensity vs depth outputs:

````matlab 
AngleX = 5;     %Tilt of the sample along X [mrad]
AngleY = 5;    %Tilt of the sample along Y [mrad]
objMUSL = MUSL(objLatticeInfo,'BravaisLattice','diamond','RotationCrystal',-45);
[intVal,~] = objMUSL.intensity(AngleX,AngleY,'depth');
````

````matlab
figure
objMUSL.plotIntVsDepth(intVal,[0,0],12)
title('(000)')
````

![exampleIntensityDepth000](images/intensityDepthExample000.png )

````matlab
figure
objMUSL.plotIntVsDepth(intVal,[2,2],12)
title('(220)')
````

![exampleIntensityDepth220](images/intensityDepthExample220.png )

## Documentation

  A `MUSL` object has the following properties that user can be modified using name-value pairs:
  
  - `Lattice`: lattice constant [angstroms] [1x3]
  
  - `Positions`: locations of the atoms in units of the Lattice constants
      [nx3]. Due to the nature of the fft2 used, atoms that appear on the
      edge also appear on the opposite edges of the unit cell, i.e. an
      atom on the left edge appears on the right edge in the same
      position, an atom on one corner appears on all corners. Thus, car
      must be taken to not include unneccesary particles. This can be
      checked by plotting the real space projected potential to see if 
      there are any spikes larger than the rest, which would indicate
      doubled or quadrupled particles.
      
  - `BravaisLattice`: type of Bravais Lattice of sample. This determines with 
      hkl indices are output, that are nonzero. Can be:
          simple-cubic (sc) - any integer miller indices allowed
          body-centered cubic (bcc) - h+k+l even
          face-centered cubic (fcc) - h,k,l all odd or all even
          diamond-fcc (diamond) - all odd, or all even with h+k+l=4n
          
  - `AtomicNum`: atomic number of element at each point of the Positions 
      matrix [nx1]
      
  - `BeamEnergy`: accelerating potential [Volts] [1x1].
  
  - `CrystalThickness`: crystal thickness [angstroms] [1x1]
  
  - `Temperature`: temperature of the sample defaults to 295K [Kelvin] [1x1]

  - `RotationCrystal`: rotation of the crystal plane relative to the x/y axes
      of tilt (ccw) [degrees] [1x1]
      
  - `KMax`: maximum reciprocal lattice vector; determines the number of beams 
      included in the simulation via the potential/transmission function 
      [angstroms^-1] [1x1]
      
  - `partKMax`: for use when partial coherence is included, i.e. beam with 
      angular spread; 0 indicates there is no angular spread [angstroms^-1]
      ParCoIndFind does not function when this value is greater than the
      separation between diffraction spots. Assumes beam has Gaussian
      spatial distribution and that the angular spread is axially 
      symmetric. partKMax is the standard deviation.
      
  - `partKExtent`: Assumes beam has Gaussian spatial distribution. Is the
      number of standard deviations out the beam extends. 
      
  - `NumUnitCells`: number of unit cells in each direction (should be a 
      power of 2 so that it can evenly divide the number of pixels,
      excluding 2^0) [no units] [1x1]
      
  - `NumPixels`: %number of total pixels in along each direction (should be 
      a power of 2 ie 2^n to increase speed of calculation) [no units]
      [1x1].
      
  - `UseGPU`: specfiy whether to use GPU or CPU. [boolean] [1x1]
  
  - `TwoBeamCondition`: if this is nonempty, then it contains the hkl miller
      indices of the reflection that will be used in applying the two
      beam condition - changes the propogator function [2 or 3 x 1]

The following properties are calculated by `MUSL` and cannot be modified directly by the user:
  
  - `Trans`: transmission function of the sample [no units] [mxm]
  
  - `Wave`: wavefunctions of the electrons as it passes through sample [no
      units] [mxm]
      
  - `Lambda`: electron wavelength [angstroms]
  
  - `Absorption`: indicates wheteher absorption is to be included [boolean]
      [1x1]
      
  - `KGrid`: the Kx and Ky values of the grid which fall within partKMax 
  
  - `xPixelSize`: Pixel size in real space (x,y) [angstroms] [1x1]
  
  - `kPixelSize`: Pixel size in inverse momentum space space [angstrom^-1] [1x1]
  
  - `NumLayers`: number of layers in the unit cell; determined by the number
      if unique l values in the Positions matrix [no units] [1x1]
      
  - `ZSpacing`: vertical spacing of the layer in the multislice method
      [angstroms] [1x1]
  
  - `ReflList`: lists the reflections as they are appear in the H,K index
      matrices [cx2]
      
  - `Ind`: gives the linear indices in the wave matrix that are associated
      with each reflection (can be greater than 1 when partial coherence
      is included) [c x 1...d]
      
  - `K2reflection`: gives the inverse space squared value for each
      reflection, which is useful for applying a profile in the case of
      partial coherence. [angstrom^-1] [1x1]
