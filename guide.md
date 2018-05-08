---
layout: default
title: Developer guide
weight: 1
---

<div id="table-of-contents">
<div id="table-of-contents-header">Contents</div>
<div markdown="1">
* TOC
{:toc}
</div>
</div>

Playing videos and music is a popular activity on Android devices. The Android
framework provides [`MediaPlayer`][] as a quick solution for playing media with
minimal code. It also provides low level media APIs such as [`MediaCodec`][],
[`AudioTrack`][] and [`MediaDrm`][], which can be used to build custom media
player solutions.

ExoPlayer is an open source, application level media player built on top of
Android's low level media APIs. This guide describes the ExoPlayer library and
its use. It refers to code in ExoPlayer's [main demo app][] in order to provide
concrete examples. The guide touches on the pros and cons of using ExoPlayer.
It shows how to use ExoPlayer to play DASH, SmoothStreaming and HLS adaptive
streams, as well as formats such as MP4, M4A, FMP4, WebM, MKV, MP3, Ogg, WAV,
MPEG-TS, MPEG-PS, FLV and ADTS (AAC). It also discusses ExoPlayer events,
messages, customization and DRM support.

## Pros and cons ##

ExoPlayer has a number of advantages over Android's built in MediaPlayer:

* Support for Dynamic Adaptive Streaming over HTTP (DASH) and SmoothStreaming,
  neither of which are supported by MediaPlayer. Many other formats are also
  supported. See the [Supported formats][] page for details.
* Support for advanced HLS features, such as correct handling of
  `#EXT-X-DISCONTINUITY` tags.
* The ability to seamlessly merge, concatenate and loop media.
* The ability to update the player along with your application. Because
  ExoPlayer is a library that you include in your application apk, you have
  control over which version you use and you can easily update to a newer
  version as part of a regular application update.
* Fewer device specific issues and less variation in behavior across different
  devices and versions of Android.
* Support for Widevine common encryption on Android 4.4 (API level 19) and
  higher.
* The ability to customize and extend the player to suit your use case.
  ExoPlayer is designed specifically with this in mind, and allows many
  components to be replaced with custom implementations.
* The ability to quickly integrate with a number of additional libraries using
  official extensions. For example the [IMA extension][] makes it easy to
  monetize your content using the [Interactive Media Ads SDK][].

It's important to note that there are also some disadvantages:

* **ExoPlayer's standard audio and video components rely on Android's
  `MediaCodec` API, which was released in Android 4.1 (API level 16). Hence they
  do not work on earlier versions of Android. Widevine common encryption is
  available on Android 4.4 (API level 19) and higher.**

## Library overview ##

At the core of the ExoPlayer library is the `ExoPlayer` interface. An
`ExoPlayer` exposes traditional high-level media player functionality such as
the ability to buffer media, play, pause and seek. Implementations are designed
to make few assumptions about (and hence impose few restrictions on) the type of
media being played, how and where it is stored, and how it is rendered. Rather
than implementing the loading and rendering of media directly, `ExoPlayer`
implementations delegate this work to components that are injected when a player
is created or when it's prepared for playback. Components common to all
`ExoPlayer` implementations are:

* A `MediaSource` that defines the media to be played, loads the media, and from
  which the loaded media can be read. A `MediaSource` is injected via
  `ExoPlayer.prepare` at the start of playback.
* `Renderer`s that render individual components of the media. `Renderer`s are
  injected when the player is created.
* A `TrackSelector` that selects tracks provided by the `MediaSource` to be
  consumed by each of the available `Renderer`s. A `TrackSelector` is injected
  when the player is created.
* A `LoadControl` that controls when the `MediaSource` buffers more media, and
  how much media is buffered. A `LoadControl` is injected when the player is
  created.

The library provides default implementations of these components for common use
cases, as described in more detail below. An `ExoPlayer` can make use of these
components, but may also be built using custom implementations if non-standard
behaviors are required. For example a custom `LoadControl` could be injected to
change the player's buffering strategy, or a custom `Renderer` could be injected
to use a video codec not supported natively by Android.

The concept of injecting components that implement pieces of player
functionality is present throughout the library. The default implementations of
the components listed above delegate work to further injected components.
This allows many sub-components to be individually replaced with custom
implementations. For example the default `MediaSource` implementations require
one or more `DataSource` factories to be injected via their constructors. By
providing a custom factory it's possible to load data from a non-standard source
or through a different network stack.

