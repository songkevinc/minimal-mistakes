# Using ffmpeg to make plot movies
ffmpeg \
    -framerate 12 -i frames/final.protein.%05d.tga \
    -c:v libx264 \
    -preset slow \
    -crf 18 \
    -r 24 \
    movie.mp4

This will make a movie from VMD frames. You can change the filename format string (%05d means zero-padded integers to width 5 (default from VMD)). The file indices have to be contiguous.

-framerate 24
You can change this to speed up or slow down your movie. The output framerate is given by the -r option.
-c:v libx264
Use H.264 to encode your movie. This is a good, well-supported encoding that seems to be everyone’s favorite.
-preset slow
Try hard to reduce the file size. It’s not that slow.
-crf 18
“Quality”. This means essentially lossless. Higher is worse. ffmpeg documentation here and explanation here.
-r 24
Output frame rate. If you are speeding up your rendered frames with -framerate > 24, make sure you set this or your movie will look jittery in Powerpoint.