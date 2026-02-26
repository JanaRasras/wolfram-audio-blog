
# Visualizing Sound: Audio Analysis with Wolfram Language

*What does a bird chirp look like? How is a piano note different from a drum beat when you see them? In this post, we'll explore how Wolfram Language can turn sound into vivid, interactive visualizations — revealing structure in audio that your ears alone can't perceive.*

---

## Sound as Data

Before we write a single line of code, let's think about what audio actually is.

When you hear a sound, what's really happening is air pressure fluctuating rapidly around you. A microphone captures those fluctuations and converts them into a long sequence of numbers — one number per moment in time, each representing the pressure at that instant. This sequence is called a **digital audio signal**.

Two properties define any digital audio signal:

- **Sample rate** — how many measurements are taken per second. CD quality audio uses 44,100 samples per second (44,100 Hz).
- **Amplitude** — the value of each measurement, representing the loudness of the signal at that moment.

So at its core, audio is just a very long list of numbers. Everything in this post — waveforms, spectrograms, frequency spectra — is simply different ways of looking at that list.

---

## Loading Audio

Wolfram Language offers three ways to get audio into your notebook.

**From a file on disk:**

```mathematica
audio = Import["example.wav"]
```

**From the built-in example library** (no files needed):

```mathematica
audio = ExampleData[{"Audio", "Bird"}]
```

**Directly from your microphone:**

```mathematica
audio = AudioCapture[]
```

The built-in library includes dozens of samples — birds, piano, drums, speech, even historical recordings like Neil Armstrong's famous words from the Moon. To see the full list:

```mathematica
ExampleData["Audio"]
```

For the rest of this post, we'll use `ExampleData[{"Audio", "Bird"}]` so you can follow along without needing any audio files.

---

## Inspecting the Signal

Before visualizing, let's get a feel for what we're working with. `AudioMeasurements` gives us the key properties of the signal in one call:

```mathematica
audio = ExampleData[{"Audio", "Bird"}];
AudioMeasurements[audio, {"Duration", "SampleRate", "RMSAmplitude", "Power"}]
```

We can also query properties individually:

```mathematica
(* How long is the audio? *)
Duration[audio]

(* How many samples per second? *)
AudioSampleRate[audio]

(* Mono or stereo? *)
AudioChannels[audio]
```

**RMS Amplitude** (Root Mean Square Amplitude) is the standard measure of average loudness — it's the square root of the average of the squared sample values, giving a single number that represents how loud the audio is overall.

---

## The Waveform: Amplitude Over Time

The simplest way to visualize audio is the **waveform** — a plot of amplitude (y-axis) against time (x-axis). It answers the question: *how does the loudness of the signal change over time?*

`AudioPlot` produces a waveform with a single function call:

```mathematica
AudioPlot[audio]
```

Wolfram Language automatically handles stereo channels, axis labels, and layout. If you see two plots stacked on top of each other, that means your audio is **stereo** — one plot per channel (left ear, right ear). To combine them into a single mono waveform:

```mathematica
AudioPlot[AudioChannelMix[audio, "Mono"]]
```

Let's customize the plot with a color and axis labels:

```mathematica
AudioPlot[audio,
  PlotStyle  -> ColorData[97, 1],
  PlotLabel  -> "Waveform — Bird",
  AxesLabel  -> {"Time (s)", "Amplitude"},
  ImageSize  -> Large,
  AspectRatio -> 1/4
]
```

The `AspectRatio -> 1/4` makes the plot wide and flat — a natural shape for time-series audio data. To zoom into a specific time range, use `AudioTrim`:

```mathematica
AudioPlot[AudioTrim[audio, {1.0, 3.0}],
  PlotLabel  -> "Waveform — Bird (1s to 3s)",
  AxesLabel  -> {"Time (s)", "Amplitude"},
  ImageSize  -> Large
]
```

`AudioTrim[audio, {1.0, 3.0}]` cuts the audio to the segment between second 1 and second 3 — like cropping an image, but for sound.

---

## The Spectrogram: Frequency Over Time

The waveform tells us *when* loud things happen, but not *what kind* of sound is happening. A waveform can't distinguish a flute from a drum — both might have the same amplitude at a given moment.

