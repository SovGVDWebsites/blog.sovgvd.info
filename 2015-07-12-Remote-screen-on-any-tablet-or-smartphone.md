# How to create additional remote display for Linux powered device

Create virtual screen ```xrandr --addmode VIRTUAL1 XXXXxYYY``` with size XXXX YYY, like 1920x1080

Use any application for setup screen to right (right by default in Ubuntu)

VNC server for incoming connections (be careful with security) with cropped part of screen ```x11vnc -clip XXXXxYYY+you_real_monitor_width+0 -many```

![pic1](https://github.com/SovGVDWebsites/blog.sovgvd.info/blob/main/pics/a23/107/a23107952fe145bf86c0232d7e43a497.jpg?raw=true)
