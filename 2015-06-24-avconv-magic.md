A lot of usefull scripts for video and photo editing =)
<cut>
# photo jpeg or raw to slideshow
## PHP with crop to 1080p
Some PHP magic to do "prejob"
```
    $pic_id=1;
    $dirs=array("100MEDIA","101MEDIA","102MEDIA","103MEDIA");
    foreach ($dirs as $d) {
    	$fs=scandir($d);
    	foreach ($fs as $f) {
    		if (is_file($d."/".$f)) {
    			copy ($d."/".$f,"pics/yi".str_pad($pic_id, 8, '0', STR_PAD_LEFT).".jpg");
    			print $pic_id.".";
    			$pic_id++;
    		}
    	}
    }
```

And magic:
```
avconv -r 25 -start_number 1 -i pics/yi%08d.jpg -b:v 50000k -vf crop=4608:2592:0:0 -s 1920x1080 test_25.mp4
```

## RawTherapee RAW to 4K timelapce
Prepare profile in RawTherapee and save profile (`profile.pp3`), then convert RAWs to JPEGs (or may be PNG). Go to path before (1 level up) directory with RAWs (`path_with_raw`)
```
rawtherapee -p /path/to/profile.pp3 -o /path/to/converted/photos -c path_with_raw
```

Prepare concat file
```
cd /path/to/converted/photos && for f in *.jpg; do echo "file '$f'" >> to_concat.txt; done
```

Convert it to 4K (`-s 3840x2160`) timelapse 29.97 fps (`-r 30000/1001`), also crop photo to 16:9 and move it to bottom in 400px (`-vf crop=4592:2583:0:400`)
```
ffmpeg -r 30000/1001 -safe 0 -f concat -i to_concat.txt -sws_flags lanczos -s 3840x2160 -vf crop=4592:2583:0:400 -crf 15 time_4k.mp4
```