The **spectrogram** solves this. It breaks the audio into short overlapping windows and computes the frequency content of each window using the **Fast Fourier Transform (FFT)**. The result is a 2D heatmap:

- **X-axis** — time (seconds)
- **Y-axis** — frequency (Hz), from low (bass) at the bottom to high (treble) at the top
- **Color** — intensity (how loud that frequency is at that moment)

```mathematica
Spectrogram[audio,
  ColorFunction -> "SunsetColors",
  PlotLabel     -> "Spectrogram — Bird",
  FrameLabel    -> {"Time (s)", "Frequency (Hz)"},
  ImageSize     -> Large,
  AspectRatio   -> 1/3
]
```

For a bird chirp, you'll see bright diagonal sweeps at high frequencies — because birds slide up and down in pitch as they chirp. For a piano note, you'll see bright horizontal stripes — a sustained frequency held over time.

### The Time-Frequency Trade-off

The spectrogram has one important parameter: the **window size**. This controls how many samples are analyzed at once to compute each frequency slice.

```mathematica
(* Short window — sharp in time, blurry in frequency *)
Spectrogram[audio, 256,
  PlotLabel  -> "Short Window — 256 Samples",
  FrameLabel -> {"Time (s)", "Frequency (Hz)"}
]

(* Long window — blurry in time, sharp in frequency *)
Spectrogram[audio, 4096,
  PlotLabel  -> "Long Window — 4096 Samples",
  FrameLabel -> {"Time (s)", "Frequency (Hz)"}
]
```

Think of it like a camera shutter: a fast shutter freezes motion but reduces exposure; a slow shutter blurs motion but captures more light. In audio, this fundamental limitation is known as the **time-frequency uncertainty principle** — you cannot have perfect resolution in both dimensions simultaneously.

For rhythmic content like drums, a short window (256) is better. For tonal content like piano or voice, a longer window (1024–4096) reveals the pitch structure more clearly.

---

## The Periodogram: The Overall Frequency Profile

While the spectrogram shows frequency content changing over time, the **periodogram** collapses the time axis entirely and shows the *overall* frequency profile of the signal — which frequencies are present on average, across the whole audio clip.

```mathematica
Periodogram[AudioData[audio][[1]],
  ScalingFunctions -> "dB",
  PlotLabel        -> "Frequency Spectrum — Bird",
  AxesLabel        -> {"Frequency (Hz)", "Power (dB)"},
  PlotRange        -> All,
  ImageSize        -> Large
]
```

Two important things to note:

1. `AudioData[audio][[1]]` extracts the raw numeric samples from the first channel — `Periodogram` requires numeric data, not an `Audio` object.
2. `ScalingFunctions -> "dB"` plots power in **decibels** (a logarithmic scale), which better matches how humans perceive loudness — our ears are logarithmic too.

The periodogram is most useful for **comparing audio sources**. Let's look at a bird and a piano side by side:

```mathematica
bird  = ExampleData[{"Audio", "Bird"}];
piano = ExampleData[{"Audio", "Piano"}];
```

```mathematica
GraphicsRow[{
  Periodogram[AudioData[bird][[1]],  PlotLabel -> "Bird",  AxesLabel -> {"Frequency (Hz)", "Power"}],
  Periodogram[AudioData[piano][[1]], PlotLabel -> "Piano", AxesLabel -> {"Frequency (Hz)", "Power"}]
}, ImageSize -> Large]
```

The bird concentrates energy in high frequencies (sharp, chirpy sounds). The piano shows distinct spikes at the fundamental frequency and its harmonics — multiples of the base note that give the piano its characteristic rich tone.

> **Note:** `Periodogram` requires raw numeric samples, not an `Audio` object. Use `AudioData[audio][[1]]` to extract the first channel's samples before passing to `Periodogram`.

---

## Putting It All Together

Now let's combine everything into a complete audio analysis notebook. Run each cell in order with `Shift + Enter`:

```mathematica
(* Cell 1 — Load audio *)
audio = ExampleData[{"Audio", "Bird"}];
```

```mathematica
(* Cell 2 — Audio metadata *)
AudioMeasurements[audio, {"Duration", "SampleRate", "RMSAmplitude", "Power"}]
```

