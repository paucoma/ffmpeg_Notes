
Extract an image from video frames
==================================
As stated in the `ffmpeg documentation <https://ffmpeg.org/ffmpeg.html#Synopsis>`_ : ffmpeg command line options are specified as

ffmpeg [global_options] {[input_options] -i input_file} ... {[output_options] output_file} ...

:code:`ffmpeg -i <input> -f image2 -update 1 img.jpg`

-------
Details
-------

Filename generation
--------------------

  :code:`-f <fmt>` (input/output)

    Force input or output file format. The format is normally auto detected for input files and guessed from the file extension for output files, so this option is not needed in most cases.

    *<fmt>* :

      - **image2** : `image2 is an image file muxer <https://www.ffmpeg.org/ffmpeg-formats.html#image2>`_, writes video frames to image files.
        + :code:`start_number <n>` : Start the sequence from <n>. Default : 1.
        + :code:`update 1` : if update == 1 filename interpreted as literal text. No patterns interpreted, Default : 0
        + :code:`strftime 1` : define the filename with date and time information , `strftime() <http://www.cplusplus.com/reference/ctime/strftime/>`_ , Default 0.
          * Once you have **strftime** activated the sequential pattern %d or %0Nd will not be interpreted, the latter will actually give an error.

      - With ffmpeg, if the format is not specified with the -f option and the output is an image file format, the image2 muxer is automatically selected.
      - Filename interpreted formats:
        + pattern may contains :code:`"%d"` or :code:`"%0Nd"` where N defines the number of *zero-padded* digits.
        + if *strftime* is activated patterns such as :code:`%Y-%m-%d_%H-%M-%S` will be interpreted.


Rate of image image extraction
-------------------------------

  :code:`-r[:stream_specifier] <fps>` (input/output,per-stream)

    Set frame rate :code:`fps` (Hz value, fraction or abbreviation).

  As an input option,
     - it ignores any timestamps stored in the file and instead generate timestamps assuming constant frame rate fps.
     - This is not the same as the -framerate option used for some input formats like image2 or v4l2 (it used to be the same in older versions of FFmpeg).  If in doubt use -framerate instead of the input option -r.

  As an output option,
     - duplicate or drop input frames to achieve constant output frame rate fps.

  If you would like to generate an image every 30 seconds:

    :code:`-r 1/30` as an output option, this means at the end of the command.

    :code:`ffmpeg -i <input> -f image2 -update 1 -r 1/30 img.jpg`

    This is not a very efficient method as durring 29 seconds ffmpeg is processing input frames for nothing, but it does work.

  Better would be to specify to output a single frame with the :code:`-vframes 1` option and then restart the command every 30 seconds.



Example usage
=============

Grab from an RTSP feed
-----------------------

:code:`ffmpeg -v warning -rtsp_transport tcp -i rtsp://<ip_addr> -f image2 -update 1 -r 1/10 img.jpg`

::

    echo off
    :loop
      ffmpeg -v warning -rtsp_transport tcp -i rtsp://<ipaddr>:554/live/ch00_0 -f image2 -strftime 1 -vframes 1 -y img%%M%%S.jpg
      timeout /t 3 /nobreak
      choice /C ˆ /D ˆ /N /T 3
      goto loop

 
Resources
---------
  - `ffmpeg -vframes option <https://trac.ffmpeg.org/wiki/Create%20a%20thumbnail%20image%20every%20X%20seconds%20of%20the%20video>`_
  - https://ffmpeg.org/ffmpeg.html
  - https://www.ffmpeg.org/ffmpeg-formats.html
  - https://stackoverflow.com/questions/25360470/ffmpeg-capture-current-frame-and-overwrite-the-image-output-file/27516825#27516825
