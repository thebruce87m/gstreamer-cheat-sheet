# gstreamer-cheat-sheet

## Basic Files

### Record example

1fps example video:

```bash
gst-launch-1.0 -e videotestsrc \
! video/x-raw,width=640,height=480,framerate=1/1 \
! timeoverlay font-desc="Sans, 50" \
! queue \
! x264enc \
! mp4mux \
! filesink location=video.mp4
```


### H265 mp4 file, manual decode pipeline:

```bash
gst-launch-1.0 filesrc location="./file.mp4" \
! qtdemux \
! h265parse \
! avdec_h265  \
! videoconvert \
! autovideosink
```

## RTSP Stream

Notes:

* For `rtspsrc` use `location=rtspt:...` to enable tcp
* For `rtspsrc` use `latency=0` to reduce latency
* For `h264parse` use `config-interval=-1` to send with every IDR frame

### Simple way to play a stream
```bash
gst-play-1.0 rtsp://username:password@192.168.0.110/ch0/stream0
```


### Display H264 Stream with explicit decode pipeline
```bash
gst-launch-1.0 -v rtspsrc location=rtsp://username:password@192.168.0.110/ch0/stream0 \
! rtph264depay \
! avdec_h264 \
! videoconvert \
! autovideosink
```


### Display an RTSP stream with automatic decode using [decodebin](https://gstreamer.freedesktop.org/documentation/playback/decodebin.html?gi-language=c)

Will automatically decode H264, H265

```bash
gst-launch-1.0 -v rtspsrc location=rtsp://username:password@192.168.0.110/ch0/stream0 \
! decodebin \
! videoconvert \
! autovideosink
```

### Save a single image from a stream

```bash
gst-launch-1.0 -e rtspsrc location=rtsp://username:password@192.168.0.110/stream0 \
! rtph265depay \
! h265parse \
! avdec_h265 \
! videoconvert \
! jpegenc snapshot=TRUE \
! filesink location=save.jpg
```


### Save a stream directly to an H264 file

```bash
gst-launch-1.0 -e rtspsrc location=rtsp://username:password@192.168.0.110/ch0/stream0 \
! rtph264depay \
! h264parse \
! mp4mux \
! filesink location=file.mp4
```

### Save stream to multiple mp4 file chunks, max size of each is 10 seconds

Note:

* May need `h264parse config-interval=-1` to inject decoding info into file to allow playback
* `10000000000` is 10 seconds in nanoseconds

```bash
gst-launch-1.0 -e rtspsrc location=rtsp://username:password@192.168.0.110/ch0/stream0 \
! rtph264depay \
! h264parse \
! splitmuxsink location=file-%03d.mp4 max-size-time=10000000000
```

### Save stream to multiple ts file chunks, max size of each is 10 seconds

Note:

* `next-file=5` tells multifilesink to be in "max-size" mode
* `10000000000` is 10 seconds in nanoseconds

```bash
gst-launch-1.0 -e \
rtspsrc location=rtsp://admin:password@192.168.0.154/ch0/stream0 \
! rtph264depay \
! h264parse \
! mpegtsmux \
! multifilesink next-file=5 max-file-duration=10000000000 location=video_%02d.ts
```

### Play back multiple ts files sequentially
```bash
gst-launch-1.0 -v \
multifilesrc location=video_%02d.ts \
! decodebin \
! videoconvert \
! autovideosink
```

### Save stream to HLS - hlssink

```bash
gst-launch-1.0 -e \
rtspsrc location=rtsp://admin:password@192.168.0.154/ch0/stream0 \
! rtph264depay \
! h264parse \
! mpegtsmux \
! hlssink
```

Example file output:
```
playlist.m3u8
segment00000.ts
segment00001.ts
segment00002.ts
segment00003.ts
...
```

### Save stream to HLS - hlssink2

Note: 

* `playlist-length=0` to save all chunks, otherwise only the last 5 will be kept

```bash
gst-launch-1.0 -e rtspsrc location=rtspt://admin:password@192.168.0.154/ch0/stream0 \
! rtph264depay \
! h264parse \
! hlssink2 playlist-length=0
```

### Play back hls file

```bash
gst-launch-1.0 filesrc location="playlist.m3u8" \
! hlsdemux \
! decodebin \
! videoconvert \
! videoscale \
! autovideosink
```


### Record two streams at once:
```bash
gst-launch-1.0 -e \
rtspsrc location=rtsp://admin:password@192.168.0.154/ch0/stream0 \
! rtph264depay \
! h264parse \
! mp4mux \
! filesink location=154.mp4 \
\
\
rtspsrc location=rtsp://admin:password@192.168.0.145/ch0/stream0 \
! rtph264depay \
! h264parse \
! mp4mux \
! filesink location=145.mp4
```

