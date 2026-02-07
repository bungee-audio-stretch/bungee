# Bungee: Audio Time-Stretching & Pitch-Shifting Library

Bungee is a modern, open-source C++ library for high-quality audio time-stretching and pitch-shifting in real-time or offline. It brings powerful audio timescale processing to your application.

Bungee can adjust the speed of audio without affecting pitch; transpose audio pitch without affecting speed; or apply any combination of playhead position and pitch manipulation.
* Simple, fast phase-vocoder-based algorithm with good quality audio output (🎧  hear [some comparisons](https://bungee.parabolaresearch.com/compare-audio-stretch-tempo-pitch-change.html) with other algorithms)
* Modern C++ for clean and resilient code

Bungee is unique in its controllability, allowing continually changing audio tempo and pitch manipulation with seamless support of zero and negative playback speeds. So it can be used for a "smooth scrub" or for rendering lifelike audio for slow-motion videos.

Bungee is often used for slowing down music or speeding up speech without affecting pitch. It is also popular in music software for changing tempo, transposing and other effects.

 ⭐️ _To support Bungee, please consider [giving this repo a star](https://github.com/bungee-audio-stretch/bungee/stargazers)_ .

![GitHub Release](https://img.shields.io/github/v/release/bungee-audio-stretch/bungee)
![GitHub License](https://img.shields.io/github/license/bungee-audio-stretch/bungee)
![GitHub Downloads](https://img.shields.io/github/downloads/bungee-audio-stretch/bungee/total)
![GitHub Repo stars](https://img.shields.io/github/stars/bungee-audio-stretch/bungee)

## Getting started with Bungee Audio Time-Stretching

### Clone and Build

Bungee's dependencies are managed as git submodules; so clone like this:
```
git clone --recurse-submodules https://github.com/bungee-audio-stretch/bungee
```

Use CMake to configure and build the bungee library and command-line executable:
```
cd bungee
mkdir build && cd build
cmake ..
cmake --build .
```

After a successful build, try the bungee executable
```
./bungee --help
```
### Pre-built Releases

Every commit pushed to this repo's main branch is automatically tagged and built into a release. Each release contains Bungee built as a shared library together with headers, sample code and a sample command-line executable that uses the shared library. Releases support common platforms including Linux, Windows, MacOS, Android and iOS.


## Integrating Bungee C++ Audio Library

Bungee operates on discrete, overlapping "grains" of audio, typically processing around 100 grains per second. Parameters such as position and pitch are provided on a per-grain basis so that they can be changed continuously as audio rendering progresses.

The Bungee API supports two modes of operation: "granular" and "streaming". Both modes offer identical audio quality and similar CPU performance. Developers should select the mode that best suits the needs of their application.

| Mode | Granular | Streaming |
|--|--|--|
| `#include` | `<bungee/Bungee.h>` | `<bungee/Stream.h>` |
| Description | Fundamental, low-level interface. | Simplified API built atop the granular interface. |
| Operation | The caller iterates over a "granular loop" where they control exactly what happens at each of the Bungee grains. | The granular nature of the underlying library is hidden and the caller instead manages an input audio stream and an output audio stream. Iterations are over "chunks" of audio that need not map to the underlying grains. |
| Benefits | Allows any arbritrary movements within the input audio: play at any speed, reverse play, infinite stretch, trick play. Lowest latency and no additional data copies | Simpler "FIFO" operation, more similar to traditional "linear playback" audio processing code. Designed to be easy to integrate to frameworks such as JUCE and FFmpeg.  |
| Restrictions | | Supports only forward playback.|
| Example | [cmd/main.cpp](./cmd/main.cpp) with `--push 0` | [cmd/main.cpp](./cmd/main.cpp)  when `--push` has a non-zero value |


### Granular Audio Time-Stretching and Pitch-Shifting Example

#### Instantiation for Granular Processing

``` C++
#include <bungee/Bungee.h>
```
``` C++
// Define stretcher input and output sample rates
const Bungee::SampleRates sampleRates{44100, 44100};

// Instantiate a stretcher for two-channel (stereo) operation.
Bungee::Stretcher<Bungee::Basic> stretcher(sampleRates, 2);

// Enable instrumentation. When done debugging, this can be removed.
stretcher.enableInstrumentation(true);

Bungee::Request request{};

// Set pitch, this example shows an upward transposition of one semitone.
request.pitch = std::pow(2., 1. / 12.);

// Set initial speed, this example shows how to achieve constant 75% output speed.
request.speed = 0.75;

// Set initial starting position at 0.5 seconds offset from the start of the input buffer.
request.position = 0.5 * sampleRate;

// This call adjusts request.position so that stretcher's pipeline will be fully initialised by the
// time it reaches the starting position of 0.5 seconds offset.
stretcher.preroll(request);
```

#### Granular Loop

`Stretcher`'s processing functions are typically called from within a loop, each iteration of which corresponds to a grain of audio. For each grain, the functions `Stretcher<Basic>::specifyGrain`, `Stretcher<Basic>::analyseGain` and `Stretcher<Basic>::synthesiseGrain` should be called in sequence.
```C++
while (true)
{
    // ...
    // Change request's members, for example, position, speed or pitch, as required here.
    // ...
 
    auto inputChunk = stretcher.specifyGrain(request);

    // ...
    // Examine inputChunk and retrieve the segment of input audio that the stretcher requires here.
    // Set data and channelStride to point to the required segment of input data.
    // ...

    stretcher.analyseGrain(data, channelStride);

    Bungee::OutputChunk outputChunk;
    stretcher.synthesiseGrain(outputChunk);

    // ...
    // Output the audio buffer indicated by outputChunk here.
    // ...

    // Prepare request for the next grain (modifies request.position according to request.speed)
    stretcher.next(request);
}
```

#### Granular Notes

* `Request::position` is a timestamp, it defines the grain centre point in terms of an input audio frame offset. It is the primary control for speed adjustments and is also the driver for seek and scrub operations. The caller is responsible for deciding  `Request::position` for each grain. 

* The caller owns the input audio buffer and must provide the audio segment indicated by `InputChunk`. Successive grains' input audio chunks may overlap. The `Stretcher<Basic>` instance reads in the input chunk data when `Stretcher<Basic>::analyseGrain` is called.

* The `Stretcher<Basic>` instance owns the output audio buffer. It is valid from when `Stretcher<Basic>::synthesiseGrain` returns up until `Stretcher<Basic>::synthesiseGrain` is called for the subsequent grain. Output audio chunks do not overlap: they should be concatenated to produce an output audio stream.

* Output audio is timestamped. The original `Request` objects corresponding to the start and end of the chunk are provided by `OutputChunk`.

### Streaming Audio Time-Stretching and Pitch-Shifting Example

#### Streaming Instantiation

``` C++
#include <bungee/Stream.h>
```

``` C++
// Define stretcher input and output sample rates
const Bungee::SampleRates sampleRates{44100, 44100};
const auto channelCount = 2;

// What is the maximum number of input 
const auto maxInputFrameCount = 1024; // for example

// Instantiate a stretcher.
Bungee::Stretcher<Bungee::Basic> stretcher(sampleRates, channelCount);

// Enable instrumentation. When done debugging, this can be removed.
stretcher.enableInstrumentation(true);

// Instantiate a Stream object.
Bungee::Stream<Bungee::Basic> stream(stretcher, maxInputFrameCount, channelCount);
```

#### Streaming Loop

```C++
while (true)
{
    // Each iteration of this loop processes the next contiguous chunk of audio data.

    std::vector<const float *> inputChannelPointers(channelCount);
    int inputSampleCount;

    std::vector<float *> outputChannelPointers(channelCount);

    // ...
    // Insert code here to: 
    // 1. Receive the next chunk of input audio, set inputChannelPointers and inputSampleCount accordingly
    // 2. Set outputChannelPointers to a user maintained buffer where output audio should be written by Bungee.
    // ...

    // Set these control variables as desired.
    const double speed=1., pitch=1.;

    const double outputFrameCountIdeal = (inputSampleCount * sampleRates.output) / (speed * sampleRates.input);

    // This call does the stretching.
    const auto outputFrameCountActual = stream.process(inputChannelPointers.data(), outputChannelPointers.data(), inputSampleCount, outputFrameCountIdeal, pitch);

    // ...
    // Place code to handle output of the time-stretched / pitch-shifted audio.
    // The processed audio resides in memory at the buffer indicated by outputChannelPointers 
    // and each channel will have outputFrameCountActual samples of new audio.
    // ...
}
```
### Streaming Notes
* The caller specifies its desired output audio speed (tempo) by the value of `outputFrameCountIdeal` above. Because the stretcher cannot output a fractional number of audio frames, `outputFrameCountActual` will be returned as one of the two closest integral values to `outputFrameCountIdeal`.


## General Bungee C++ API Notes

* In Bungee code and API, a "frame" refers to a single time-aligned set of audio samples—one sample from each channel at a specific point in time. For example, in stereo audio, a frame consists of one left and one right sample occurring simultaneously.

* Bungee is most commonly used for stereo and mono operation at sample rates of 44.1kHz and 48kHz. In principle, though, any practical sample rate and number of audio channels are supported and results should be similar.

* Bungee works with 32-bit floating point audio samples and expects sample values in the range -1 to +1 on both input and output. The algorithm performs no clipping.

* When configured for 1x speed and no pitch adjustment, the difference between input and output signals should be small: minor windowing discrepencies only.

* Any special or non-numeric float values such as NaN or infinity within the input audio may disrupt or cause loss of output audio.

* It is strongly recommended to enable Bungee's internal instrumentation whem working on the integration of the Bungee API. The instrumentation is particuarly helpful for the granular mode of operation because it can detect common usage errors.

## Bungee's Dependencies

The Bungee library gratefully depends on:
* The Eigen C++ library for buffer management and mathematical operations on vectors and arrays
* The PFFFT library for Fast Fourier Transforms

The sample `bungee` command-line utility also uses:
* cxxopts library for parsing command-line options

See this repo's [.gitmodules](.gitmodules) file for versioned links to these projects.

## Bungee License

Bungee is permissively licensed under the Mozilla Public License Version 2.0. 

## Bungee Support

Bungee's goal is to be the _best open source audio timescale manipulation library_ available. User feedback is invaluable: please use Github issues or contact the team directly to report anything that could be improved.

> # Bungee Pro
> 
> Bungee Pro is a commercial product providing an upgrade path from the open-source Bungee audio time stretcher. It uses proprietary new algorithms for sharp and clear professional-grade audio and runs at least as fast as Bungee, thanks to platform-specific performance optimisations.
> 
> Whilst open-source Bungee aims to be the best open-source audio time-stretch algorithm, the goal of Bungee Pro is to be the _best commercially-available audio time-stretch and pitch-adjustment technology_.
> 
> * Novel processing techniques that deliver crisp transients and preserve vocal and instrumental timbre
> * Adaptive to all genres of speech, music and sound with subjective transparency up to infinite time stretch
> * Performance optimisations for:
>    * Web AudioWorklet with SIMD128 WebAssembly
>    * Arm NEON for Android, iOS and MacOS
>    * x86-64 SIMD for Linux, Windows and MacOS
> * A ready-to-use Web Audio implementation 
> * Professional support
>
> Bungee Pro is today deployed in a wide variety of applications including movie post production software, educational apps and popular musicians' tools. 
>
> Check out an [extensive evaluation](https://bungee.parabolaresearch.com/compare-audio-stretch-tempo-pitch-change.html) of Bungee and Bungee Pro against state-of-the-art techniques.
>
> | ![Bungee waveform animation: audio time-stretching and pitch-shifting demo](./README.md-waveform.png) | Try the [WebAssembly demo](https://bungee.parabolaresearch.com/change-audio-speed-pitch.html)  of Bungee Pro in your browser.|
> |--|--|
>
