..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Scheduling simulations currently use ESO's skycalc software to pre-calculate sky brightness for the survey. The current storage format, healpix maps of the sky sampled every few minutes, results in inconveniently large files of cached data. Here, I present an exploration of the use of Zernike polynomial coefficients as an alternative.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

Summary
=======

#. Sky brightness is diffuse light spread across the field of view of the camera. See :numref:`fig-healpix-map`.
  
   * Sky brightness includes components from emission from the atmosphere,  scattered light from various source (the Moon, Sun, etc.), and Zodiacal light.
   * Sky brightness varies smoothly over the sky, except near the moon.
   * Sky brightness varies slowly with time, except when the moon is rising or setting.

#. The Rubin Observatory scheduler requires an estimate of the foreground sky brightness over the visible sky at each time for which it schedules exposures.
     
   * The limiting magnitude (which depends on sky brightness) is one of the "features" the feature-based scheduler uses to order candidate exposures.
   * When Poisson noise from the sky background is the dominante source of noise, the error in limiting magnitude due to an error in sky brightness (in magnitudes/asec^2) is half the error in sky brightness. For example, an uncertainty of 0.2 magnitudes in sky brightness corresponds to an uncertainty of 0.1 magnitudes is limiting magnitude.
       
#. The current version of the scheduler uses a lookup table of sky values.

   * The Rubin Obs. scheduler looks up sky brightness values in a set of healpix_ (nside=32) sky maps, for each of the six filters, at 5 to 15 minute timesteps from 2022-09-01 to 2037-09-04 (excluding daytime). These healpix_ maps are created using skycalc_.
   * skycalc_ is a sky brightness model written for the exposure time calculator used by the European Southern Observatory’s (ESO) Very Large Telescope (VLT) at Cerro Paranal.
   * skycalc_ estimates the sky brightness in each Rubin Observatory filter by integrating the modeled SED over the corresponding instrument transmission curves.
   * The full set of healpix_ maps occupy 144 GiB of disk space.

#. Approximations of the sky brightness maps can be saved compactly using a Zernike transform.
   
   * Zernike polynomials form a set of othrogonal basis functions on the unit disk. See :numref:`fig-zernike-z`.
   * Low order terms of a Zerinke transform can approximate smoothly varying functions over the unit disk.
   * Residuals between the skycalc_ healpix maps and their approximation with a 6th order (27 term) Zernike approximation have a standard deviation of between 0.006 and 0.014 in all bands, much less than the standard deviation of the residuals between the skycalc_ model and the observed sky: typical uncertainty in the residuals is not significant compared to the uncertainty of :math:`\sim 0.2` in the model itself.
   * The residuals do not follow a normal distribution; the distribution has narrow tails extending from -0.27 magnitudes to 0.12 magintudes. See rb:numref:`fig-resid-full`.
   * The extreme tails in the residuals occurr near the moon, when the moon is itself at high airmass. See :numref:`fig-resid-worst`, :numref:`fig-moon-sep-hist`, and :numref:`fig-moon-alt-hist`.

#. The ``lsst.sims.sky_brightness_pre.zernike`` python module provides an interface for use of Zernike coefficients to store sky brightness.

   * This module uses coefficients stored in an ``hdf5`` file with a size of 294MiB (for coefficints up to 6th order).
   * This module interpolates Zernike coefficients for times not exactly covered by the original set of healpix maps.
   * This module provides an API similar to that of ``lsst.sims.sky_brightness_pre.SkyModelPre``, but does not include options no longer needed by current versions of the scheduler.

Background
==========

The Legacy Survey of Space and Time (LSST) is an imaging sky survey which will be performed using the Vera C. Rubin Observatory, currently under construction.
Exposure scheduling, the selection of which pointings on the sky to image at which times, will be automated by the Rubin Observatory automated scheduler.
This scheduler selects exposures using a feature-based Markov decision process: at the time an exposure (or set of exposures) needs to be scheduled, the scheduler calculates a set of values ("features") for each filter, over the visible sky.
Using a linear combination of these features, the scheduler then calculates a score (the "reward function") that represents how good it is to perform exposures in that band and that pointing and desired time, and selects exposures to maximize the reward.
The current scheduler uses a set of three features:

