# comfyui-cast-receiver

A custom [Google Cast](https://developers.google.com/cast) receiver for the
[android-comfyui](https://github.com/owenobyrne/android-comfyui) phone client.

It exists because three things are impossible with Google's Default Media Receiver, and all
three are trivial once the receiver is your own:

- **No metadata banner.** The default receiver overlays title/artwork whenever playback is
  paused, and no sender-side option turns it off.
- **Frame stepping.** A paused Chromecast ignores `RemoteMediaClient.seek()` for distances
  below roughly 750ms (measured), so a one-frame step is a no-op. Here it is just
  `video.currentTime += 1/fps`.
- **Zoom.** There is no sender API for scaling the picture. Here it is a CSS transform.

## Messages

The phone sends JSON on `urn:x-cast:com.obyrne.comfyui`:

```jsonc
{ "type": "zoom", "scale": 2.0, "x": -10, "y": 5 }   // x/y are percentages of the video box
{ "type": "step", "forward": true, "fps": 24 }
```

## Why it is hosted here

Cast receiver applications must be served over HTTPS with a valid certificate. The media
itself is a plain `http://` URL on a private LAN, served by the sibling
`comfyui-output-gallery-api`. Whether a Chromecast will load HTTP media from an HTTPS receiver
is the first thing this deploy is meant to establish — Google's own default receiver does
exactly that today, so it is expected to work, but it had not been proven for a custom one.

No address, credential or other detail of the private setup is baked into this page; the media
URL arrives at load time from the phone.

The `#status` overlay is diagnostic and should be removed once playback is confirmed.
