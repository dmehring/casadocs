.. include:: <isogrk1.txt>

.. _Description:

Description
   This task changes the phase center of an MS by modifying the *UVW*
   coordinates and the specified data column(s) (via the **datacolumn**
   parameter) of the input MS and creating an output MS with these changes.
   The *PHASE_DIR* column of the *FIELD* subtable of the new MS is updated
   with the new phase center. Many MS selection parameters are supported (see
   `Visibility Data Selections
   <../../notebooks/visibility_data_selection.ipynb>`__
   for details). The input MS is not modified. The implementation assumes
   that the *UVW* coordinates are correct in the frame in which they are
   specified; these coordinates are transformed via rotation to the new
   phase center. No attempt is made to recompute the *UVW* values because of,
   eg, antenna position changes (please see the `Development`_ section for more
   details),

   The new phase center is specified via the **phasecenter** parameter.
   The standard syntax for specifying astronomical world direction coordinates
   is supported (*e.g.* 'J2000 19h45m20.56 -50d30m45.7' or
   'J2000 19:45:20.56 -50.30.45.7'). Coordinate systems that are time
   dependent are not supported, such as topocentric or geodetic systems
   (*e.g.* azimuth-elevation). Ephemeris objects are likewise not supported.
   An example of verification and usage can be found `here
   <https://docs.google.com/document/d/1wZhjizgHoTtI3_tdg6fqB5E8FTbwygViC2TSNGiFl7c>`__
   
   The **phaseshift** application uses a similar algorithm as **tclean** (via its 
   **phasecenter** parameter) for phase center shifting. However, these two
   applications use a signficantly different algorithm than **plotms** does for phase
   shifting, so results, particularly for larger shifts, are likely to diverge for
   **plotms** and **phaseshift**/**tclean**.
   
.. _Examples:

Examples
   ::
   
      # shift the phase center to J2000 04:52:16 -02.04.55
      # note that any valid direction syntax is supported, including
      # FRAME XXhXXmXX.Xs YYdYYmYY.Ys
      # FRAME XX.XXdeg YY.YYdeg
      # FRAME XX.XXrad YY.YYrad
      # the longitude-like and latitude-like coordinates can have different syntaxes, existing
      FRAME XXhXXmXX.Xs YY.YYrad
      phaseshift(
          vis='unshifted.ms', outvis='shifted.ms',
          phasecenter='J2000 04:52:16 -02.04.55'
      )

.. _Verfication:
Verification
    
    The following script can be used to verify that the behavior of phaseshift is as expected.
    The details are described following the script.

    .. code-block:: python

        # much of this code is adapted from that of rurvashi
        # https://gitlab.nrao.edu/rurvashi/simulation-in-casa-6/-/blob/master/Simulation_Script_Demo.ipynb

        from casatools import componentlist, ctsys, measures, quanta, simulator
        from casatasks import flagdata, imfit, imstat, phaseshift, tclean
        from casatasks.private import simutil
        import glob
        import os
        import shutil

        cl = componentlist()
        me = measures()
        qa = quanta()
        sm = simulator()

        comp_list = 'sim_onepoint.cl'
        orig_ms = 'sim_data.ms'
        orig_im = 'im_orig'
        pshift_ms = 'sim_data_pshift.ms'
        pshift_im = 'im_post_phaseshift'
        tshift_im = 'im_post_tclean_shift'
        exp_flux = 5

        def cleanup():
            for f in (comp_list, orig_ms, pshift_ms):
                if os.path.exists(f):
                    shutil.rmtree(f)
            for x in (orig_ms, orig_im, pshift_im, tshift_im):
                for path in glob.glob(x + '.*'):
                    shutil.rmtree(path)

        def __direction_string(ra, dec, frame):
            return ' '.join([frame, ra, dec])

        def __makeMSFrame(antenna_file, spwname, freq, radir, decdir, dirframe):
            # sphinx-build objects to triple quote comments...

            # Construct an empty Measurement Set that has the desired observation
            # setup.

            # Open the simulator
            sm.open(ms=orig_ms)

            # Read/create an antenna configuration.
            # Canned antenna config text files are located at
            # /home/casa/data/trunk/alma/simmos/*cfg
            # antennalist = os.path.join(ctsys.resolve("alma/simmos"), "vla.d.cfg")
            # Fictitious telescopes can be simulated by specifying x, y, z, d,
            # an, telname, antpos.
            # x,y,z are locations in meters in ITRF (Earth centered)
            # coordinates.
            # d, an are lists of antenna diameter and name.
            # telname and obspos are the name and coordinates of the
            # observatory.
            (x, y, z, d, an, an2, telname, obspos) = (
                simutil.simutil().readantenna(antenna_file)
            )
            # Set the antenna configuration
            sm.setconfig(
                telescopename=telname, x=x, y=y, z=z, dishdiameter=d, mount=['alt-az'],
                antname=an, coordsystem='global', referencelocation=me.observatory(telname)
            )
            # Set the polarization mode (this goes to the FEED subtable)
            sm.setfeed(mode='perfect R L', pol=[''])
            # Set the spectral window and polarization (one
            # data-description-id).
            # Call multiple times with different names for multiple SPWs or
            # pol setups.
            sm.setspwindow(
                spwname=spwname, freq=freq, deltafreq='0.1GHz',
                freqresolution='0.2GHz', nchannels=1, stokes='RR LL'
            )

            # Setup source/field information (i.e. where the observation phase
            # center is) Call multiple times for different pointings or source
            # locations.
            sm.setfield(
                sourcename="fake", sourcedirection=me.direction(
                    rf=dirframe, v0=radir, v1=decdir
                )
            )

            # Set shadow/elevation limits (if you care). These set flags.
            sm.setlimits(shadowlimit=0.01, elevationlimit='1deg')

            # Leave autocorrelations out of the MS.
            sm.setauto(autocorrwt=0.0)

            # Set the integration time, and the convention to use for timerange
            # specification
            # Note : It is convenient to pick the hourangle mode as all times
            #   specified in sm.observe() will be relative to when the source
            #   transits.
            sm.settimes(
                integrationtime='60s', usehourangle=True,
                referencetime=me.epoch('UTC', '2019/10/4/00:00:00')
            )

            # Construct MS metadata and UVW values for one scan and ddid
            # Call multiple times for multiple scans.
            # Call this with different sourcenames (fields) and spw/pol
            # settings as defined above.
            # Timesteps will be defined in intervals of 'integrationtime',
            # between starttime and stoptime.
            sm.observe(
                sourcename="fake", spwname=spwname, starttime='-5.0h',
                stoptime='+5.0h'
            )
            # Close the simulator
            sm.close()
            # Unflag everything (unless you care about elevation/shadow flags)
            flagdata(vis=orig_ms, mode='unflag')

        def __makeCompList(ra, dec, frame):
            # make a componentlist of point sources
            
            # Add sources, one at a time.
            # Call multiple times to add multiple sources.
            # ( Change the 'dir', obviously )
            cl.addcomponent(
                dir=__direction_string(ra, dec, frame),
                flux=exp_flux,
                fluxunit='Jy', freq='1.5GHz', shape='point',
                spectrumtype="constant"
            )
            # Save the file
            cl.rename(filename=comp_list)
            cl.done()

        def __predictSimFromComplist():
            sm.openfromms(orig_ms)
            # Predict from a component list
            sm.predict(complist=comp_list, incremental=False)
            sm.close()

        def __summarize(imagename, imfit_box, sim_source_dir, label, prec):
            # get the image statistics, and print the world coordinates of the maxposf,
            # and note that they are within one 8" pixel of the simulated source position,
            # as expected.
            x = imstat(imagename)
            # to be even more accurate, fit a 2-D gaussian to the source to show that, to
            # approximately within the fit errors, the position is coincident to the
            # expected position
            y = imfit(imagename, box=imfit_box)
            poserr = y['deconvolved']['component0']['shape']['direction']['error']
            print(label)
            #    print('    Simulated source position',__direction_string(radir, decdir, dirframe)) 
            print('    Simulated source position', sim_source_dir)
            print("    coordinates of max position from imstat", x['maxposf'])
            cl.fromrecord(y['deconvolved'])
            rd = cl.getrefdir(0)
            cl.done()
            ra_err = qa.div(
                        qa.div(qa.quantity(poserr['longitude']), 15), 
                        qa.cos(qa.quantity(rd['m1']))
                    )
            ra_err['unit'] = 's'
            dec_err = qa.quantity(poserr['latitude'])
         
            print(
                "    fitted position from imfit",
                qa.time(qa.totime(qa.quantity(rd['m0'])), prec=6+prec, form='hms')[0], '\u00b1',
                qa.tos(ra_err, prec=prec),
                qa.angle(qa.totime(qa.quantity(rd['m1'])), prec=6+prec)[0], '\u00b1',
                qa.tos(dec_err, prec=prec),
            )

        def verify():
            def __create_input_ms():
                # create the input MS
                __makeMSFrame(antenna_file, spwname, freq, fra, fdec, fframe)
                # Make the component list
                __makeCompList(radir, decdir, dirframe)
                # Predict Visibilities
                __predictSimFromComplist()

            for observatory in ('VLA', 'ALMA'):
                cleanup()
                print(observatory, 'simulation:')
                # This is the source position
                if observatory == 'VLA':
                    radir = '19h49m43'
                    decdir = '38d45m15'
                    dirframe = 'J2000'
                    ant_cfg = "vla.d.cfg"
                    spwname = 'LBand'
                    freq = '1.0GHz'
                    cell = '8.0arcsec'
                    imfit_box = '1870, 165, 1890, 185'
                    prec = 3
                elif observatory == 'ALMA':
                    radir = '19h59m33.2'
                    decdir = '40d40m53.2'
                    dirframe = 'J2000'
                    antenna_file = ant_cfg = 'alma.cycle8.8.cfg'
                    spwname = 'Band4'
                    freq = '150GHz'
                    cell = '0.06arcsec'
                    imfit_box = '123, 1876, 143, 1896'
                    prec = 5
                antenna_file = os.path.join(ctsys.resolve("alma/simmos"), ant_cfg)
                dirstring = __direction_string(radir, decdir, dirframe) 
                # this is the original phase center
                fra = '19h59m28.5'
                fdec = '+40.40.01.5'
                fframe = 'J2000'
                __create_input_ms()
                # image simulated MS with no phase shift. The source is offset from the phase
                # center of the image. We use wproject and wprojplanes to correctly account
                # for the non-negligible values of the w coordinate because the source
                # is far from the phase center
                tclean(
                    vis=orig_ms, imagename=orig_im, datacolumn='data',
                    imsize=2048, cell=cell, gridder='wproject',
                    niter=20, gain=0.2, wprojplanes=128, pblimit=-0.1
                )
                __summarize(
                    orig_im + '.image', imfit_box, dirstring, 'Image with no shift applied', 5
                )
                # Now use phaseshift to shift the phase center of the MS to the source position
                phaseshift(vis=orig_ms, outputvis=pshift_ms, phasecenter=dirstring)
                # image the phase shifted MS. The image can be significantly smaller because the
                # source will now be at the image center. 
                tclean(
                    vis=pshift_ms, imagename=pshift_im, datacolumn='data',
                    imsize=256, cell=cell, gridder='wproject',
                    niter=20, gain=0.2, wprojplanes=128, pblimit=-0.1
                )
                __summarize(
                    pshift_im + '.image', '118, 118, 138, 138', dirstring,
                    'Phase shifted image using phaseshift to set the phase center', prec + 3
                )

                # Now image the original, unshifted MS using a phase shift in tclean
                tclean(
                    vis=orig_ms, imagename=tshift_im, datacolumn='data',
                    imsize=256, cell=cell, gridder='wproject',
                    niter=20, gain=0.2, wprojplanes=128, pblimit=-0.1,
                    phasecenter=__direction_string(radir, decdir, dirframe)
                )
                __summarize(
                    tshift_im + '.image', '118, 118, 138, 138', dirstring,
                    'Phase shifted image using tclean to set phase center', prec + 3
                )
                print()

        if __name__ == '__main__':
            verify()

    .. Overview_:
    
    Overview
    
        The script performs two simulations, the first based on the VLA and the second based on ALMA.
        For each, an MS is created using the simulator tool. In order to make the verification process
        more transparent, no noise is introduced into the simulated data. The visibilities are then
        predicted using a 5 Jy point source that has a significant offset (so that the *W* coordinate
        is non-negligible). The MS is imaged by tclean using no phase center shift, to illustrate that
        the source is indeed significantly offset from the phase center. Because the *w* coordinate
        must be properly accounted, *gridder=***'wproject'** and *wprojectplanes* are set in tclean.
        The peak pixel coordinates are computed via the task *imstat* and a two dimensional Gaussian
        fit using the task *imfit* is done to further constrain the source position to illustrate
        that it is indeed located at the coordinates specified when creating the MS. The *phaseshift*
        task is then run on the original MS, shifting the phase center to the position of the source.
        The resulting MS is imaged using tclean. Both *imstat* and *imfit* are run as before to verify
        the source coordinates. Finally, *tclean* is run using the original MS and specifying the
        *phasecenter* parameter to be the source position, to illustrate that *tclean* can also
        properly shift the phase center. Results, described in the next sections, indicate the both
        *phaseshift* and *tclean* properly shift the phase center to within a small fraction of a
        pixel, and that *phaseshift* is slightly more accurate, although both get very close to the
        expected result.

        The output of the script, when run on an RHEL 7 machine, is

        ::

            VLA simulation:
            Image with no shift applied
                Simulated source position J2000 19h49m43 38d45m15
                coordinates of max position from imstat 19:49:42.907, +38.45.13.185, I, 999980842.28Hz
                fitted position from imfit 19:49:42.99118 ± 0.01171s +038.45.15.21608 ± 0.10346arcsec
            Phase shifted image using phaseshift to set the phase center
                Simulated source position J2000 19h49m43 38d45m15
                coordinates of max position from imstat 19:49:43.000, +38.45.15.000, I, 999983449.88Hz
                fitted position from imfit 19:49:43.000008 ± 0.001796s +038.45.14.999908 ± 0.016103arcsec
            Phase shifted image using tclean to set phase center
                Simulated source position J2000 19h49m43 38d45m15
                coordinates of max position from imstat 19:49:43.000, +38.45.15.000, I, 999980842.28Hz
                fitted position from imfit 19:49:42.989652 ± 0.001851s +038.45.14.869487 ± 0.016600arcsec

            ALMA simulation:
            Image with no shift applied
                Simulated source position J2000 19h59m33.2 40d40m53.2
                coordinates of max position from imstat 19:59:33.200, +40.40.53.214, I, 1.49997e+11Hz
                fitted position from imfit 19:59:33.20002 ± 0.00002s +040.40.53.20107 ± 0.00038arcsec
            Phase shifted image using phaseshift to set the phase center
                Simulated source position J2000 19h59m33.2 40d40m53.2
                coordinates of max position from imstat 19:59:33.200, +40.40.53.200, I, 1.49997e+11Hz
                fitted position from imfit 19:59:33.20000000 ± 0.00000442s +040.40.53.19999711 ± 0.00007270arcsec
            Phase shifted image using tclean to set phase center
                Simulated source position J2000 19h59m33.2 40d40m53.2
                coordinates of max position from imstat 19:59:33.200, +40.40.53.200, I, 1.49997e+11Hz
                fitted position from imfit 19:59:33.20007214 ± 0.00000593s +040.40.53.20079661 ± 0.00009744arcsec

        .. VLA_:

    VLA Simulation

        The VLA simulation uses antenna positions in the D configuration and a frequency of 1.0 GHz.
        The resulting images have 8.0" pixels. The phase center and the source in the original MS
        were separated by about 2.7 degrees. Figure 1a shows the full image created from the
        original MS. The point source can be seen in the lower right corner of the image. Figure 1b 
        shows the image created from the MS after running **phaseshift** to shift the phase center
        to the position of the source. The contours represent the image created from the original MS,
        and using **tclean** to shift the phase center to the source position using the *phasecenter*
        parameter. Figure 1c is the central portion of Figure 1b.

        The results (see above) indicate that source position in the image that is not phase shifted
        is indeed as expected. The somewhat large fit errors are likely due to the fact that the
        source is not located at the center of a pixel. The results for the image created from
        the MS that has been produced by **phaseshift** show excellent agreement with the
        expected source position. The fit uncertainty provides an upper limit of about 30 m"
        (0.003 pixels) of the offset of the source from the phase center, and the best fit results
        are two orders of magnitude better than that at about 0.1 m" (0.00002 pixels). The results
        of the image created from the unshifted MS by applying the phase shift in **tclean** using
        the *phasecenter* parameter are also very good, although not as good as the image made from
        the MS created using **phaseshift**.  In this case, the offset between source and
        phase center falls significantly outside the fit uncertainty, with a separation of about
        0.2" (0.02 pixels).

        .. figure:: _apimedia/VLA_orig.png
            :alt: VLA simulated data image prior to phase shift. The source is in the lower right corner.

            Figure 1a. VLA simulated data image prior to phase shift. The source is in the lower
            right corner.

        .. figure:: _apimedia/VLA_shifted_full.png
            :alt: VLA simulated data full image after running phaseshift.

            Figure 1b. VLA simulated data full image after running phaseshift. The contours represent
            the image created by setting the phasecenter parameter in tclean.

        .. figure:: _apimedia/VLA_shifted_zoomed.png
            :alt: VLA simulated data image central portion after running phaseshift.

            Figure 1c. VLA simulated data image central portion after running phaseshift. The
            contours represent the image created by setting the phasecenter parameter in tclean.

    .. ALMA_:

    ALMA Simulation

        The ALMA simulation uses antenna positions in the eighth configuration of cycle 8 and a
        frequency of 150 GHz. The resulting images have 60 m" pixels. The phase center and the
        source in the original MS were separated by about 1.2'. Figure 2a shows the full image
        created from the original MS. The point source can be seen in the upper left corner of
        the image. Figure 2b shows the image created from the MS after running **phaseshift**
        to shift the phase center to the position of the source. The contours represent the
        image created from the original MS, and using **tclean** to shift the phase center to
        the source position using the *phasecenter* parameter. Figure 2c is the central portion
        of Figure 2b.

        The results (see above) indicate that source position in the image that is not phase shifted
        is indeed as expected. The somewhat large fit errors are likely due to the fact that the
        source is not located at the center of a pixel. The results for the image created from
        the MS that has been produced by **phaseshift** show excellent agreement with the
        expected source position. The fit uncertainty provides an upper limit of about 90 |mgr|"
        (0.001 pixels) of the offset of the source from the phase center, and the best fit results
        are an order of magnitude better than that at about 3 |mgr|" (0.00005 pixels). The results
        of the image created from the unshifted MS by applying the phase shift in **tclean** using
        the *phasecenter* parameter are also very good, although not as good as the image made from
        the MS created using **phaseshift**.  In this case, the offset between source and
        phase center falls significantly outside the fit uncertainty, with a separation of about
        1 m" (0.02 pixels).

        .. figure:: _apimedia/ALMA_orig.png
            :alt: ALMA simulated data image prior to phase shift. The source is in the upper left corner.

            ALMA simulated data image prior to phase shift. The source is in the upper right corner.

        .. figure:: _apimedia/ALMA_shifted_full.png
            :alt: ALMA simulated data full image after running phaseshift.

            ALMA simulated data full image after running phaseshift. The contours represent the image
            created by setting the phasecenter parameter in tclean.


        .. figure:: _apimedia/ALMA_shifted_zoomed.png
           :alt: ALMA simulated data image central portion after running phaseshift.

           ALMA simulated data image central portion after running phaseshift. The contours represent
           the image created by setting the phasecenter parameter in tclean.

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
     the *UVW* values (eg, based on antenna position updates), but not the associated
     data values. The deprecated task **fixvis** has this functionality, so it may
     be used for this purpose. If such support will be needed after **fixvis** is
     removed, it can be added. Please send us a request via the Help Desk should you
     have such a need.

 