1. The slew time: the amount of time "lost" in overhead due to repointing the telescope.
2. Survey progress: the difference between the number of already completed exposures (at each pointing, in each band) and the number desired (for the current time).
3. The difference between the predicted point source :math:`5\sigma` limiting magnitude (at each pointing, in each band) and the :math:`5\sigma` in dark time on the meridian (for that pointing and band).

The third feature in this list is a measure of the expected data quality: the faintness of objects whose brightnesses can be measured to a reference precesion.
There are two dominant factors that influence the limiting magnitude: the point spread function (the sharpness of the image), and the noise in the image.
The dominant factor in the noise relevant to objects near the magnitude limit is from statistical uncertainty in the light that covers the same pixels as the image of the object, but is not from the object being measured.
For isolated point sources, this is from "sky brightness", diffuse light that covers the entire image.
Estimation of the sky brightness is therefore essenital for the predicting the limiting magnitude, which is required to calculate the reward function, and therefor for selection of exposures by the scheduler.

Sky brightness arises from a variety of sources, including:

* airglow, light emmitted by the Earth's upper atmosphere from a variety of causes, including recombination of atoms photoionized by the Sun;
* twilight, sunlight scattered by the Earth's atmosphere when the Sun is just under the horizon;
* starlight and moonlight scattered by the Earth's atmosphere;
* Zodiacal light, sunlight scattered by dust in the plane of our solar system; and
* light polution, light from terrestrial sources scattered by the Earth's atmosphere.

Krisciunas and Schaefer (1991) FIXME describe a simple model for estimating using a highly simplified model: airglow from a thin spherical shell in the atmosphere, and single scattering of moonlight in the atmosphere.
The Dark Energy Survey (DES) scheduler used a refinement of this model, plus a rough model for twilight, to estimate the sky brightness to between 0.2 and 0.3 :math:`\frac{\textrm{mag}}{\textrm{asec}^2}`, depending on the band, for dark and moony skies outside of twilight.
(Bluer bands had lower residuals.) 
ESO's skycalc_ software improves on this model in several ways, estimating the full spectral energy distribution of the sky brightness using physical models for atmospheric processes that result in airglaw, multiple scattering of starlight and moonlight, and a model for Zodiacal light.
These improvements result in a model that estimates the sky brightness with residuals of 0.2 :math:`\frac{\textrm{mag}}{\textrm{asec}^2}`  across all bands FIXME 2013A%26A...560A..91J.

:numref:`fig-healpix-map` shows examples of sky brightness maps calculated by skycalc_, for times when the moon is down (so the sky brightness is dominated by airglow), and when the moon is near full and above the horizon (so scattered moonlight is a major contributor to sky brightness). In both cases, the sky brightness varies smoothly.
The sharpent variation occurrs where the sky is brightest: near the moon, and just above the maximum airmass: locations on the sky the scheduler will avoid anyway.

.. _label: fig-healpix-map
.. figure:: /_static/healpix_map.png
   :name: fig-healpix-map

   Sky brightness maps of the sky brightness as stored in the cached healpix map files, generated using skycalc_.
   The color scales are in units of :math:`\frac{\textrm{mag}}{\textrm{asec}^2}`.
   The maps are in horizon coordinates: the center of each map is the zenith, and the radial coordinate the angle with zenith (with a maximum zenith distance of :math:`69^{\circ}`).

.. _skycalc: https://www.eso.org/sci/software/pipelines/skytools/skymodel

..
  ESO Skycalc references: https://www.eso.org/sci/software/pipelines/skytools/skymodel
  https://ui.adsabs.harvard.edu/abs/2012A%26A...543A..92N/abstract
  https://ui.adsabs.harvard.edu/abs/2013A%26A...560A..91J/abstract

   
The Current System
==================

