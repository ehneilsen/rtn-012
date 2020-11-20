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

..
  The IEEE provides a template for a "Concept of Operations document", which
  covers a lot of the ground that should be covered by this note. So, I
  repeat the outline here to serve as a list of things I need to at least
  consider including.
  4.1 Scope (Clause 1 of the ConOps document)
      identification
      doc overview
      system overview
  4.2 Referenced documents (Clause 2 of the ConOps document)
  4.3 Current system or situation (Clause 3 of the ConOps document)
      background, objectives, and scope
      operational policies and constraints
      description of the current system
      modes of operation
      user classes
      support environment
  4.4 Justification for and nature of changes (Clause 4 of the ConOps document)
      justification of changes
      description of changes
      priorities among changes
      changes considered but not included
      assumptions and constraints
  4.5 Concepts for the proposed system (Clause 5 of the ConOps document)
      background, objectives, and scope
      operational policies and constraints
      description of the current system
      modes of operation
      user classes
      support environment
  4.6 Operational scenarios (Clause 6 of the ConOps document)
  4.7 Summary of impacts (Clause 7 of the ConOps document)
      operational impacts
      organization impacts
  4.8 Analysis of the proposed system (Clause 8 of the ConOps document)
      summary of improvements
      disadvantages and limitations
      alternatives and trade-offs

Overview
========

The Rubin Observatory scheduler uses the expected depth (limiting magnitude) of potential exposures as one of the fundamental features it uses to select among candidates.
One of the primary factors in calculating this depth is the sky brightness: diffuse light not orginating in any object the survey is interested in including in its catalog. 
The present (November, 2020) version of the scheduler estimates skybrightness values using the ESO skycalc_ sky brightness estimator. 
Computing values using the skycalc_ software is sufficiently cumbersome that, rather than calculate sky brightness estimates at the time they are needed, the scheduler development team pre-calculates and caches all sky values that might be needed over the course of the survey.
These pre-computed values are currently stored in Healpix_ maps with nside=32, sampled every 5 minutes.
This format results in a set of files that are inconveniently large: 152GiB all told.
Sky brightness typically varies smoothly, both over the sky and over time.
Rather than store values for full healpix maps, significant storage space can be saved by saving the coefficients for a Zernike polynomial tha approximates the sky values to be cached.
After giving a brief overview of the sky brightness estimate itself, this note describes an implementation of such an approximation, and examines its implications for disk space required, time taken to provide the scheduler sky values, and precision with whith the skycalc_ values are approximated. 

Sky value estimation
====================

The "sky brightness" consists of diffuse light from (mostly) foreground sources, spread smoothly across the detector.
The primary contributing sources include airglow, moon and starlight scattered off of particles in the Earth's atmospher, and Zodiacal light (sunlight scattered off of interplanetary dust in the disk of the solar system).
ESO's skycalc_ software estimates the full spectral energy distribution of the sky brightness using physical models, including light singly and multiply scattered though Mie and Rayleigh scattering.

To prepare the arrays of pre-computed sky brightness values, the scheduling team integrates skycalc-generated spectra over each of the Rubin Observatory bandpasses, for each (nside=32) healpixel, in each 5 minute interval in survey schedule.
The resultant data-set is divided by date into a set of files totaling 152GiB. 
When a sky brightness value (or set of values) is needed by the scheduler, it reads the file for the relevant date range from disk (if it is not already loaded) and looks up the value (or values).

:numref:`fig-healpix-map` shows extreme examples of such sky brightness maps.
The left map shows a full dark sky (no moon or twilight). The dominant source of light in such a dark sky is airglow, which is dependent only on airmass.
The map therefore shows near radial symmetry about the zenith. The other extreme is sky with a full moon at moderate airmass.
Some radial dependence remains, but it is dominated by a gradient across the sky (in the sample in the figure, from left to right), and high brightnees about the moon itself.

.. _label: fig-healpix-map
.. figure:: /_static/healpix_map.png
   :name: fig-healpix-map

   Sky brightness maps of the sky brightness as stored in the cached healpix map files, generated using skycalc_.
   The color scales are in units of :math:`\frac{\textrm{mag}}{\textrm{asec}^2}`.
   The maps are in horizon coordinates: the center of each map is the zenith, and the radial coordinate the angle with zenith (with a maximum zenith distance of :math:`69^{\circ}`).