## Getting started ##

For simple use cases, getting started with `ExoPlayer` consists of implementing
the following steps:

1. Add ExoPlayer as a dependency to your project.
1. Create a `SimpleExoPlayer` instance.
1. Attach the player to a view (for video output and user input).
1. Prepare the player with a `MediaSource` to play.
1. Release the player when done.

These steps are outlined in more detail below. For a complete example, refer to
`PlayerActivity` in the [main demo app][].

### Add ExoPlayer as a dependency ###

The first step to getting started is to make sure you have the JCenter and
Google repositories included in the `build.gradle` file in the root of your
project.

```gradle
repositories {
    jcenter()
    google()
}
```

Next add a dependency in the `build.gradle` file of your app module. The
following will add a dependency to the full ExoPlayer library:

```gradle
implementation 'com.google.android.exoplayer:exoplayer:2.X.X'
```

where `2.X.X` is your preferred version. Alternatively, you can depend on only
the library modules that you actually need. For example the following will add
dependencies on the Core, DASH and UI library modules, as might be required for
an app that plays DASH content:

```gradle
implementation 'com.google.android.exoplayer:exoplayer-core:2.X.X'
implementation 'com.google.android.exoplayer:exoplayer-dash:2.X.X'
implementation 'com.google.android.exoplayer:exoplayer-ui:2.X.X'
```

The available library modules are listed below. Adding a dependency to the full
ExoPlayer library is equivalent to adding dependencies on all of the library
modules individually.

* `exoplayer-core`: Core functionality (required).
* `exoplayer-dash`: Support for DASH content.
* `exoplayer-hls`: Support for HLS content.
* `exoplayer-smoothstreaming`: Support for SmoothStreaming content.
* `exoplayer-ui`: UI components and resources for use with ExoPlayer.

In addition to library modules, ExoPlayer has multiple extension modules that
depend on external libraries to provide additional functionality. These are
beyond the scope of this guide. Browse the [extensions directory][] and their
individual READMEs for details.

### Creating the player ###

You can create an `ExoPlayer` instance using `ExoPlayerFactory`. The factory
provides a range of methods for creating `ExoPlayer` instances with varying
levels of customization. For the vast majority of use cases one of the
`ExoPlayerFactory.newSimpleInstance` methods should be used. These methods
return `SimpleExoPlayer`, which extends `ExoPlayer` to add additional high level
player functionality. The code below is an example of creating a
`SimpleExoPlayer`.

{% highlight java %}
// 1. Create a default TrackSelector
Handler mainHandler = new Handler();
BandwidthMeter bandwidthMeter = new DefaultBandwidthMeter();
TrackSelection.Factory videoTrackSelectionFactory =
    new AdaptiveTrackSelection.Factory(bandwidthMeter);
TrackSelector trackSelector =
    new DefaultTrackSelector(videoTrackSelectionFactory);

// 2. Create the player
SimpleExoPlayer player =
    ExoPlayerFactory.newSimpleInstance(context, trackSelector);
{% endhighlight %}

### Attaching the player to a view ###

The ExoPlayer library provides a `PlayerView`, which encapsulates a
`PlayerControlView` and a `Surface` onto which video is rendered. A `PlayerView`
can be included in your application's layout xml. Binding the player to the view
is as simple as:

{% highlight java %}
// Bind the player to the view.
playerView.setPlayer(player);
{% endhighlight %}

If you require fine-grained control over the player controls and the `Surface`
onto which video is rendered, you can set the player's target `SurfaceView`,
`TextureView`, `SurfaceHolder` or `Surface` directly using `SimpleExoPlayer`'s
`setVideoSurfaceView`, `setVideoTextureView`, `setVideoSurfaceHolder` and
`setVideoSurface` methods respectively. You can use `PlayerControlView` as a
standalone component, or implement your own playback controls that interact
directly with the player. `setTextOutput` and `setId3Output` can be used to
receive caption and ID3 metadata output during playback.

### Preparing the player ###