Each time the scheduler selects an exposures (or set of exposures), it evaluates the reward function across the sky, sampled at (nside=32) Healpix_ healpixel locations (in equatorial coordinates) at the time for which exposures need to be chosen..
It therefore needs values for sky brightness estimates on these sample points, as a function of time.

It is impractical to use the skycalc_ software to calculate these values on demand.
Instead, the scheduling team pre-calculates the sky brightness at these sample points at a set of sample times (every 5 to 15 minutes for each night between 2022-09-01 and 2037-09-04, covering the full date range of LSST).
These data are saved in a set of files totalling 144 GiB.
When evaluating the reward function, the scheduler looks up the pre-computed sky brightness values near the desired times, and interpolates for the desired time.

.. _Healpix: https://healpix.jpl.nasa.gov/

The Rubin Observatory scheduler calls its sky brightness estimator by passing a time (as a floating point MJD) and set of healpix coordinates, which returns a dictionary of ``numpy`` arrays of sky brightnes values.
The keys of this dictionary are the filters, and the values are arrays that hold the sky brightnes values (corresponding to the array of indices provided).

>>> import numpy as np
>>> import healpy
>>> from lsst.sims.skybrightness_pre import SkyModelPre
>>>
>>> mjd = 59854.3
>>> npix = 32
>>>
>>> ra1, decl1 = 0, -50
>>> pointing1 = healpy.ang2pix(npix, ra1, decl1, lonlat=True)
>>>
>>> ra2, decl2 = 0, -20
>>> pointing2 = healpy.ang2pix(npix, ra2, decl2, lonlat=True)
>>>
>>> healpix_idxs = np.array((pointing1, pointing2))
>>>
>>> sky_model_pre = SkyModelPre()
>>> sky_brightness = sky_model_pre.returnMags(mjd, healpix_idxs)
>>>
>>> print("Sky brightness in i at pointing 1:",  sky_brightness['i'][0])
Sky brightness in i at pointing 1: 20.211941485692027
>>> print("Sky brightness in g at pointing 2:",  sky_brightness['g'][1])
Sky brightness in g at pointing 2: 21.908871333901892

If no healpix ids are provided in the call to ``returnMags``, then the array of sky values over the whole sphere is returned, and the healpix ids are the indices of the array.

Justification and scope of changes
==================================

The current method of storing the sky brightness values is inconveniently and unnecessarily large: a full ``nside=32``  healpix_ map (pre-computed for each time sample) stores the sky brightness for 12288 sample pointings, at high precision. The sky brightness, however, varies slowly as a function pointing for most locations on the sky, and the model is only good to a precision of 0.2 :math:`\frac{\textrm{mag}}{\textrm{asec}^2}`.

The scope of this proposal is limited to replacing the use of (lossless) storage of pre-computed sky values by the scheduler with a more compact approximation. It does not propose to change the underlying physical model used, nor calculation of sky brightness in any context outside of the scheduler itself.

The Proposed System
===================

Background: Zernike polynomials as basis functions
--------------------------------------------------

As approximations of smoothly varying functions on the unit disk that show significant radial symmetry, Zernike coefficients are a promising candidate.
Zernike polynomials form a set of orthogonal basis functions on the unit disk.
This use of Zernike polynomials is directly analogous to simple Fourier-transform based lossy image compression techniques, but is more naturally applied to the unit disk, and particulary suitable for functions with rotational symmetry. In the simplest applications, the transform can be truncated to include only lower order terms. Such a truncation has the effect of blurring the image. In more sophisticated applications, terms near zero can be set to zero and the result compressed. The same approaches can be applied using the Zernike transform as well. Because the "image" being compressed is smoothly varying, only a simple truncation is explored her ( although the more sophisticated approach may be useful).

There are several convertions for indexing and normalizing Zernike polynomials. Those used here are from FIXME Thebos 2002:

.. math::
   Z^{m}_n(\rho,\phi) = \begin{cases}
                                  N^m_n \times R^m_n(\rho) \times \cos(m \phi) & m \ge 0 \\
				  -N^m_n \times R^m_n(\rho) \times \sin(m \phi) & m \lt 0 \\
                           \end{cases}