In both cases, the sky brightness varies smoothly.
The sharpent variation occurrs where the sky is brightest: near the moon, and just above the maximum airmass: locations on the sky the scheduler will avoid anyway.
Furthermore, tests of the skycalc_ model reported in FIXME 2013A%26A...560A..91J show a standard deviation between the model and measured values of 0.2 :math:`\frac{\textrm{mag}}{\textrm{asec}^2}` or greater, indicating that there is little to be gained from storing model values to high precision. These considerations suggest that the full, high-resolution healpix map is unnecessary, and that a smooth approximation will be adequate.

.. _skycalc: https://www.eso.org/sci/software/pipelines/skytools/skymodel
.. _Healpix: https://healpix.jpl.nasa.gov/

..
  ESO Skycalc references: https://www.eso.org/sci/software/pipelines/skytools/skymodel
  https://ui.adsabs.harvard.edu/abs/2012A%26A...543A..92N/abstract
  https://ui.adsabs.harvard.edu/abs/2013A%26A...560A..91J/abstract

Zernike polynomials
===================

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

Here, :math:`m` is the angular frequency of the term, and :math:`n` the radial order. For the purposes of storing values and coefficients in a single dimensional array, it is convenient to define a single index, the mode number:

.. math::
   j = \frac{n(n+2) + m}{2}


Zernike coefficients that fit a function on the unit disk (the values of the Zernike transform of the function) are then:

.. math::
   \begin{align*}
   F(\rho, \phi) & = \sum_{m,n}\left[ a_{m,n}\ Z^{m}_n(\rho,\phi) + b_{m,n}\ Z^{-m}_n(\rho,\phi) \right] \\
                & = \sum_j c[j]\ Z[j](\rho,\phi)
   \end{align*}

:numref:`fig-zernike-z` shows :math:`Z^{m}_n(\rho,\phi)` graphically, and provides some intuition for the kinds of furctions low order Zernike coefficients can effectively represent.

.. _label: fig-zernike-z
.. figure:: /_static/basis7.png
   :name: fig-zernike-z

   The Zernike polyniomials, :math:`Z^{m}_n(\rho,\phi)`, for :math:`n<7`. The number to the upper left of each subplot shows the mode number, :math:`j` (the single-valued index).


Zernike approximations of sky brightness
========================================

The Rubin Observatory team produced a collection of nside=32 healpix maps at 5 minute incruments from MJD 59823 (2022-09-01) through MJD 65305 (2037-09-04).
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

:numref:`fig-residual-hists` shows the histograms of the residualsfor each order tested, in each filter, for the first year of tested data.
The distribution is dominated by residuals less than 0.05 magnitudes in all cases, but thin tails extend from :math:`\sim -0.3` to :math:`\sim 0.1`, accentuated by the log scale in the figure.
Recall that the standard deviation of the residuals of the skycalc_ model with respect to actual data is :math:`\sim 0.2`: instances where the the difference between the Zernike approximation and the skycalc_ value are comparable to the precision of the skycalc_ model are rare, but they exist.
   
.. _label: fig-residual-hists
.. figure:: /_static/residual_hists.png
   :name: fig-residual_hists

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

   A 2-dimensional histogram of the sky estimates as a function of residual between skycalc_ magnitude and its Zernike approximation, and the angular separation between the point on the sky and the moon. The color is coded according to a log scale, covering 6 orders of magnitude. Note that the worst residuals are within :math:`20^{\circ}` of the moon.

.. _label: fig-moon-alt-hist
.. figure:: /_static/moon_alt_hist.png
   :name: fig-moon-alt-hist

   A 2-dimensional histogram of the sky estimates as a function of residual between skycalc_ magnitude and its Zernike approximation, and the altitude of the moon.. The color is coded according to a log scale, covering 6 orders of magnitude. Note that the worst residuals occur when the moon is at an altitude of about :math:`20^{\circ}`.

.. _label: fig-alt-hist
.. figure:: /_static/alt_hist.png
   :name: fig-alt-hist

   A 2-dimensional histogram of the sky estimates as a function of residual between skycalc_ magnitude and its Zernike approximation, and the altitude an the sky. The color is coded according to a log scale, covering 6 orders of magnitude. Note that the worst residuals occur at altitudes below :math:`40^{\circ}` (an airmass of about 1.6).

Although these histograms confirm that the very worst residuals occurr in situations similar to those shown in :numref:`fig-resid-worst`, they also show some residuals as bad as :math:`\sim 0.15` magnitudes occurr even in dark time.
   
.. _label: fig-resid-worst-dark
.. figure:: /_static/resid_worst_dark.png
   :name: fig-resid-worst-dark
	  
   The subplots above have the same meaning as those in :numref:`fig-resid-new`, except for the dark time (moon below the horizon) with the worst residuals.


   
Calculation and storage of Zernike coefficients
===============================================


Conclusion
==========

	  
.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
