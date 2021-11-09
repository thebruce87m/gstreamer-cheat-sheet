# gstreamer-cheat-sheet

Send a stream from one PC to another:


Server:

Note: change `ip-address-of-client` to the hostname or ip address of the target

```
gst-launch-1.0 videotestsrc ! queue ! x264enc ! queue ! rtph264pay ! queue ! udpsink host=ip-address-of-client port=9002
```

Client:

```
gst-launch-1.0 udpsrc port=9002 caps="application/x-rtp" ! queue ! rtph264depay ! queue ! avdec_h264 ! queue ! autovideosink
```