where


.. math::
   N^m_n = \sqrt{\frac{2(n+1)}{1+\delta_{m0}}}

and

.. math::
   R^m_n(\rho) = \sum_{s=0}^{\frac{n-m}{2}} \frac{(-1)^k\,(n-s)!}{
   k!
   \left ( \frac{1}{2}[n + |m| - s] \right )!
   \left ( \frac{1}{2}[n - |m| - s] \right )!
   }
   \rho^{n-2s}

Here, :math:`\delta` is the Kroneker delta, :math:`m` is the angular frequency of the term, and :math:`n` the radial order. For a given radial order `n`, the angular frequency can have values :math:`-n, -n+2, -n+4, ..., n`. For the purposes of storing values and coefficients in a single dimensional array, it is convenient to define a single index, the mode number:

.. math::
   j = \frac{n(n+2) + m}{2}


Zernike coefficients that fit a function on the unit disk (the values of the Zernike transform of the function) are then:

.. math::
   F(\rho, \phi) = \sum_{m,n}\left[ a_{m,n}\ Z^{m}_n(\rho,\phi) + b_{m,n}\ Z^{-m}_n(\rho,\phi) \right]

:numref:`fig-zernike-z` shows :math:`Z^{m}_n(\rho,\phi)` graphically, and provides some intuition for the kinds of furctions low order Zernike coefficients can effectively represent.

.. _label: fig-zernike-z
.. figure:: /_static/basis7.png
   :name: fig-zernike-z

   The Zernike polyniomials, :math:`Z^{m}_n(\rho,\phi)`, for :math:`n<7`. The number to the upper left of each subplot shows the mode number, :math:`j` (the single-valued index).

Implementation
--------------

Computation from Zernike coefficients and polynomials
.....................................................

To compute estimates of the sky brightness, the ``lsst.sims.skybrightness_pre.zernike.ZernikeSky`` class evaluates

.. math::
   F(t, \rho, \phi) = \sum_j c[j](t)\ Z[j](\rho,\phi)

:math:`t` represents the time (stored as an MJD), and :math:`\rho, \phi` the coordinates in the disk over which the Zernike polynomials are orthogonal.
As used in the ``ZernikeSky`` class, :math:`\phi` is the azimuthal angle in horizon coordinates, and :math:`\rho = \frac{\mathrm{zd}}{\mathrm{zd_{max}}}`, where zd is the angular zenith distance, and zd :sub:`max` the maximum zenith distance for which surface brightness is to be calculated.
      
The mode number, :math:`j`, is used here rather that the more traditional radial order and angular frequency indices (:math:`n, m`) to simplify storage in a ``pandas.DataFrame`` or ``numpy`` array.
The sum contains two components: the coefficients, :math:`c[j](t)`, and the values of the Zernike polynomial themselves, :math:`Z[j](\rho,\phi)`.
The values of the coefficients for a given mode number is a function only of time, not location on the sky, while the values of the Zernike polynomials (for a given mode number) are a function only of the location on the sky (in horizon coordinates).

To implement the API shown above, the implementation of the Zernike-based sky brightness code requires values of :math:`c[j]` at the ``mjd`` requested, and :math:`Z[j](\rho,\phi)` for each healpix index requested.

Estimation of Zernike coefficients :math:`c[j](t)`
..................................................

The ``ZernikeSky.load_coeffs`` method reads values for the Zernike coefficients from an ``hdf5`` file into a ``pandas.DataFrame``, with columns for each Zernike mode number and rows for each time step fit, such that each row corresponds to a time at which the ``lsst.sims.skybrightness_pre.SkyModelPre`` stores a full healpix map.
Each (``nside=32``) healpix map contains 12288 values, while a row in the ``pandas.DataFrame`` of Zernike coefficients though a radial of 6 has 28 values, a factor of :math:`\sim 439` times more compact.


