Exploring ffmpeg setpts for slowmotion / fastmotion video
==========================================================

An explanation of the **pts**, presentation time stamp, can be found [here](https://stackoverflow.com/questions/43333542/what-is-video-timescale-timebase-or-timestamp-in-ffmpeg/43337235#43337235)

Timebase = 1/75; Timescale = 75
 Frame        pts           pts_time
   0          0          0 x 1/75 = 0.00
   1          3          3 x 1/75 = 0.04
   2          6          6 x 1/75 = 0.08
   3          9          9 x 1/75 = 0.12
   ...

In a constant frame rate video, we can interpret the relationship between Frame N and PTS as the following linear function.

<image here>

The derivative (slope) of this function is the speed at which it is reproducing the frames, i.e. 1 in this case, as we have normalized it for simplicity.

The ffmpeg documentation describing the parameters we can touch can be found [here](http://ffmpeg.org/ffmpeg-all.html#setpts_002c-asetpts)


`ffmpeg -i input.mp4 -an -vf "setpts=PTS*1.0" output.mp4`

will not modify the timestamps at all.
We can view this function as `f(pts)=pts*1.0`, just as the linear graph presented before.

`ffplay -i input.mp4 -an -vf "setpts=2.0*PTS" output.mp4`

will play at half speed
We can view this function as `f(pts)=pts*2.0`, just as the linear graph presented before, but with a higher slope, specifically 2. This means that each frame is more spaced out in time and therefore it will take longer to reproduce the same frames.

`ffplay -i input.mp4 -an -vf "setpts=0.5*PTS" output.mp4`

will play at double speed
We can view this function as `f(pts)=pts*0.5`, just as the linear graph presented before, but with a lower slope, specifically 0.5. This means that each frame is more tightly spaced in time and therefore it will take shorter to reproduce the same frames.

The following graph shows these pts transformations.

<image here>

you could view the multiplier as `setpts=(<input_speed>/<output_speed>)*...` so a number greater than 1 would mean that output is slower than the input and vice-versa

Piece-wise speed modification
-----------------------------

As presented in [this stackoverflow answer](https://video.stackexchange.com/a/21804/23130) we can build a complex filter where we apply a pts redefinition at specific parts of a video.

ffmpeg -i in.mp4 -an -filter_complex ^
[0:v]trim=0:1,setpts="(PTS-STARTPTS)*0.5"[v1];^
[0:v]trim=1,setpts="(PTS-STARTPTS)*2.0"[v2];^
[v1][v2]concat=n=2:v=1 -y ssout1.mp4

With this setup it is important to note that we cannot just use PTS directly otherwise, unexpected results will appear.

PTS - "The presentation timestamp in input"
STARTPTS - "The PTS of the first frame."

The PTS of a file increases as time goes by. PTS at t=0.5s may be 1000 and at t=1s may be 2000. If we split a video by trim the first frame that enters the first trim=0:1 will be PTS=0 and STARTPTS=0, but in the second trim the first frame will be PTS=2000 and STARTPTS=2000. If we want everything 0 based, we should take this into account by defining our PTS to modify as a relative one by subracting the first frame PTS on the input from the current PTS = `(PTS-STARTPTS)`.


Creating speed Ramps
---------------------
To create a ramp we need to have a counter which idealy progress from 0 to 1 as the frames we process increase.

ffmpeg offers us the following variables which may be of interest to us:

N - The count of the input frame for video or the number of consumed samples, not including the current frame for audio, starting from 0.
FRAME_RATE, FR - frame rate, only defined for constant frame-rate video

so, lets assume we are working with a constant frame-rate video. We can design a formula as such.

`[0:v]trim=0:2,setpts="(PTS-STARTPTS)*(0.5+0.5*N/(FR*2))";`

trim=0:2 - we work on a 2 second segment of our input video
FR*2 - is the total number of frames in 2 seconds of video

Here we see that our ramp starts with a multiplier of 0.5 and as we move ahead in time we will reach a multiplier of 1 when we have reached the total number of frame in this segment.

Previously we multiplied our existing function by a constant, now we are multiplying the function by another function.

With a bit of simplification we can consider it as `y=x*(0.5+0.5x)`.

The speed at which it is going will be the derivative of this function: `y'=0.5+x` , as we can see this is a linear **speed ramp**.

Following a couple examples of ramps and their derivatives corresponding to the previous formula and the following:

`[0:v]trim=0:2,setpts="(PTS-STARTPTS)*(0.2+0.4*N/(FR*2))";`

<image here>

derivatives of the functions are represented as dashed lines.

Here we can define a bit more generically our formula:

`[0:v]trim=m:n,setpts="(PTS-STARTPTS)*(a+b*N/(FR*(n-m)))";`

a = start speed (a>0)
b = --> a+2*b = end speed
a + b = duration of modified clip

We can see these numbers also in the graph.

Whats about negative slopes? i.e. from slow to faster speeds.

Well theoretically we just need b to be negative, but lets take a look at what we have in our general formula if we consider b < 0

- duration of the modified clip needs to be greater than 0:

`a+b > 0` --> `a > -b`

- we will only consider positive speeds at the moment (no backwards video)

`a+2*b > 0` --> `a > -2b`

In the following graph we can see the inverse slopes of previously graphed ramps:

<image graph here>


**Why we cannot do backwards video with this formula approach**

As we can see in the following graph:
C:\tmp\vid\Backwards.png

Frames from the future would get mapped to the same pts_time as past frames have already been maped.

Examples
---------

SETLOCAL
SET myin=sktin.mp4
SET expr1a="(PTS-STARTPTS)*(1-0.35*N/(FR))"
SET expr1b="(PTS-STARTPTS)*(0.3+0.7*N/(FR*2))"
SET expr1e="(PTS-STARTPTS)*1"
SET expr1c="(PTS-STARTPTS)*(1.7-0.35*N/(FR))"
SET expr1d="(PTS-STARTPTS)*(1-0.4*N/(FR*2.5))"

ffmpeg -v info -i %myin% -an -filter_complex ^"^
[0:v]trim=0:1,setpts=%expr1a%[v1];^
[0:v]trim=1:3,setpts=%expr1b%[v2];^
[0:v]trim=3:4,setpts=%expr1c%[v3];^
[0:v]trim=4:7,setpts=%expr1d%[v4];^
[0:v]trim=7,setpts=%expr1e%[v5];^
[v1][v2][v3][v4][v5]concat=n=5:v=1" -y ssout1.mp4
ffmpeg -v warning -i %myin% -i ssout1.mp4 -an -filter_complex ^
"[0:v][1:v]hstack[v]" -map "[v]" -y tmpcmp1.mp4
ffplay -v warning tmpcmp1.mp4
