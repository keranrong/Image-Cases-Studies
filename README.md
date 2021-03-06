# Image Cases Studies
Python prototypes for image processing methods

© 2017 - Aurélien Pierre, Polytechnique Montréal.

## Presentation

### Motivation

This collection of scripts is intended to prototype methods and functionalities that
could be useful in [darktable](https://github.com/darktable-org/darktable) and
show proofs of concept.

### Intent

While most studies in digital imagery and signal processing focus on making things possible,
investigating options, analysing SNR and computing errors and convergence, they tend to produce unnatural results
with artifacts, looking unpleasant to the eye. 

This work tries to implement state-of-the-art algorithms into a realistic framework for photography and photographers, 
realistic both in terms of computation time and of photographic results. 

### How it's made

It's written in Python 3, and relies on PIL (Python Image Library) for the I/O, Numpy for the arrays
operations, and Numba to optimize the execution time. Heavy arrays operations 
are parallelized through multiprocesses but can be run serialized as well.


Every function is timed natively, so you can benchmark performance. 

### What's inside

For now, we have :

* Filters :
    * Gaussian blur
    * Bessel blur (Kaiser denoising)
    * Bilateral filter
    * Unsharp mask
    * Richardson-Lucy blind and non-blind deconvolution with Total Variation regularization
        * method TV-PAM : http://www.cvg.unibe.ch/dperrone/tvdb/index.html
        * method TV-PD (work in progress) : http://www.cvg.unibe.ch/dperrone/logtv/index.html
        * method TV-MM : ibid.
* Windows/Kernels : (*for convolution and frequential analysis*)
    * Poisson/exponential
    * Kaiser-Bessel
    * Gauss
    * Uniform
    
A collection of test pictures is in `img` directory and the converted pictures
are in `img` subfolders. The built-in generic functions are in the `lib.utils` module.

### What may come one day

* Refocusing : 
    * http://web.media.mit.edu/~bandy/refocus/PG07refocus.pdf
    * http://manao.inria.fr/perso/~ihrke/Publications/wscg12.pdf
    * https://users.soe.ucsc.edu/~milanfar/publications/journal/dbe_tip_final.pdf
* TV denoise : http://scikit-image.org/docs/dev/auto_examples/filters/plot_denoise.html
* Markov-chain Monte-Carlo denoise : http://www.sciencedirect.com/science/article/pii/S016516841100096X
* Lens PSF calibration : 
    * http://www.cim.mcgill.ca/~langer/MY_PAPERS/MannanLanger-CRV16-DFDCalib.pdf
    * http://people.csail.mit.edu/yichangshih/lensEnhancement/lensFittingEccv_camera_ready.pdf
    * http://people.csail.mit.edu/sparis/publi/2011/iccp_blur/Kee_11_Optical_Blur.pdf
    * http://wwwuser.gwdg.de/~uboehm/images/25.pdf
    * http://www.cs.ubc.ca/labs/imager/tr/2013/SimpleLensImaging/SimpleLensImaging_Heide2013.pdf
    
### Current prototypes

#### Richardson-Lucy deconvolution

##### Overview

In theory, blurred and noisy pictures can be perfectly sharpened if we perfectly 
know the [*Point spread function*](https://en.wikipedia.org/wiki/Point_spread_function) 
of their maker. In practice, we can only estimate it.
One of the means to do so is the [Richardson-Lucy deconvolution](https://en.wikipedia.org/wiki/Richardson%E2%80%93Lucy_deconvolution).

Deconvolution differs from usual sharpness filters : while high-pass filters and unsharp masks increase the local contrast
(thus the *percieved* sharpness), deconvolution actually recovers the stiffness of the edges and can recover motion blur.

The main drawback of this method is the creation of artifacts in the deblured picture, such as ringing and periodic 
replication of the edges. The modified Richardson-Lucy algorithm used here is modified to implement [Total Variation regularization
](http://www.cs.sfu.ca/~pingtan/Papers/pami10_deblur.pdf). It can run in a blind or non-blind fashion, meaning that the 
PSF can be passed as an user-input or estimated from scratch using bayesian statistics and refined during the process.

These algorithms are all auto-adaptative, meaning that all the regularization parameters are
estimated by the algorithm based on statistical assumptions. However, as stats don't always represent
the reality, the user can still force his own parameters.

##### Math details

Let the blurry image be the convolution product of a sharp image and a kernel (the PSF) plus a noise matrix.
The method used hire aims at finding the sharp image by a gradient descend method, following the Richardson-Lucy algorithm.
However, both the PSF and the sharp image are unknown, leading to an ill-posed problem. The challenge is yet
to find a convex problem to solve this problem in a Sobolev space.

Perrone & Favaro base their method on assuming that a sharp image has the logarithms of its gradients following a
Cauchy distribution. Therefore, at every step of the gradient descent, it is possible to compute a penalty for
every pixel not following this law, and attenuate these pixels from the intermediate solution. This penalty is such as 
the L-1 anisotropic norm of the Total Variation is minimized at every iteration, leading to noise and ringing
attenuation. In the blind setup, the initial PSF is assumed to be an uniform blur, and then refined at every step along with the image.

While the theory says that minimizing gradients cannot bring back a sharper image (whose gradients are maximum), the practice shows
that it actually works. The actual implementation relies on 3 different solvers : the Projected Alternating Minimization,
the Primal-Dual method (similar to A. Chambolle's TV denoising), and the Majorization-Minimizatiom.

##### Contributions

This work improves the used references in several ways :

The regularization has been modified from the original work to include a [collaborative L^{infinite, 1, 1} RGB total variation norm](http://www.ipol.im/pub/art/2016/141/)
as the regularization term. It acts as handcuffs between channels and helps dramatically in chromatic noise removal.

Also, the gradients are computed in 2D over the 9 neighbouring pixels (over 3 channels) with a 2D filter giving
a second-order accurate approximation of the gradient.

This improves greatly the convergence when noise is present and when the PSF guess is oversized. In these cases, 
the algorithms often never converges and only creates ringging.

In auto mode, after the initial PSF evaluation along the whole picture, this work allows the user to select a specific zone
to refine the PSF. This helps in cases where the PSF varies spatially, to constrain the in-focus zone, but also speed-up the computations.

This implementation has a stopping criterion which stops the gradient descent when convergence is reached or when the gradient
begins to increase. This criterion prevents the degeneration of the solution as well as useless computations.

The deconvolution has 2 separate steps : a first one, blind, to determine the PSF, and a second one, non-blind, to deblur the picture.
This is useful because : 

* the PSF can be computed on a region of the picture only, to ensure fast computations,
* the PSF can then be saved to be used elsewhere, and checked on a plot before running the full and slow deconvolution,
* the convergence criterion is always reached sooner on the PSF than the one on the picture. 

The convolutions are done in the frequency domain using FFTW. The FFTW wisdoms are saved and reused, meaning that the system gets
faster as it performs new work.

##### Results
The tests have been done on a laptop with an Intel® Core™ i7-2670QM CPU @ 2.20GHz running 8 processes.

###### Blurred original :
![alt text](img/blured.jpg)

###### After (blind algorithm, MM method - 189 s - 325 iterations - Blind.):
This is the implementation of the Majorization-Minimisation algorithm proposed by [Perrone & Favaro in 2015](http://www.cvg.unibe.ch/dperrone/logtv/index.html).
The computations are much slower that's why the PSF is only estimated on a patch of the picture.
This method has found the sharp picture at a margin of error of 5 % in more than 50 % of the tests. The PAM method never reaches
the sharp picture, but comes close enough.
![alt text](img/richardson-lucy-deconvolution/blured-v26.jpg)

This method deblurs by recovering the sharpness of the edges. However, it does not recover the local contrast. Further
edition of the above picture with local contrast added through wavelets high-pass filter
and a laplacian filter is advised.

###### Sharp image

This is a detail from a 24 Mpx "sharp" image taken with a consumer zoom lens, rather soft.
![alt text](img/153412-crop.jpg)

Its result deblured in 18 min :
![alt text](img/richardson-lucy-deconvolution/153412-v20-crop.jpg)


## Installation

It's not recommended to install this *unstable* framework on your Python environnement, but rather to build
its modules and use it from its directory.

    python setup.py build_ext --inplace

On Linux systems, if you have Python 2 and 3 interpreters installed together, you may run :

    python3 setup.py build_ext --inplace
    
The Python interpreter should be in the 3.x serie. Python 2 will trigger `ZeroDivisionError` since it performs
integer divisions by default.
    
Unfortunately, the setup file has been reported defective so in most cases, the dependencies will
not be automatically installed.

To solve this problem until an elegant solution is found, the simpliest way is to first install the [Anaconda Python distribution](https://www.anaconda.com/download/)
which is a bundle of Python packages for scientific computation and signal processing.

Then, ensure the following packages are installed :

    PIL (known as pillow)
    numba
    scipy 
    numpy (normally included into scipy)
    sympy
    skimage
    pyfftw

    
## Use

### In console

Execute :

```shell
 python3 richardson_lucy_deconvolution.py 
```

Import the required Python packages : 

```python
from lib import utils # if you are working directly in the package directory
from PIL import Image 
import numpy as np
from skimage import color
```
    
Load an image as a 3D RGB array:

```python
with Image.open("path/image") as pic:

        pic = np.array(pic).astype(float)
```
    
Set/Reset RGB channels 

```python
pic[..., 0] = numpy.array([...]) # sets the R channel with a 2D numpy array
pic[..., 1] = numpy.array([...]) # sets the G channel with a 2D numpy array
pic[..., 2] = numpy.array([...]) # sets the B channel with a 2D numpy array
```
    
    
Blur a channel : 

    i = # 0, 1 or 2
    pic[..., i] = utils.bilateral_filter(pic[..., i], 10, 6.0, 3.0)
    
Save the picture :
    
```python
with Image.fromarray(pic) as output:

    output.save("file.jpg")
```
    
See the scripts in the root directory for real examples.