:numref:`fig-worst-coeff-vs-time` shows the variation in the values of the Zernike coeffients over time, over the course of a dynamic night.
At the end of evening twiling, the moon is below the horizon, but rises shortly thereafter.
The effect is particulary clean in the :math:`Z_0^0` term, in which the sky brightness drops sharply at the start of the night (evening twilight), briefly plateaus (the portion of the night during which the moon is below the horizon), the brightens as the moon rises.
Over the course of the night, the moon rises into the area covered by the sky approximation, transits, and begins to set.
It can be seen from the plots that the contribution of each Zernike polynomial drops as the radial order increases, with the lowest order terms having the most significant influence on the calculated sky brightness.
It can also be seen that, except at the precise time step at which the moon rises, the values of the coefficients vary smoothly with time relative to the sampling in time:  values of coefficients between points can be effectively estimated by simple linear interpolation.
This is what the ``lsst.sims.skybrightness_pre.zernike.ZernikeSky`` class does.

.. _label: fig-worst-coeff-vs-time
.. figure:: /_static/worst_coeff_vs_time.png
   :name: fig-worst-coeff-vs-time

   The values of the Zernike coefficients for a particularly dynamic night, in which the moon begins beneath the horizon and then rises and transits over the course of the night.
   The coefficients are scaled such that the values on the vertical axis represent the maximum change in magnitude contributed by each mode number.
   The heavy red points show the values for `i` band, while the fainter points in other colors represent other filters.

Estimation of Zernike polynomial values :math:`Z[j](\rho, \phi)`
................................................................
   
While the coefficients themselves are functions of the time and independent of the location on the sky, the values of the Zernike polynomials of a given mode number are themselves are functions only of the location on the sky (in horizon coordinates).
Each :math:`Z_m^n(\rho, \phi)` term is the product of a polynomial in :math:`\rho` and a trigonometric fuction of :math:`\phi`, making it the most computationally expensive requirements for calculating the :math:`F(t, \rho, \phi)`.
The values, however, of :math:`Z_m^n(\rho, \phi)` do not change with time.
If we are making repeated calculations at specific values of :math:`\rho, \phi`, these values can be computed once and cached.
To fulfill the API, ``ZernikeSky`` must provide sky brightness values at a predefined set of coordinates, suggesting that that we can simply calculate :math:`Z_m^n` at these values, but there is a problem: the healpix coordinates in the argument are predefined in equatorial coordinates, but the values of :math:`Z_m^n` are constant in horizon coordinates.
The values of :math:`\rho, \phi` for the given set of healpix values are a function of the local Sidereal time (LST), so the values of :math:`Z_m^n` are as well.
   
.. _label: fig-Z-vs-LST
.. figure:: /_static/Z_vs_LST.png
   :name: fig-Z-vs-LST

   The variation of :math:`Z_m^n` as a function of LST for several sample equatorial healpixels at different declinations.
   (Variation in R.A. shifts the values horizontally, but does not affect the shape of the curves.)
   Sidereal times for which the curves have no values shown are time at which these pointings are at a zenith distance greater than the range of the Zerinke function.
   The lowest order functions (shown in the upper rows) correspond to Zernikes that have the greatest influence on the ultimate sky brightness estimate (see corresponding plots in :numref:`fig-worst-coeff-vs-time`), have Zernike polynomials that vary most smoothly over the sky (:numref:`fig-zernike-z`), and have the smoothest behavior in this plot. 

Rather than calculate :math:`Z_m^n(\rho, \phi)` "from scratch" for each healpix at the time requested, ``ZernikeSky`` pre-computes ``Z[j][hpix]`` at a sequences of LST values for each Zernike mode number and equatorial healpixel, and interpolates to obtain the value of ``Z[j][hpix]`` at the LST corresponding to the MJD with which it is called.
The inset in the upper left of :numref:`fig-Z-vs-LST` shows sampled points for five different healpixels and different declinations. 

Computing sky brightness with ``ZernikeSky``
............................................

To calculate the sky brightness using an API that following that used by the scheduler:

>>> import numpy as np
>>> import healpy
>>> from lsst.sims.skybrightness_pre.zernike import SkyModelZernike
>>>
>>> mjd = 59854.3
>>> npix = 32
>>>
>>> ra1, decl1 = 0, -50
>>> pointing1 = healpy.ang2pix(npix, ra1, decl1, lonlat=True)
>>>
>>> ra2, decl2 = 0, -20
>>> pointing2 = healpy.ang2pix(npix, ra2, decl2, lonlat=True)
>>>
>>> healpix_idxs = np.array((pointing1, pointing2))
>>>
>>> sky_model = SkyModelZernike()
>>> sky_brightness = sky_model.returnMags(mjd, healpix_idxs)
>>>
>>> print("Sky brightness in i at pointing 1:",  sky_brightness['i'][0])
Sky brightness in i at pointing 1: 20.240004108584117
>>> print("Sky brightness in g at pointing 2:",  sky_brightness['g'][1])
Sky brightness in g at pointing 2: 21.967150461105156

This example relies on finding the ``hdf`` file with the Zernike coefficients in ``${SIMS_SKYBRIGHTNESS_DATA}/zernike/zernike.h5``.
If it is located elsewhere, the first argument in the instantiation of ``SkyModelZernike`` should be the path to this file.

Analysis
--------

To test the effectiveness of approximating skycalc_ sky brightness maps using truncated Zernike transforms, I fit Zernike coefficents through the 5th (21 terms), 6th (27 terms), and 11th orders (78 terms) orders to each of these sampled time steps.
Masked values in the healpix maps (around the zenith and moon) results in an unevently sampled starting data set, so a least squares fit was used to derive the coefficients rather than a traditional sum of products. 

:numref:`fig-resid-new` and :numref:`fig-resid-full` show typical skycalc_ maps, their 6th order (27 term) Zernike approximation, and residuals for dark (moon below the horizon) and bright (full moon above the horizon) sample times. The residuals show high-frequency patterns not representable by Zernike functions of this order; compare the lower left subplots of these figures with the basis functions in :numref:`fig-zernike-z`. 

.. _label: fig-resid-new
.. figure:: /_static/resid_new.png
   :name: fig-resid-new
	  
   The upper two pannels show the skycalc_ sky brightness for a typical fully dark (no moon) time (in horizon coordinates, with zenith at the center) on the left, and the Zernike approximation of these values on the night.
   The lower left figure shows the difference between the skycalc_ sky brightness and its Zernike approximation, and the lower right histogram shows the distribution of these residuals, masking the :math:`20^{\circ}` around the moon.
	  
.. _label: fig-resid-full
.. figure:: /_static/resid_full.png
   :name: fig-resid-full
	  
   The subplots above have the same meaning as those in :numref:`fig-resid-new`, except for a time with a full moon above the horizon.

The distribution is dominated by residuals less than 0.05 magnitudes in all cases, but thin tails extend from :math:`\sim -0.3` to :math:`\sim 0.1`. The quantiles and extreme of residuals are as follows:

