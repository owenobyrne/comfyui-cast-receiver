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

## Playlists come for free

This page is built on the Cast Application Framework (`cast_receiver_framework.js`,
`cast.framework.CastReceiverContext` + `PlayerManager`) rather than a bare `<video>` tag with
hand-rolled message handling. One consequence worth knowing: **CAF's stock `PlayerManager`
already handles a multi-item Cast queue** — auto-advancing between items and keeping
`RemoteMediaClient` state in sync on every transition — with no code here at all. The phone
app's "Cast all"/"Shuffle" (see `CastController.castPlaylist` in `android-comfyui`) is a real
`queueLoad` with several `MediaQueueItem`s rather than one item at a time, and needed no
changes on this side. The single-video load this receiver already handled was itself a
one-item queue (used to get `REPEAT_MODE_REPEAT_SINGLE` looping), so a playlist is the same
mechanism at a different size, not a new one.

**Not built, and not free:** a cross-fade transition between playlist items. CAF's
`PlayerManager` hard-cuts between queue items with no transition API, so that would mean
bypassing it — two manually-managed `<video>` elements, custom preload/fade timing at each
queue boundary. Most Chromecast/Google TV hardware also has only one hardware video decoder,
so decoding two clips at once to cross-fade them risks dropped frames or an outright failure —
unverified either way, and the kind of thing worth measuring on a real device before building.
If cross-fade is wanted, pre-rendering the sequence server-side (ffmpeg's `xfade` filter) and
casting the result as one ordinary item avoids all of this.

## Why it is hosted here

Cast receiver applications must be served over HTTPS with a valid certificate. The media
itself is a plain `http://` URL on a private LAN, served by the sibling
`comfyui-output-gallery-api`. Whether a Chromecast will load HTTP media from an HTTPS receiver
is the first thing this deploy is meant to establish — Google's own default receiver does
exactly that today, so it is expected to work, but it had not been proven for a custom one.

No address, credential or other detail of the private setup is baked into this page; the media
URL arrives at load time from the phone.

The `#status` overlay is diagnostic and should be removed once playback is confirmed.
