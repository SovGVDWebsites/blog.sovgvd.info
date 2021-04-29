Some good and useful Xiaomi Yi hacks. (1.2.x firmware)

# enable RAW
```
t app test debug_dump 14
```

Also you need convert it to DNG and do some corrections [URL to software](https://drive.google.com/file/d/0B4tyaJWIqCb_cDJkTUxCVTlFSE0/view?usp=sharing). Don't use with timelapse - RAW files too big for this.

# better JPEG
## 100% quality?
```
writeb 0xC0BC205B 0x64
```

(be careful, could be not for 1.2.x fw)

## ISO100
```
t ia2 -ae exp 100 0 0
```

(be careful, shutter speed could be too long)

## ISO400
```
t ia2 -ae exp 400 0 0
```

# better video
## 2K
```
writeb 0xC06CE446 0x02
```

## 35Mb/s
```
writew 0xC05C1006 0x420C
```

# WDR and some other PRO-like magick
```
t ia2 -adj ev 0 0 120 0 0 130 0
t ia2 -adj l_expo 163
t ia2 -adj autoknee 255
t ia2 -adj gamma 220
t cal -sc 14
```

# disable noise reduction
```
t ia2 -adj tidx -1 0 -1 
```
