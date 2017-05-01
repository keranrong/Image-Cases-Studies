 # Image Cases Studies
Python prototypes of image processing methods

## Presentation

### Motivation

This collection of scripts is intended to prototype methods and functionalities that
could be useful in [DarkTable](https://github.com/darktable-org/darktable) and
show proofs of concept.

### How it's made

It's written in Python 3, and relies deeply on PIL (Python Image Library) for the I/O, Numpy for the arrays
operations, and Cython to optimize the execution time. Heavy arrays operations 
are parallelized through multithreading but
can be run serialized as well.

Images are open from 8 bits RGB and stored in a class that keeps LAB and RGB copies,
automatically updating the RGB representation when L, A, or B channels are modified.

Every function is timed natively, so you can benchmark performance. 

The built-in functions are staticly typed and compiled with Cython.

### What's inside

For now, we have :

* Blending modes in LAB:
    * overlay
* Filters in LAB :
    * Gaussian blur
    * Bessel blur (Kaiser denoising)
    * bilateral filter
    * unsharp mask
    
A collection of test pictures is in `img` directory and the converted pictures
are in `img` subfolders.
    
### Current prototypes

#### Unsharp mask with bilateral filter

Using bilateral filter in LAB allows to perform a better unsharp mask without
halos. Run `bilateral_unsharp_mask.py`.

Before :
![alt text](img/original.jpg "Before")

After :
![alt text](img/bilateral-unsharp-mask/original.jpg "After")

## Installation

    python setup.py install

will install the package and its dependencies. On Linux systems, if you have
Python 2 and 3 interpreters installed together, run :

    python3 setup.py install
    
## Use

Import PIL and the library : 

    from lib import utils
    from PIL import Image
    
Load an image :

    with Image.open("path/image") as pic:

            pic = utils.image_open(pic)
    
Then, the LAB channels have `numpy.ndarray` types can be accessed and set from properties :

    pic.L = numpy.array([...]) # sets the L channel with a 2D numpy array
    pic.A = numpy.array([...]) # sets the A channel with a 2D numpy array
    pic.B = numpy.array([...]) # sets the B channel with a 2D numpy array
    
    pic.LAB = numpy.array([...]) # sets the LAB channels with a 3D numpy array
    
When you set/reset an LAB channel, the RGB ones are automatically updated. However,
once set with `utils.image_open()`, the RGB channels are read-only. To override them, you need to create a
new instance :

    pic.RGB = numpy.array([...]) # ERROR
    pic = utils.image_open(pic.RGB + 2) # Set/Reset RGB channels
    

Blur the L channel : 

    pic.L = utils.bilateral_filter(pic.L, 10, 6.0, 3.0)
    
Save the picture :
    
    with Image.fromarray(pic.RGB) as output: #Save the RGB channels

                output.save("file.jpg")
    