### Play both recordings

```bash
gst-launch-1.0 \
filesrc location=145.mp4 \
! decodebin \
! videoconvert \
! autovideosink \
\
\
filesrc location=154.mp4 \
! decodebin \
! videoconvert \
! autovideosink
```

# NVIDIA JETSON / Low latency using nveglglessink

```bash
gst-launch-1.0 -v rtspsrc latency = 0 location=rtsp://admin:password@192.168.250.120:554/stream1 \
! rtph265depay \
! h265parse \
! nvv4l2decoder \
! nvvideoconvert \
! timeoverlay halignment=right valignment=bottom text="Stream time:" shaded-background=true font-desc="Sans, 24" \
! video/x-raw,width=1920,height=1080 \
! nveglglessink sync=0
```

# NVIDIA JETSON / Two incoming streams tiled

```bash
gst-launch-1.0 \
\
\
rtspsrc location=rtsp://admin:password@192.168.250.120:554/stream1 latency=200 ! \
rtph265depay ! \
h265parse ! \
nvv4l2decoder ! \
queue ! \
mux.sink_0 \
\
\
rtspsrc location=rtsp://admin:password@192.168.250.145:554/stream1 latency=200 ! \
rtph265depay ! \
h265parse ! \
nvv4l2decoder ! \
queue ! \
mux.sink_1 \
\
\
nvstreammux name=mux width=1920 height=1080 batch-size=2 batched-push-timeout=40000 ! \
nvmultistreamtiler rows=2 columns=2 width=1920 height=1080 ! \
nvvideoconvert ! \
nveglglessink sync=0
```



## Send a stream from one PC to another:

### udp using 264

Server:

Note: change `ip-address-of-client` to the hostname or ip address of the target

```
gst-launch-1.0 videotestsrc ! queue ! x264enc ! queue ! rtph264pay ! queue ! udpsink host=ip-address-of-client port=9002
```

Client:

```
gst-launch-1.0 udpsrc port=9002 caps="application/x-rtp" ! queue ! rtph264depay ! queue ! avdec_h264 ! queue ! autovideosink
```

### tcp using mjpeg

Server:
```bash
gst-launch-1.0 videotestsrc is-live=true ! videoconvert ! jpegenc ! multipartmux ! tcpserversink host=0.0.0.0 port=8081
```

Client:
```bash
gst-launch-1.0 tcpclientsrc host=0.0.0.0 port=8081 ! multipartdemux ! jpegdec ! autovideosink
```

## Undistort

Undistort script:

```bash
#!/bin/bash

DATA="<?xml version=\"1.0\"?><opencv_storage><cameraMatrix type_id=\"opencv-matrix\"><rows>3</rows><cols>3</cols><dt>f</dt><data>2.85762378e+03 0. 1.93922961e+03 0. 2.84566113e+03 1.12195850e+03 0. 0. 1.</data></cameraMatrix><distCoeffs type_id=\"opencv-matrix\"><rows>5</rows><cols>1</cols><dt>f</dt><data>-6.14039421e-01 4.00045455e-01 1.47132971e-03 2.46772077e-04 -1.20407566e-01</data></distCoeffs></opencv_storage>"

gst-launch-1.0 -vv videotestsrc ! videoconvert ! cameraundistort settings="$DATA" ! videoconvert ! autovideosink
```


Undistort App:

```
//
// g++ calibrate.cpp -o calibrate `pkg-config --cflags --libs gstreamer-1.0 opencv4`
//

#include <gst/gst.h>
#include <opencv2/opencv.hpp>


//
// USB Webcam
//
const char *string = " \
gst-launch-1.0 -v v4l2src device=/dev/video0 ! videoconvert ! cameraundistort name=undist ! cameracalibrate name=cal ! ximagesink \
";

//
// RTSP
//
const char *string2 = " \
gst-launch-1.0 -v rtspsrc location=rtsp://username:password@192.168.0.97/ch0/stream0 ! rtph264depay ! avdec_h264 ! videoconvert ! videoscale \
        ! video/x-raw,width=600,height=300 ! cameraundistort name=undist ! cameracalibrate name=cal ! ximagesink \
";

//
//
//
const char *string3 = " \
gst-launch-1.0 -v rtspsrc location=rtsp://username:password@192.168.0.86/Streaming/Channels/102/ ! rtph264depay ! avdec_h264 ! videoconvert \
 ! cameraundistort name=undist ! cameracalibrate name=cal ! ximagesink \
";


gchar *
camera_serialize_undistort_settings (cv::Mat & cameraMatrix,
    cv::Mat & distCoeffs)
{
  cv::FileStorage fs (".xml", cv::FileStorage::WRITE + cv::FileStorage::MEMORY);
  fs << "cameraMatrix" << cameraMatrix;
  fs << "distCoeffs" << distCoeffs;
  std::string buf = fs.releaseAndGetString ();

  return g_strdup (buf.c_str ());
}

int
main (int argc, char *argv[])
{

  gchar value[100] = {0};

  GstElement *pipeline;
  GstElement *filesrc;
  GstMessage *msg;
  GstBus *bus;
  GError *error = NULL;

  gst_init (&argc, &argv);

  //pipeline = gst_parse_launch ("filesrc name=my_filesrc ! mad ! osssink", &error);
  pipeline = gst_parse_launch (string3, &error);



  if (!pipeline) {
    g_print ("Parse error: %s\n", error->message);
    exit (1);
  }



  gst_element_set_state (pipeline, GST_STATE_PLAYING);

  bus = gst_element_get_bus (pipeline);

  /* wait until we either get an EOS or an ERROR message. Note that in a real
   * program you would probably not use gst_bus_poll(), but rather set up an
   * async signal watch on the bus and run a main loop and connect to the
   * bus's signals to catch certain messages or all messages */
  msg = gst_bus_poll (bus, (GstMessageType) ( GST_MESSAGE_EOS | GST_MESSAGE_ERROR ), -1);

  switch (GST_MESSAGE_TYPE (msg)) {
    case GST_MESSAGE_EOS: {
      g_print ("EOS\n");
      break;
    }
    case GST_MESSAGE_ERROR: {
      GError *err = NULL; /* error to show to users                 */
      gchar *dbg = NULL;  /* additional debug string for developers */

      gst_message_parse_error (msg, &err, &dbg);
      if (err) {
          filesrc = gst_bin_get_by_name (GST_BIN (pipeline), "undist");

        gchar *val;

        g_object_get (filesrc, "settings", &val, NULL);
        
        g_print("Data:\n");

        g_print ("%s", val);

        g_object_unref (filesrc);

        g_print("\nDone\n");

        g_printerr ("ERROR: %s\n", err->message);
        g_error_free (err);
      }
      if (dbg) {
        g_printerr ("[Debug details: %s]\n", dbg);
        g_free (dbg);
      }
    }
    default:
      g_printerr ("Unexpected message of type %d", GST_MESSAGE_TYPE (msg));
      break;
  }
  gst_message_unref (msg);


  gst_element_set_state (pipeline, GST_STATE_NULL);
  gst_object_unref (pipeline);
  gst_object_unref (bus);

  return 0;
}

```

## RTSP streams mosaic with undistort

```
#!/bin/bash

DATA="<?xml version=\"1.0\"?><opencv_storage><cameraMatrix type_id=\"opencv-matrix\"><rows>3</rows><cols>3</cols><dt>f</dt><data>2.85762378e+03 0. 1.93922961e+03 0. 2.84566113e+03 1.12195850e+03 0. 0. 1.</data></cameraMatrix><distCoeffs type_id=\"opencv-matrix\"><rows>5</rows><cols>1</cols><dt>f</dt><data>-6.14039421e-01 4.00045455e-01 1.47132971e-03 2.46772077e-04 -1.20407566e-01</data></distCoeffs></opencv_storage>"


gst-launch-1.0 -e \
    videomixer name=mix \
            sink_0::xpos=0   sink_0::ypos=0  sink_0::alpha=0\
            sink_1::xpos=0   sink_1::ypos=0 \
            sink_2::xpos=0 sink_2::ypos=300 \
            \
            \
    rtspsrc location=rtsp://username:password@192.168.0.110/ch0/stream0 ! rtph264depay ! avdec_h264 ! videoconvert \
        ! queue \
        ! cameraundistort settings="$DATA" \
        ! queue \
        ! videoscale \
        ! video/x-raw,width=600,height=300 \
        ! mix.sink_1 \
        \
        \
    rtspsrc location=rtsp://username:password@192.168.0.118/ch0/stream0 ! rtph264depay ! avdec_h264 ! videoconvert \
        ! decodebin ! videoconvert \
        ! queue \
        ! cameraundistort settings="$DATA" \
        ! queue \
        ! videoscale \
        ! video/x-raw,width=600,height=300 \
        ! mix.sink_2 \
        \
        \
        mix. ! queue ! videoconvert ! queue ! autovideosink

```

# Video Mosaic