========  ==========  =====  =====  =======  ======  =====  =====  =====  =======  ========  =====
filter    order         std    min    0.01%    0.1%     1%    50%    99%    99.9%    99.99%    max
========  ==========  =====  =====  =======  ======  =====  =====  =====  =======  ========  =====
u         5th order    0.02  -0.14    -0.12   -0.10  -0.04   0.00   0.05     0.06      0.08   0.10
g         5th order    0.02  -0.26    -0.16   -0.11  -0.05   0.00   0.05     0.08      0.12   0.17
r         5th order    0.02  -0.24    -0.15   -0.11  -0.05   0.00   0.05     0.08      0.10   0.14
i         5th order    0.02  -0.19    -0.13   -0.10  -0.04   0.00   0.05     0.07      0.09   0.13
z         5th order    0.01  -0.17    -0.12   -0.09  -0.04   0.00   0.05     0.06      0.08   0.12
y         5th order    0.01  -0.14    -0.12   -0.09  -0.03  -0.00   0.04     0.06      0.07   0.08
u         6th order    0.01  -0.11    -0.06   -0.05  -0.02   0.00   0.02     0.05      0.06   0.07
g         6th order    0.01  -0.23    -0.14   -0.08  -0.03  -0.00   0.04     0.06      0.09   0.12
r         6th order    0.01  -0.20    -0.13   -0.07  -0.03  -0.00   0.04     0.06      0.09   0.12
i         6th order    0.01  -0.17    -0.09   -0.05  -0.02   0.00   0.03     0.06      0.09   0.12
z         6th order    0.01  -0.15    -0.06   -0.04  -0.02  -0.00   0.03     0.05      0.07   0.11
y         6th order    0.01  -0.11    -0.05   -0.04  -0.02  -0.00   0.02     0.04      0.05   0.08
u         11th order   0.00  -0.17    -0.07   -0.03  -0.01   0.00   0.02     0.04      0.07   0.10
g         11th order   0.01  -0.16    -0.09   -0.04  -0.02   0.00   0.03     0.06      0.09   0.13
r         11th order   0.01  -0.15    -0.08   -0.04  -0.02   0.00   0.03     0.07      0.10   0.13
i         11th order   0.01  -0.13    -0.06   -0.03  -0.02   0.00   0.02     0.07      0.09   0.12
z         11th order   0.01  -0.11    -0.05   -0.03  -0.01   0.00   0.02     0.05      0.08   0.10
y         11th order   0.00  -0.08    -0.03   -0.02  -0.01   0.00   0.01     0.04      0.06   0.08
========  ==========  =====  =====  =======  ======  =====  =====  =====  =======  ========  =====


:numref:`fig-residual-hists` shows the histograms of the residuals for each order tested, in each filter, for the first year of tested data. The log scale accentuates the long tails of the distribution.
Recall that the standard deviation of the residuals of the skycalc_ model with respect to actual data is :math:`\sim 0.2`: instances where the the difference between the Zernike approximation and the skycalc_ value are comparable to the precision of the skycalc_ model are rare, but they exist.
   
.. _label: fig-residual-hists
.. figure:: /_static/residual_hists.png
   :name: fig-residual-hists

   Histograms of the skycalc_ - Zernike approximation residuals, on a log scale, for all filters and with Zernike approximations to 5th, 6th, as 11th order. Note the log scale covering 7 orders of magnitude: the disctribution is sharply peaked around 0.
	  
Examination of examples of time samples with bad residuals indicate conditions under which Zernike approximations perform most poorly.
:numref:`fig-resid-worst` shows the maps, residuals, and histogram of residuals for the worst timestep in the first year.
It occurs when the moon is at an altitude of :math:`\sim 20^{\circ}`, just outside the area covered by the map (which extends only to a zenith distance of :math:`67^{\circ}`).
The worst residuals occur at the same azimuth as the moon: just above the moon on the sky.
   
.. _label: fig-resid-worst
.. figure:: /_static/resid_worst.png
   :name: fig-resid-worst
	  
   The subplots above have the same meaning as those in :numref:`fig-resid-new`, except for the time with the worst residuals.

:numref:`fig-moon-sep-hist` and :numref:`fig-moon-alt-hist` indicate that this is typical of the worst time steps: they occurr when the moon has an altitude of :math:`\sim 20^{\circ}`, in sky within :math:`\sim 20^{\circ}` of the moon, and an altitude of less than :math:`\sim 40^{\circ}`.
	
.. _label: fig-moon-sep-hist
.. figure:: /_static/moon_sep_hist.png
   :name: fig-moon-sep-hist

   A 2-dimensional histogram of the sky estimates as a function of residual between skycalc_ magnitude and its Zernike approximation, and the angular separation between the point on the sky and the moon.
   The color is coded according to a log scale, covering 6 orders of magnitude. Note that the worst residuals are within :math:`20^{\circ}` of the moon.

