# gstreamer-cheat-sheet

## RTSP Stream

Notes:

* Use `location=rtspt:...` to enable tcp

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

### Save a stream directly to an H264 file

```bash
gst-launch-1.0 -e rtspsrc location=rtsp://username:password@192.168.0.110/ch0/stream0 \
! rtph264depay \
! h264parse \
! mp4mux \
! filesink location=file.mp4
```

### Save stream to multiple file chunks, max size of each is 10 seconds

Note:

* May need `h264parse config-interval=-1` to inject decoding info into file to allow playback

```bash
gst-launch-1.0 -e rtspsrc location=rtsp://username:password@192.168.0.110/ch0/stream0 \
! rtph264depay \
! h264parse \
! splitmuxsink location=file-%03d.mp4 max-size-time=10
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

## Send a stream from one PC to another:


Server:

Note: change `ip-address-of-client` to the hostname or ip address of the target

```
gst-launch-1.0 videotestsrc ! queue ! x264enc ! queue ! rtph264pay ! queue ! udpsink host=ip-address-of-client port=9002
```

Client:

```
gst-launch-1.0 udpsrc port=9002 caps="application/x-rtp" ! queue ! rtph264depay ! queue ! avdec_h264 ! queue ! autovideosink
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