# timelapse
based on [https://ubuntuforums.org/showthread.php?t=2022316](https://ubuntuforums.org/showthread.php?t=2022316)
```
mkdir renamed
counter=1
ls -1tr *.JPG | while read filename; do cp $filename renamed/$(printf %05d $counter)_$filename; ((counter++)); done
cd renamed
wget https://github.com/cyberang3l/timelapse-deflicker/raw/master/timelapse-deflicker.pl
chmod +x timelapse-deflicker.pl
./timelapse-deflicker.pl -v
cd Deflickered/
ffmpeg -pattern_type glob -i '*.JPG' -vf setdar=16:9  -crf 19 -r 30000/1001 video.mp4
ffmpeg -pattern_type glob -i '*.JPG' -vf scale=1280x720,setdar=16:9  -crf 19 -r 30000/1001 video.720.mp4
```
Be careful with file sort, sort by name
```
ls -1 *.JPG
```

Sort by date
```
ls -1tr *.JPG
```

# video stab

## ffmpeg
First step to make motion detect
```
ffmpeg -i video.mp4 -vf vidstabdetect=stepsize=6:shakiness=7:accuracy=15:result=transform_vectors.trf -f null -
```

Second step for do the job (video stabilization, rotate 90, rotate 90, do it sharpen in x264 with almost lossless and without sound)
```
ffmpeg -i video.mp4 -vf vidstabtransform=input=transform_vectors.trf:zoom=0:smoothing=20,transpose=2,transpose=2,unsharp=5:5:0.8:3:3:0.4 -vcodec libx264 -preset slow -profile:v high -level 41 -tune film -crf 15 -an video.stabed.mp4
```

Batch motion detect (first step)
```
for f in *.MP4; do ffmpeg -i $f -vf vidstabdetect=stepsize=6:shakiness=7:accuracy=15:result=transformvectors.$f.trf -f null -; done
```

Second step
```
for f in *.MP4; do ffmpeg -i $f -vf vidstabtransform=input=transformvectors.$f.trf:zoom=0:smoothing=20:unsharp=5:5:0.8:3:3:0.4 -crf 20 -preset slow -tune film $f.stab.mp4; done
```

## ffmpeg stabilization for images sequence
Get image sequnce from video (for ex. for timelapse)
```
ffmpeg -i video.mp4 -vf fps=5 image-%5d.png
```

Detect (`eq=contrast=1.4` only for "flat" color, ex. DJI Phantom d-log or gopro protune):
```
ffmpeg -i image-%05d.png -vf eq=contrast=1.4 -pix_fmt yuv420p -f yuv4mpegpipe pipe: | ffmpeg -i pipe: -pix_fmt yuv420p -vf "vidstabdetect=stepsize=6:shakiness=6:accuracy=15:result=transform_vectors.trf" -f null -
```

Save images sequence:
```
ffmpeg -i image-%05d.png -pix_fmt yuv420p -f yuv4mpegpipe pipe: | ffmpeg -i pipe: -vf vidstabtransform=input=transform_vectors.trf:zoom=0:smoothing=60,unsharp=5:5:0.8:3:3:0.4 imgstabed-%05d.png
```


## hugin
Video stabilization using Hugin for timelapse, looks very very great, but required a lof of RAM. This code will crop video, get every 5 frame `fps=5` (todo: or get every 5 frames and crop them?), found control points and make a lof of TIFFs `align_image_stack` and crop it (`-C`), next ffmpeg convert it to nice and amost stable timelapse
```
ffmpeg -i 'myvideo.mp4' -vf crop=2704:760:0:0,fps=5 image-%5d.png && align_image_stack -C -a tif *.png && ffmpeg -r 30000/1001 -start_number 1 -i tif%04d.tif -crf 18 stab.mp4
```

## transcode

```
transcode -J stabilize=shakiness=5:accuracy=15 -i test.mp4 && transcode -J transform=smoothing=20 -i test.mp4 -y xvid -w 50000 -o test.mp4.stab-5-15-20.mkv
```

For some iterlaced MTS we need specified frame rate and remove audio to prevent segfault
```
...transcode -J transform=smoothing=20 -i test.mp4 -y xvid -w 50000 --export_frc 3 -o test.mp4.stab-5-15-20_25fps.mkv
```

Return audio:
```
avconv -i video.stab.mp4 -i 00002.MTS -map 0:v:0 -map 1:a:0 -vcodec copy -acodec copy video.final.mp4
```

*export_frc* table

|id|frame rate|
|--|-----------|
|1 | 23.976   (24000/1001.0|
|2 | 24|
|3 | 25|
|4 | 29.970   (30000/1001.0)|
|5 | 30|
|6 | 50|
|7 | 59.940   (2 * 29.970)|
|8 | 60|
|9 |  1|
|10 |  5|
|11 | 10|
|12 | 12|
|13 | 15|

To remove autozoom
```
...transcode -J transform=smoothing=60:crop=1:optzoom=0 -i DSCN3205.MOV -y xvid -w 50000 -o DSCN3205.MOV.stab-5-15-60-z0.mkv
```

Deinterlace and force 25fps
```
avconv -i 00010.MTS -acodec copy -filter:v yadif -qmin 10 -r 25 -y tostab10.mp4
```

# video to photo
```
avconv -i video.mp4 image-%3d.png
```


# defisheye avconv
```
avconv -i to_stab.mp4 -qmax 15 -vf frei0r=defish0r:0.75:y:0.6:0 -an to_stab_def.mp4
```

Actually values just copied from somewhere. Could be not the best.

# downscale and external sound with mapping
```
avconv -i 4k_video.avi -i raw_audio.wav -map 0:v:0 -map 1:a:0 -sws_flags lanczos -s 1920x1080 -preset slow -tune film -profile:v high -level 41 -crf 15 -c:a aac -b:a 192k -ar 48000 -sample_fmt fltp -strict experimental 1080p_final.mp4
```

# batch file processing
```
for f in *.MP4; do avconv -i "$f" -s 1280x720 -preset medium -tune film -profile:v high -level 41 -crf 18 -c:a aac -b:a 128k -ar 48000 -sample_fmt fltp -strict experimental "${f%.avi}.720p.mp4"; done
```


# concat mp4 files with encoding
Create file with files list
```
for f in *.MP4; do echo "file '$f'" >> to_concat.txt; done
```

Do job
```
ffmpeg -f concat -i to_concat.txt -s 1280x720 -preset medium -tune film -profile:v high -level 41 -crf 18 -c:a aac -b:a 128k -ar 48000 -sample_fmt fltp -strict experimental all.720p.mp4
```

# ffmpeg slowmotion 60fps to 30fps or 60fps to 24fps

60fps to 24fps 2.5 slower (not tested)
```
ffmpeg -i GP012594.MP4 -r '24000/1001' -filter:v "setpts=2.5*PTS" -sws_flags lanczos -s 2704x1520 -preset slow -tune film -profile:v high -level 41 -crf 15 -an GP012594.slow_2k.MP4 
```

increase video frame size (if you need)
```
-sws_flags lanczos -s 2704x1520
```
 