In ExoPlayer every piece of media is represented by `MediaSource`. To play a
piece of media you must first create a corresponding `MediaSource` and then
pass this object to `ExoPlayer.prepare`. The ExoPlayer library provides
`MediaSource` implementations for DASH (`DashMediaSource`), SmoothStreaming
(`SsMediaSource`), HLS (`HlsMediaSource`) and regular media files
(`ExtractorMediaSource`). These implementations are described in more detail
later in this guide. The following code shows how to prepare the player with a
`MediaSource` suitable for playback of an MP4 file.

{% highlight java %}
// Measures bandwidth during playback. Can be null if not required.
DefaultBandwidthMeter bandwidthMeter = new DefaultBandwidthMeter();
// Produces DataSource instances through which media data is loaded.
DataSource.Factory dataSourceFactory = new DefaultDataSourceFactory(context,
    Util.getUserAgent(context, "yourApplicationName"), bandwidthMeter);
// This is the MediaSource representing the media to be played.
MediaSource videoSource = new ExtractorMediaSource.Factory(dataSourceFactory)
    .createMediaSource(mp4VideoUri);
// Prepare the player with the source.
player.prepare(videoSource);
{% endhighlight %}

### Controlling the player ###

Once the player has been prepared, playback can be controlled by calling methods
on the player. For example `setPlayWhenReady` can be used to start and pause
playback, the various `seekTo` methods can be used to seek within the media,
`setRepeatMode` can be used to control if and how the media is looped, and
`setPlaybackParameters` can be used to adjust the playback speed and pitch.

If the player is bound to a `PlayerView` or `PlayerControlView` then user
interaction with these components will cause corresponding methods on the player
to be invoked.

### Releasing the player ###

It's important to release the player when it's no longer needed, so as to free
up limited resources such as video decoders for use by other applications. This
can be done by calling `ExoPlayer.release`.

## MediaSource ##

In ExoPlayer every piece of media is represented by `MediaSource`. The ExoPlayer
library provides `MediaSource` implementations for DASH (`DashMediaSource`),
SmoothStreaming (`SsMediaSource`), HLS (`HlsMediaSource`) and regular media
files (`ExtractorMediaSource`). Examples of how to instantiate all four can be
found in `PlayerActivity` in the [main demo app][].

