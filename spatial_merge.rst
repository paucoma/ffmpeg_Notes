Spatial Merge two videos
========================

Thanks to another `Stack overflow reply <https://unix.stackexchange.com/questions/233832/merge-two-video-clips-into-one-placing-them-next-to-each-other>`_ :

ffmpeg -i left.mp4 -i right.mp4 -filter_complex \
"[0:v][1:v]hstack=inputs=2[v]; \
 [0:a][1:a]amerge[a]" \
-map "[v]" -map "[a]" -ac 2 output.mp4

`hstack <http://ffmpeg.org/ffmpeg-filters.html#hstack>`_ / `vstack <http://ffmpeg.org/ffmpeg-filters.html#vstack>`_ (equivalent Vertically)
  Stacks input videos horizontally.
	all streams must be of same pixel format and of same height.
  The filter accept the following option:
  *inputs*
    Set number of input streams. Default is 2.
  *shortest*
    If set to 1, force the output to terminate when the shortest input terminates. Default value is 0.

`amerge <http://ffmpeg.org/ffmpeg-filters.html#amerge>`_ will combine the audio from both inputs into a single, multichannel audio stream, and -ac 2 will make it stereo; without this option the audio stream may end up as 4 channels if both inputs are stereo.