.. _label: fig-moon-alt-hist
.. figure:: /_static/moon_alt_hist.png
   :name: fig-moon-alt-hist

   A 2-dimensional histogram of the sky estimates as a function of residual between skycalc_ magnitude and its Zernike approximation, and the altitude of the moon.. The color is coded according to a log scale, covering 6 orders of magnitude.
   Note that the worst residuals occur when the moon is at an altitude of about :math:`20^{\circ}`.

.. _label: fig-alt-hist
.. figure:: /_static/alt_hist.png
   :name: fig-alt-hist

   A 2-dimensional histogram of the sky estimates as a function of residual between skycalc_ magnitude and its Zernike approximation, and the altitude an the sky. The color is coded according to a log scale, covering 6 orders of magnitude.
   Note that the worst residuals occur at altitudes below :math:`40^{\circ}` (an airmass of about 1.6).

Although these histograms confirm that the very worst residuals occurr in situations similar to those shown in :numref:`fig-resid-worst`, they also show some residuals as bad as :math:`\sim 0.15` magnitudes occurr even in dark time. :numref:`fig-resid-worst-dark` shows the sample time with the worst residuals in dark time.
The "spike" of brightness at an azimuth of :math:`\sim90^{\circ}` is suggestive of zodiacal light, and indeed at this timestep is near twilight, with the sun azimuth near the area with high residuals (:math:`\mbox{az}=95^{\circ}`), as one would expect if this were zodiacal light.
   
.. _label: fig-resid-worst-dark
.. figure:: /_static/resid_worst_dark.png
   :name: fig-resid-worst-dark
	  
   The subplots above have the same meaning as those in :numref:`fig-resid-new`, except for the dark time (moon below the horizon) with the worst residuals.

If zodial light is the cause of all of the extreme dark time residuals, then one would expect these high residuals only to occurr at low ecliptic latitude, and indeed this is what :numref:`fig-dark-ecl-lat-hist` shows.

.. _label: fig-dark-ecl-lat-hist
.. figure:: /_static/dark_ecl_lat_hist.png
   :name: fig-dark-ecl-lat-hist

   A 2-dimensional histogram of the sky estimates as a function of residual between skycalc_ magnitude and its Zernike approximation, and the ecliptic latitude. The color is coded according to a log scale, covering 6 orders of magnitude.
   Note that the worst residuals occur at ecliptic latitudes close to 0.

Timing
======

The calculation of the Zernike sky requires computing the sums of Zernike coefficients, and therefore requires additional compute time over a simple lookup table.
Testing on a lightly loaded system shows the following timings for the lookup table and Zernike approximation computation of the full set of healpixel values at one MJD:

==========  =========
Method      Time (ms)
==========  =========
healpix     4.5
5th order   9.8
6th order   11.9
11th order  24.3
==========  =========


Conclusion and future work
==========================

Replacement of the healpix lookup table based storage of sky brightness values with one based on approximations using fit Zernike coefficients will have the following effects:

* The disk storage required for the sky brightness data will drop from 144GiB to 220MiB (for 5th order), 294MiB (for 6th order), or 811MiB (for 11th order).
* The values retured will not be exactly those calculated by skycalc_, but only approximations with residuals with standard deviations of 0.02 mag/asec^2 in all cases, and rare extreme deviations of up to 0.26, 0.23, or 0.17 mag/asec^2 for 5th, 6th, and 11th order Zernike polynomials, respectively. These extreme deviations occur near a bright moon, when the moon is at an altitude of :math:`\sim 20^{\circ}`. For comparison, the precision of the model is about 20 mag/asec^2.
* The time required for the schedule to obtain sky brightness values increases by 77%, 122%, and 448% for 5th, 6th, and 11th order Zernikes, respectively.

There are additional optimizations that can be made to reduce the disk space required. :numref:`fig-worst-coeff-vs-time` shows that the coefficients change only slowly with time relative to the current sampling: the coefficiently can potentially be stored much more sparsely with little loss of precision. Furnthermore, the limited precision of the model means that double precision data type with which the coefficients are stored may be excessive: the coefficients could potentially be stored as short floats without loss of effective precision.
  
.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