{% include infobox.html content="`MediaSource` instances are not designed to be
re-used. If you want to prepare a player more than once with the same piece of
media, use a new instance each time." %}

In addition to the MediaSource implementations described above, the ExoPlayer
library also provides `MergingMediaSource`, `LoopingMediaSource`,
`ConcatenatingMediaSource` and `DynamicConcatenatingMediaSource`. These
`MediaSource` implementations enable more complex playback functionality through
composition. Some of the common use cases are described below. Note that
although the following examples are described in the context of video playback,
they apply equally to audio only playback too, and indeed to the playback of any
supported media type(s).

### Side-loading a subtitle file ###

Given a video file and a separate subtitle file, `MergingMediaSource` can be
used to merge them into a single source for playback.

{% highlight java %}
// Build the video MediaSource.
MediaSource videoSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(videoUri);
// Build the subtitle MediaSource.
Format subtitleFormat = Format.createTextSampleFormat(
    id, // An identifier for the track. May be null.
    MimeTypes.APPLICATION_SUBRIP, // The mime type. Must be set correctly.
    selectionFlags, // Selection flags for the track.
    language); // The subtitle language. May be null.
MediaSource subtitleSource =
    new SingleSampleMediaSource.Factory(...)
        .createMediaSource(subtitleUri, subtitleFormat, C.TIME_UNSET);
// Plays the video with the sideloaded subtitle.
MergingMediaSource mergedSource =
    new MergingMediaSource(videoSource, subtitleSource);
{% endhighlight %}

### Looping a video ###

{% include infobox.html content="To loop indefinitely, it is usually better to
use `ExoPlayer.setRepeatMode` instead of `LoopingMediaSource`." %}

A video can be seamlessly looped a fixed number of times using a
`LoopingMediaSource`. The following example plays a video twice.

{% highlight java %}
MediaSource source =
    new ExtractorMediaSource.Factory(...).createMediaSource(videoUri);
// Plays the video twice.
LoopingMediaSource loopingSource = new LoopingMediaSource(source, 2);
{% endhighlight %}

### Playing a sequence of videos ###

`ConcatenatingMediaSource` enables sequential playback of two or more individual
`MediaSource`s. The following example plays two videos in sequence. Transitions
between sources are seamless. There is no requirement that the sources being
concatenated are of the same format (e.g., it’s fine to concatenate a video file
containing 480p H264 with one that contains 720p VP9). The sources may even be
of different types (e.g., it’s fine to concatenate a video with an audio only
stream).

{% highlight java %}
MediaSource firstSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(firstVideoUri);
MediaSource secondSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(secondVideoUri);
// Plays the first video, then the second video.
ConcatenatingMediaSource concatenatedSource =
    new ConcatenatingMediaSource(firstSource, secondSource);
{% endhighlight %}

`DynamicConcatenatingMediaSource` is similar to `ConcatenatingMediaSource`,
except that it allows `MediaSource`s to be dynamically added, removed and moved
both before and during playback. `DynamicConcatenatingMediaSource` is
well suited to playlist use cases where the user is able to modify the playlist
during playback.

{% include infobox.html content="A `MediaSource` instance should not be added
more than once to a `DynamicConcatenatingMediaSource`, or be re-added having
previously been removed. Use a new instance instead." %}

### Advanced composition ###

It’s possible to further combine composite `MediaSource`s for more unusual use
cases. Given two videos A and B, the following example shows how
`LoopingMediaSource` and `ConcatenatingMediaSource` can be used together to play
the sequence (A,A,B).

{% highlight java %}
MediaSource firstSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(firstVideoUri);
MediaSource secondSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(secondVideoUri);
// Plays the first video twice.
LoopingMediaSource firstSourceTwice = new LoopingMediaSource(firstSource, 2);
// Plays the first video twice, then the second video.
ConcatenatingMediaSource concatenatedSource =
    new ConcatenatingMediaSource(firstSourceTwice, secondSource);
{% endhighlight %}

The following example is equivalent, demonstrating that there can be more than
one way of achieving the same result.

{% highlight java %}
MediaSource firstSource =
    new ExtractorMediaSource.Builder(firstVideoUri, ...).build();
MediaSource secondSource =
    new ExtractorMediaSource.Builder(secondVideoUri, ...).build();
// Plays the first video twice, then the second video.
ConcatenatingMediaSource concatenatedSource =
    new ConcatenatingMediaSource(firstSource, firstSource, secondSource);
{% endhighlight %}

{% include infobox.html content="It is important to avoid using the same
`MediaSource` instance multiple times in a composition, unless explicitly
allowed according to the documentation. The use of firstSource twice in the
example above is one such case, since the Javadoc for `ConcatenatingMediaSource`
explicitly states that duplicate entries are allowed. In general, however, the
graph of objects formed by a composition should be a tree. Using multiple
equivalent `MediaSource` instances in a composition is allowed." %}

## Player events ##

During playback, your app can listen for events generated by ExoPlayer that
indicate the overall state of the player. These events are useful as triggers
for updating user interface components such as playback controls. Many ExoPlayer
components also report their own component specific low level events, which can
be useful for performance monitoring.

### High level events ###

ExoPlayer allows `EventListener`s to be added and removed by calling the
`addListener` and `removeListener` methods. Registered listeners are notified of
changes in playback state, as well as when errors occur that cause playback to
fail.

Developers who implement custom playback controls should register a listener and
use it to update their controls as the player’s state changes. An app should
also show an appropriate error to the user if playback fails.

When using `SimpleExoPlayer`, additional listeners can be set on the player. In
particular `addVideoListener` allows an application to receive events related to
video rendering that may be useful for adjusting the UI (e.g., the aspect ratio
of the `Surface` onto which video is being rendered). Listeners can also be set
to receive debugging information, for example by calling `setVideoDebugListener`
and `setAudioDebugListener`.

### Low level events ###

In addition to high level listeners, many of the individual components provided
by the ExoPlayer library allow their own event listeners. You are typically
required to pass a `Handler` object to such components, which determines the
thread on which the listener's methods are invoked. In most cases, you should
use a `Handler` associated with the app’s main thread.

## Sending messages to components ##

It's possible to send messages to ExoPlayer components. These can be created
using `createMessage` and then sent using `PlayerMessage.send`. By default,
messages are delivered on the playback thread as soon as possible, but this can
be customized by setting another callback thread (using
`PlayerMessage.setHandler`) or by specifying a delivery playback position
(using `PlayerMessage.setPosition`). Sending messages through the `ExoPlayer`
ensures that the operation is executed in order with any other operations being
performed on the player.

Most of ExoPlayer's out-of-the-box renderers support messages that allow
changes to their configuration during playback. For example, the audio
renderers accept messages to set the volume and the video renderers accept
messages to set the surface. These messages should be delivered
on the playback thread to ensure thread safety.

## Customization ##

One of the main benefits of ExoPlayer over Android's `MediaPlayer` is the
ability to customize and extend the player to better suit the developer’s use
case. The ExoPlayer library is designed specifically with this in mind, defining
a number of interfaces and abstract base classes that make it possible for app
developers to easily replace the default implementations provided by the
library. Here are some use cases for building custom components:

* `Renderer` &ndash; You may want to implement a custom `Renderer` to handle a
  media type not supported by the default implementations provided by the
  library.
* `TrackSelector` &ndash; Implementing a custom `TrackSelector` allows an app
  developer to change the way in which tracks exposed by a `MediaSource` are
  selected for consumption by each of the available `Renderer`s.
* `LoadControl` &ndash; Implementing a custom `LoadControl` allows an app
  developer to change the player's buffering policy.
* `Extractor` &ndash; If you need to support a container format not currently
  supported by the library, consider implementing a custom `Extractor` class,
  which can then be used to together with `ExtractorMediaSource` to play media
  of that type.
* `MediaSource` &ndash; Implementing a custom `MediaSource` class may be
  appropriate if you wish to obtain media samples to feed to renderers in a
  custom way, or if you wish to implement custom `MediaSource` compositing
  behavior.
* `DataSource` &ndash; ExoPlayer’s upstream package already contains a number of
  `DataSource` implementations for different use cases. You may want to
  implement you own `DataSource` class to load data in another way, such as over
  a custom protocol, using a custom HTTP stack, or from a custom persistent
  cache.

### Customization guidelines ###

* If a custom component needs to report events back to the app, we recommend
  that you do so using the same model as existing ExoPlayer components, where an
  event listener is passed together with a `Handler` to the constructor of the
  component.
* We recommended that custom components use the same model as existing ExoPlayer
  components to allow reconfiguration by the app during playback, as described
  in [Sending messages to components](#sending-messages-to-components). To do
  this, you should implement an `ExoPlayerComponent` and receive configuration
  changes in its `handleMessage` method. Your app should pass configuration
  changes by calling ExoPlayer’s `sendMessages` and `blockingSendMessages`
  methods.

## Digital Rights Management ##

On Android 4.4 (API level 19) and higher, ExoPlayer supports Digital Rights
Management (DRM) protected playback. In order to play DRM protected content with
ExoPlayer, your app must inject a `DrmSessionManager` when instantiating the
player. `ExoPlayerFactory` provides factory methods allowing this. A
`DrmSessionManager` object is responsible for providing `DrmSession` instances,
which provide `MediaCrypto` objects for decryption as well as ensuring that the
required decryption keys are available to the underlying DRM module being used.

The ExoPlayer library provides a default implementation of `DrmSessionManager`,
called `DefaultDrmSessionManager`, which uses `MediaDrm`. The session manager
supports any DRM scheme for which a modular DRM component exists on the device.
All Android devices are required to support Widevine modular DRM (with L3
security, although many devices also support L1). Some devices may support
additional schemes such as PlayReady. All Android TV devices support PlayReady.

`PlayerActivity` in the [main demo app][] demonstrates how a
`DefaultDrmSessionManager` can be created and injected when instantiating the
player.

[Supported formats]: https://google.github.io/ExoPlayer/supported-formats.html
[IMA extension]: https://github.com/google/ExoPlayer/tree/release-v2/extensions/ima
[Interactive Media Ads SDK]: https://developers.google.com/interactive-media-ads
[ExoPlayer library]: https://github.com/google/ExoPlayer/tree/release-v2/library
[main demo app]: https://github.com/google/ExoPlayer/tree/release-v2/demos/main
[`MediaPlayer`]: {{ site.sdkurl }}/android/media/MediaPlayer.html
[`MediaCodec`]: {{ site.sdkurl }}/android/media/MediaCodec.html
[`AudioTrack`]: {{ site.sdkurl }}/android/media/AudioTrack.html
[`MediaDrm`]: {{ site.sdkurl }}/android/media/MediaDrm.html
[extensions directory]: https://github.com/google/ExoPlayer/tree/release-v2/extensions/