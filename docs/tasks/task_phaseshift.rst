

.. _Description:

Description
   This task changes the phase center of an MS by modifying the *UVW*
   coordinates and the specified data column(s) (via the **datacolumn**
   parameter) of the input MS and creating an output MS with these changes.
   The *PHASE_DIR* column of the *FIELD* subtable of the new MS is updated
   with the new phase center. Many MS selection parameters are supported (see
   `Visibility Data Selections
   <../../notebooks/visibility_data_selection.ipynb>`__
   for details). The input MS is not modified.

   The new phase center is specified via the **phasecenter** parameter.
   The standard syntax for specifying astronomical world direction coordinates
   is supported (*e.g.* 'J2000 19h45m20.56 -50d30m45.7' or
   'J2000 19:45:20.56 -50.30.45.7'). Coordinate systems that are time
   dependent are not supported, such as topocentric or geodetic systems
   (*e.g.* azimuth-elevation). Ephemeris objects are likewise not supported.
   An example of verification and usage can be found `here
   <https://docs.google.com/document/d/1wZhjizgHoTtI3_tdg6fqB5E8FTbwygViC2TSNGiFl7c>`__
   
   The **phaseshift** application uses the same algorithm as **tclean** (via its 
   **phasecenter** parameter) for phase center shifting. However, these two
   applications use a different algorithm than **plotms** does for phaseshifting,
   so results, particularly for larger shifts, are likely to diverge for
   **plotms** and **phaseshift**/**tclean**.
   
 
.. _Examples:

Examples
   ::
   
      # shift the phase center
      phaseshift(
          vis='unshifted.ms', outvis='shifted.ms',
          phasecenter='J2000 04:52:16 -02.04.55'
      )

.. _Development:

Development
   * Support for time-dependent coordinate frames and ephemeris objects
     is planned for ngCASA only. However, if greatly desired, requests for
     such support will be considered prior to that. Please send us a request
     via the Help Desk should you have such a need.
   * Specifying the new phase center in terms of an offset from
     the original phase center is currently not supported. However, if
     there is a need for such support, it can be added. Please send us a request
     via the Help Desk should you have such a need.
   * There is currently no support for the possible use case of updating only
     the *UVW* values but not the associated data values. This support can
     be added if there is a need for this mode of operation. Please send us a
     request via the Help Desk should you have such a need.

.. raw:: html
   :file: html/test.html