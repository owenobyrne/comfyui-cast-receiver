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

## The remote

Hiding the platform's controls overlay takes the remote's playback UI with it, which leaves the
d-pad doing nothing. It is put back to work here:

| button | does |
|---|---|
| left / right, tapped | step one frame back / forward |
| left / right, held | play at half speed, forward or backward |
| centre, or the play-pause key | play / pause |
| up / down | previous / next item in the gallery |

Tap and hold are told apart by `event.repeat`, which is all a d-pad gives you. The first press
always steps, so a tap is never delayed waiting to see whether it becomes a hold; if repeats
start arriving, motion takes over from where the step left off. Releasing restores the rate and
whatever the clip was doing beforehand, rather than leaving it playing or paused as a side
effect.

Forward uses the video's own `playbackRate`. Backward cannot: a negative `playbackRate` is not
supported, so reverse is a seek loop that walks `currentTime` back by however much wall clock
has passed. It ticks at 100ms rather than per frame, because a television seeking backwards is
decoding far more than it appears to.

Repeats are **ignored** for up/down and for play/pause. Holding down would otherwise tear
through the gallery an item per repeat with no way to stop on the one you wanted.

Play/pause goes through `PlayerManager` rather than the media element directly, so the phone's
own transport stays in step with the television. Stepping does not: it is `currentTime`
arithmetic, for the same reason the phone's step button sends a message instead of a seek.

Frame rate cannot be read from a `<video>`, so it arrives from the phone (which reads it off the
decoder) and is remembered for the remote's use — a remote press can easily come before the
phone has ever sent a step. It is 24 until told otherwise.

Up and down move through the **Cast queue**, not through anything invented here. Giving each
item the URLs of its neighbours would put a second copy of the gallery's ordering on the
television, where it could drift from the phone's; the queue is already that ordering. "Cast
all" has always loaded a real multi-item queue, and the phone now sends the surrounding items
for a single cast too, so there is something to move through either way. `sendLocalMediaRequest`
with `QUEUE_NEXT`/`QUEUE_PREV` is the supported way for a receiver to issue itself a media
command, and it keeps `RemoteMediaClient`'s current item correct — which is what the phone reads
to keep its own transport pointed at the right file.

The centre button matches `Enter`, `Select`, space, Android's `KEYCODE_DPAD_CENTER` (23) and the
media play/pause key. It was bound to `Enter` alone at first and did nothing on real hardware: a
television remote is not a keyboard.

The key handler runs in the capture phase and consumes what it handles, because the SDK's own
key handling is still attached even with its controls hidden.

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
URL arrives at load time from the phone. That question is now settled — it does — and the
`#status` overlay that existed to answer it has been removed.

## The two overlays, and which is which

Two different things draw over the video, and confusing them wasted a deploy:

- **The default receiver's metadata banner.** Gone by virtue of this receiver existing at all;
  it was never ours to draw.
- **The platform's playback overlay** — the title and scrubber a television puts up when
  playback starts or pauses. This one survives into a custom receiver, because it is not the
  receiver's. There is no option for it, and which element it *is* depends on the device.

Reading the shipped `cast_receiver_framework.js`, the SDK classifies the device (`ie()`) and
gives each class a different UI:

| `ie()` | device | overlay |
|---|---|---|
| 1 | plain non-touch | `document.createElement("tv-overlay")`, swapped into the shadow DOM in place of `<tv-overlay-placeholder>` |
| 2 | non-touch **with a d-pad** | `a.A(true).getTouchControlsElement()`, built by `/media_player.js` |
| 3, 4 | touch / Android with touch | touch-optimised controls |
| 5, 6 | audio-only, automotive | none / other |

**A Google TV is type 2, not type 1** — no touch, but a d-pad remote. The first attempt here hid
`tv-overlay` by name and changed nothing on real hardware for exactly that reason. Worse, type
2's element comes from `/media_player.js`, which is served by the **device's own web server**
(hence the `port-for-web-server` platform value) and 404s on gstatic — so it cannot be read, or
even named, from a development machine.

There is a `dpad-controls-overlay-disabled` platform value that skips the controls path
entirely, but it is read *from* the device and a receiver page cannot set it.

So `hidePlatformChrome()` names nothing. It keeps the `<video>` and its ancestor chain and hides
everything else inside the player, which does not depend on knowing what any of it is called,
and repeats on a `MutationObserver` because the controls are built lazily, long after the page
script runs.

**Two consequences:**

- The TV's own remote loses its playback UI. That suits this setup — the phone is the remote —
  but it is a real loss.
- **If an overlay survives this, it is not in the page.** It would be Google TV system UI drawn
  above the web view, which no receiver-side change can reach. The version badge is how to tell
  that apart from a stale cached page.

## The build badge

The bottom-left corner shows the ComfyUI mark and a version, e.g. `2026-09-11.1`. It answers
exactly one question: which build of this page the device is actually running. Cast devices hold
a receiver page across sessions, so "I pushed a fix and nothing changed" has two explanations,
and without a visible marker there is no telling them apart from the sofa.

`RECEIVER_VERSION` at the top of the script is the value, **and it is hand-maintained — bump it
whenever this file changes.** A version that silently stops moving is worse than none: it reads
as proof the device is current while proving nothing. It is a date rather than a serial because
the question is always "is this recent?", which a date answers without having to remember what
the current number should be.

Stamping it automatically at deploy time would remove that failure mode, but Pages publishes
this branch directly with no build step; swapping to an Actions-based deploy is a change to how
the live receiver ships and worth doing deliberately rather than as a side effect.

The icon is the app's own mark, inlined as a data URI so this page stays a single self-contained
file with no second request to 404.

## Verifying a change here

There is no browser emulator: the page loads in Chrome, but `CastReceiverContext.start()` needs
platform APIs only real hardware has. With a cast in progress the device opens port 9222, so
`http://<tv-ip>:9222` in desktop Chrome gives console, elements and network against the live
page — which is also where the `[comfyui-receiver]` log lines go now that nothing is drawn on
screen. The port is closed when nothing is casting, which is why a scan finds nothing until you
start.
