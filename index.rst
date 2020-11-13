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
  4.9 Notes (Clause 9 on the ConOps document)
  4.10 Appendices (Appendices of the ConOps document)
  4.11 Glossary (Glossary of the ConOps document)

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




.. _skycalc: https://www.eso.org/sci/software/pipelines/skytools/skymodel
.. _Healpix: https://healpix.jpl.nasa.gov/

..
  ESO Skycalc references: https://www.eso.org/sci/software/pipelines/skytools/skymodel
  https://ui.adsabs.harvard.edu/abs/2012A%26A...543A..92N/abstract
  https://ui.adsabs.harvard.edu/abs/2013A%26A...560A..91J/abstract

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