```bash
#!/bin/bash

# Input video files

video1="253.mp4"
video2="90.mp4"
video3="98.mp4"
video4="213.mp4"
video5="95.mp4"
video6="94.mp4"


# Mosaic width and height of each video
tile_width=640
tile_height=360



# GStreamer pipeline with dynamic pad linking for decodebin
gst-launch-1.0 \
  compositor name=mix sink_0::xpos=0 sink_0::ypos=0 \
                     sink_1::xpos=$tile_width sink_1::ypos=0 \
                     sink_2::xpos=$(($tile_width * 2)) sink_2::ypos=0 \
                     sink_3::xpos=0 sink_3::ypos=$tile_height \
                     sink_4::xpos=$tile_width sink_4::ypos=$tile_height \
                     sink_5::xpos=$(($tile_width * 2)) sink_5::ypos=$tile_height ! \
  videoconvert ! autovideosink \
  filesrc location="$video1" ! decodebin ! videoconvert ! videoscale ! video/x-raw, width=$tile_width, height=$tile_height ! mix. \
  filesrc location="$video2" ! decodebin ! videoconvert ! videoscale ! video/x-raw, width=$tile_width, height=$tile_height ! mix. \
  filesrc location="$video3" ! decodebin ! videoconvert ! videoscale ! video/x-raw, width=$tile_width, height=$tile_height ! mix. \
  filesrc location="$video4" ! decodebin ! videoconvert ! videoscale ! video/x-raw, width=$tile_width, height=$tile_height ! mix. \
  filesrc location="$video5" ! decodebin ! videoconvert ! videoscale ! video/x-raw, width=$tile_width, height=$tile_height ! mix. \
  filesrc location="$video6" ! decodebin ! videoconvert ! videoscale ! video/x-raw, width=$tile_width, height=$tile_height ! mix.

```



# CMake

```cmake
cmake_minimum_required(VERSION 3.16)

project(PlayMp4)

set(CMAKE_C_STANDARD 11)


find_package(PkgConfig) 
pkg_search_module(GLIB REQUIRED glib-2.0) 
pkg_check_modules(GST REQUIRED gstreamer-1.0)
pkg_check_modules(GST_APP REQUIRED gstreamer-app-1.0)
pkg_check_modules(GST_VIDEO REQUIRED gstreamer-video-1.0)

pkg_search_module(GST REQUIRED gstreamer-1.0>=1.4
        gstreamer-sdp-1.0>=1.4
        gstreamer-app-1.0>=1.4
        gstreamer-video-1.0>=1.4
        )


add_executable(PlayMp4 src/main.c)

target_compile_options(PlayMp4 PRIVATE 
-Wextra
-Wall
-Wfloat-equal
)

target_include_directories(PlayMp4 PRIVATE ${GST_INCLUDE_DIRS})
target_link_libraries(PlayMp4 ${GST_LIBRARIES})
```



# Python example

```python
import cv2
import gi
import numpy as np
gi.require_version('Gst', '1.0')
from gi.repository import Gst, GLib

def gst_to_opencv(sample):
    buf = sample.get_buffer()
    caps = sample.get_caps()
    array = np.ndarray(
        (caps.get_structure(0).get_value('height'),
         caps.get_structure(0).get_value('width'),
         3),
        buffer=buf.extract_dup(0, buf.get_size()),
        dtype=np.uint8)
    return array

def open_video(video_path):
    # Initialize GStreamer
    Gst.init(None)

    # Software decode
    pipeline_str = f'filesrc location={video_path} ! qtdemux ! h265parse ! avdec_h265  ! videoconvert ! video/x-raw,format=BGR ! appsink name=sink emit-signals=true sync=true'

    # nvidia / Jetson
    # pipeline_str = f'filesrc location={video_path} ! qtdemux ! h265parse ! nvv4l2decoder  ! nvvideoconvert ! video/x-raw,format=BGR ! appsink name=sink emit-signals=true sync=true'


    # Create GStreamer pipeline with an explicit output format
    pipeline = Gst.parse_launch(
        pipeline_str
    )

    appsink = pipeline.get_by_name('sink')

    # Start playing
    print('Starting pipeline')
    pipeline.set_state(Gst.State.PLAYING)

    while True:
        sample = appsink.emit('pull-sample')
        if sample:
            buf = sample.get_buffer()
            pts = buf.pts
            if pts is not None:
                # Convert nanoseconds to seconds
                pts_seconds = pts / 1e9
                print(f"PTS: {pts}")

            frame = gst_to_opencv(sample)

            frame = cv2.resize(frame, (800, 600))

            cv2.imshow('Frame', frame)
            if cv2.waitKey(1) & 0xFF == ord('q'):
                break
        else:
            break

    # Clean up
    pipeline.set_state(Gst.State.NULL)
    cv2.destroyAllWindows()

# Example usage
open_video("video.mp4")
```
