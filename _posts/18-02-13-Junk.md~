
# Using ffmpeg to make plot movies


```python
%%HTML
<iframe width="560" height="224" src="https://songkevinc.github.io/images/tic1_movie.mp4" frameborder="0" allowfullscreen></iframe>

```


<iframe width="560" height="224" src="https://songkevinc.github.io/images/tic1_movie.mp4" frameborder="0" allowfullscreen></iframe>


In the video above, I'm showing the structure of a protein at the corresponding TIC values. The movie is designed so that you can see the structures of the monomer as you move from left to right in TIC1 space. Prior to using FFMPEG to combine the 2 movies together, there are a couple of things we need to do:

1) Generate a sequence of images where the purple dot is moving from left to right along TIC1 space, but keep the TIC1 vs TIC2 heatmap in the background.

2) Generate protein images from VMD that correspond to the purple dot in the plot


## After you finish steps 1 and 2, we can use the final magic command:

ffmpeg -framerate 24 -i img1_name_%03d.png\ <br \>
       -framerate 24 -i img2_name.%05d.tga\ <br \>
       -c:v libx264 -vcodec libx264 -pix_fmt yuv420p\ <br \>
       -preset slow -crf 18\ <br \>
       -filter_complex "[0:v]scale=768:512 [left] ;[1:v]scale=512:512 [right]; [left][right]
       hstack" final_image_name.mp4 <br \>

# There's a lot to consume in the above magic command. Here's what they actually mean:

`-framerate 24 -i img1_name_%03d.png\` <br />
We're controlling the framerate of the first image sequence. In my case, this is the TIC1 vs TIC2 heatmap. <br />
<br />
`-framerate 24 -i img2_name.%05d.tga\` <br />
Same thing as above, but now we're loading the second image sequence, which is the sequence of protein structures.

`-c:v libx264 -vcodec libx264 -pix_fmt yuv420p\` <br />
Use H.264 to encode your movie. This seems to be everyone's favorite library.

`-preset slow -crf 18` <br />
Try hard to reduce the file size. It's not that slow. crf indicates quality. Higher means worse, but 18 is essentially quality lossless.

`-filter_complex "[0:v]scale=768:512 [left] ;[1:v]scale=512:512 [right]; [left][right]           hstack" final_image_name.mp4` <br />
This is the main beast of the command. We're applying a transformation on video1 and video2 and stacking them together. In the first line `[0:v]scale=768:512 [left]` means we're rescaling video1 so that the height will match the second video and labeling as the left video. In the second line, `[1:v]scale=512:512 [right]`, we're basically doing the same thing and labeling as right video. In our final line, `[left][right] hstack`, we're horizontally stacking the left and right video, and we're outputting it to `final_image_name.mp4`


```python


```