```mathematica
(* Cell 3 — Waveform *)
AudioPlot[audio,
  PlotStyle   -> RGBColor[0.2, 0.6, 0.9],
  PlotLabel   -> Style["Waveform", Bold, 14],
  AxesLabel   -> {"Time (s)", "Amplitude"},
  ImageSize   -> 500,
  AspectRatio -> 1/4
]
```

```mathematica
(* Cell 4 — Spectrogram *)
Spectrogram[audio,
  ColorFunction -> "SunsetColors",
  PlotLabel     -> Style["Spectrogram", Bold, 14],
  FrameLabel    -> {"Time (s)", "Frequency (Hz)"},
  ImageSize     -> 500,
  AspectRatio   -> 1/3
]
```

```mathematica
(* Cell 5 — Frequency spectrum *)
Periodogram[AudioData[audio][[1]],
  ScalingFunctions -> "dB",
  PlotLabel        -> Style["Frequency Spectrum", Bold, 14],
  AxesLabel        -> {"Frequency (Hz)", "Power (dB)"},
  ImageSize        -> 500,
  AspectRatio      -> 1/3
]
```

Each cell produces one clean output. This approach works reliably in both Wolfram Cloud and desktop Mathematica.

---

## An Interactive Explorer

One of Wolfram Language's most distinctive features is `Manipulate` — a function that instantly turns any computation into an interactive UI. Let's wrap our visualizer in `Manipulate` to create a live audio explorer with dropdowns and sliders:

```mathematica
Manipulate[
  Module[{trimmed},
    trimmed = AudioTrim[audio, {tStart, tEnd}];
    Column[{
      AudioPlot[trimmed,
        PlotLabel  -> "Waveform",
        AxesLabel  -> {"Time (s)", "Amplitude"},
        ImageSize  -> 500
      ],
      Spectrogram[trimmed, windowSize,
        ColorFunction -> colorScheme,
        PlotLabel     -> "Spectrogram",
        FrameLabel    -> {"Time (s)", "Frequency (Hz)"},
        ImageSize     -> 500
      ],
      Periodogram[AudioData[trimmed][[1]],
        ScalingFunctions -> "dB",
        PlotLabel        -> "Frequency Spectrum",
        AxesLabel        -> {"Frequency (Hz)", "Power (dB)"},
        ImageSize        -> 500
      ]
    }]
  ],
  {{audio, ExampleData[{"Audio", "Bird"}], "Audio Source"},
    {ExampleData[{"Audio", "Bird"}]  -> "Bird",
     ExampleData[{"Audio", "Piano"}] -> "Piano",
     ExampleData[{"Audio", "Drums"}] -> "Drums"}},
  {{tStart, 0,    "Start (s)"}, 0,   5,  0.1},
  {{tEnd,   2,    "End (s)"},   0.5, 10, 0.1},
  {{windowSize, 1024, "Window Size"},
    {256 -> "256 (Time)", 1024 -> "1024 (Balanced)", 4096 -> "4096 (Frequency)"}},
  {{colorScheme, "SunsetColors", "Color Scheme"},
    {"SunsetColors", "TemperatureMap", "Rainbow", "GreenPinkTones"}}
]
```

Every time you move a slider or change a dropdown, all three plots update instantly. You can:

- Switch between a bird, a piano, and a drum beat
- Zoom into any time window of the audio
- Compare how different window sizes change the spectrogram resolution
- Pick a color scheme for the spectrogram

This kind of interactivity — which would require a full web application in most languages — is a single function call in Wolfram Language.

---

## Summary

Here is everything we used in this project:

| Visualization | Function | What it shows |
|--------------|----------|--------------|
| Waveform | `AudioPlot` | Amplitude over time |
| Spectrogram | `Spectrogram` | Frequency content over time |
| Frequency spectrum | `Periodogram` | Overall frequency profile |
| Audio metadata | `AudioMeasurements` | Duration, sample rate, loudness |
| Interactive UI | `Manipulate` | Live explorer with controls |

The key insight is that all three visualizations look at the same underlying data — a list of numbers — from different angles. The waveform is the time domain view. The periodogram is the frequency domain view. The spectrogram is both at once, at the cost of some resolution in each.

---


