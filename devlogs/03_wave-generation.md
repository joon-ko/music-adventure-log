# \[03\] wave generation

In the first entry, we covered how to write a sine wave directly to an AudioStreamGenerator. Now I want to write the sine wave to a generic target node. Instead of writing a variable number of frames based on the space available in the buffer, we can write to the target audio node's input buffer, which has a fixed block size:

```C#
private override void Process()
{
    if (!_graph.IsAddressInUse(audioOutputAddress))
    {
        return;
    }
    var targetIndex = _graph.GetOppositeAddress(audioOutputAddress).index;
    var outputNode = _graph.GetTargetAudioNodeFromSourceAddress(audioOutputAddress);
    var targetInputBuffer = outputNode.inputBuffers[targetIndex];
    _GenerateSineWave(targetInputBuffer);
}

private void _GenerateSineWave(float[] targetInputBuffer)
{
    for (int i=0; i < AudioConstants.BLOCK_SIZE; i++)
    {
        float increment = Frequency / AudioConstants.MIX_RATE;
        float baseValue = Mathf.Sin(_phase * Mathf.Tau);
        targetInputBuffer[i] = SINE_NORMALIZATION_FACTOR * baseValue;
        _phase = Mathf.PosMod(_phase + increment, 1.0f);
    }
}
```

Some notes:
- IOAddresses will come up from time to time in the code. This is the best idea I have for now on how to uniquely identify every IO port on the screen. Addresses are data structs with a nodeId, IO family, IO type, and index. Examples of IO families are things like Audio, Midi, Trigger, and Data. The IO type is simply whether it's an input or output. The index allows one to distinguish between two of the same family-type combination in the same module. For example, mixers would have more than one audio input, so they need to be differentiated by index. It is actually the IOAddresses that serve as the vertices of the audio graph, not the modules!
- I've augmented the spec for AudioNode to allow for multiple input buffers. This will be useful for something like a mixer that takes in multiple audio inputs. 
- The sine normalization factor is a hardcoded factor that tries to keep the perceived loudness of the audio signal at a normalized level -- I'm aiming for -6.0 dB. This hardcoded value can be improved with a more dynamic normalization in the future, since different frequencies probably have different energy levels and would need different normalization factors.

If we connect a sine wave module to a speaker module, we should be able to make our first sounds in our system!

https://github.com/user-attachments/assets/bbd0e5eb-2aa4-4aba-af8c-334c6e05259b

I've been going with a pixel art aesthetic because it is relatively easy to iterate with and prototype, brings a unique video-game-like feel to a music system, and brings me comfort.

I'll omit the minutiae of maintaining the state of connecting and removing wires, as it's not too interesting. The backing state is the dictionary of connections in the audio graph. For drawing the slight curve in the wires to simulate gravity, I used Bezier curves. I want the wires to jiggle a bit eventually and feel more physical! I also want to add an option to represent the wires as straight-line connections if one so wishes. In general, I want the aesthetics of everything to be very customizable. 

Now let's move on to other wave types. In a sense, sine waves are the fundamental wave type that is a building block for all other periodic waves: [Fourier series](https://en.wikipedia.org/wiki/Fourier_series) are what back this claim. At a high level, any periodic waveform can be modeled as a combination of sine waves at a specific combination of amplitude-frequency-phase tuples. The frequencies are multiples of a *fundamental frequency* -- these multiples are also called *harmonics* in an audio/music context. The fundamental frequency corresponds to the perceived pitch of a wave. The secret sauce that gives the "recipe" of amplitude-frequency-phase combinations for a time-series waveform is the [Fourier transform](https://en.wikipedia.org/wiki/Fourier_transform).

We will definitely explore Fourier transforms, especially the [discrete-time Fourier transform](https://en.wikipedia.org/wiki/Discrete-time_Fourier_transform), when we start manipulating audio in more complex ways. For now, though, it is sufficient to just look up the Fourier series for the common wave types and go from there.

The [triangle wave](https://en.wikipedia.org/wiki/Triangle_wave) has the following Fourier series:

$$x_{\mathrm{triangle}}(t) = -\frac{8}{\pi^2} \sum_{k=1}^{\infty} \frac{(-1)^k}{(2k-1)^2} \sin (2\pi (2k - 1)t)$$

In code:

```C#
private void _GenerateTriangleWave(float[] targetInputBuffer)
{
    for (int i=0; i < AudioConstants.BLOCK_SIZE; i++)
    {
        float increment = Frequency / AudioConstants.MIX_RATE;
        float sumOfTerms = 0f;
        for (int k=0; k < NUM_TRIANGLE_HARMONICS; k++)
        {
            float n = 2*k + 1;
            float sign = k % 2 == 0 ? 1 : -1;
            sumOfTerms += sign / (n * n) * Mathf.Sin(n * _phase * Mathf.Tau);
        }

        float baseValue = 8f / (Mathf.Pi * Mathf.Pi) * sumOfTerms;
        targetInputBuffer[i] = TRIANGLE_NORMALIZATION_FACTOR * baseValue;
        _phase = Mathf.PosMod(_phase + increment, 1.0f);
    }
}
```

Here's what that sounds like! To help see the audio signals we generate, I've created a module called the *wave visualizer* that functions as an *audio passthrough*: when it's processed, it simply copies its input to its output. Visually, it draws the data in its input buffer on the screen.

https://github.com/user-attachments/assets/c984df54-ba90-4490-b2f0-57cb61f0da2d

By the way, `NUM_TRIANGLE_HARMONICS` is the number of harmonics we want to include. A Fourier series expansion is technically infinite, so we have to stop somewhere. The more harmonics we include, the more accurate the Fourier series approximattes the actual function. Perhaps due to only using odd-number harmonics, triangle waves actually converge pretty quickly, as shown in this Wikipedia animation:

![](../images/synthesis_triangle.gif)

On the other hand, square waves feel like they take longer to converge:

![](../images/synthesis_square.gif)

There is usually this unwanted overshooting of the signal at the transition points, which is called the [Gibbs phenomenon](https://en.wikipedia.org/wiki/Gibbs_phenomenon). This is not yet resolved in this project's current implementation of the square wave, but a technique called [sigma approximation](https://en.wikipedia.org/wiki/Sigma_approximation) can be done to greatly reduce these "ringing" artifacts.

The Fourier series for the square wave is:

$$x_{\mathrm{square}}(t) = \frac{4}{\pi} \sum_{k=1}^{\infty} \frac{1}{2k-1} \sin(2\pi (2k-1) t)$$

```C#
private void _GenerateSquareWave(float[] targetInputBuffer)
{
    for (int i=0; i < AudioConstants.BLOCK_SIZE; i++)
    {
        float increment = Frequency / AudioConstants.MIX_RATE;
        float sumOfTerms = 0f;
        for (int k=0; k < NUM_SQUARE_HARMONICS; k++)
        {
            float n = 2*k + 1;
            sumOfTerms += 1 / n * Mathf.Sin(n * _phase * Mathf.Tau);
        }

        float baseValue = 4 / Mathf.Pi * sumOfTerms;
        targetInputBuffer[i] = SQUARE_NORMALIZATION_FACTOR * baseValue;
        _phase = Mathf.PosMod(_phase + increment, 1.0f);
    }
}
```

https://github.com/user-attachments/assets/08149b0a-f70b-40f4-b219-c5a9b7218ddf

And for sake of completeness, the sawtooth wave:

$$x_{\mathrm{sawtooth}}(t) = -\frac{2}{\pi} \sum_{k=1}^{\infty} \frac{(-1)^k}{k} \sin (2\pi kt)$$

```C#
private void _GenerateSawtoothWave(float[] targetInputBuffer)
{
    for (int i=0; i < AudioConstants.BLOCK_SIZE; i++)
    {
        float increment = Frequency / AudioConstants.MIX_RATE;
        float sumOfTerms = 0f;
        for (int k=1; k < NUM_SAWTOOTH_HARMONICS; k++)
        {
            float sign = k % 2 == 0 ? 1 : -1;
            sumOfTerms += sign / k * Mathf.Sin(k * _phase * Mathf.Tau);
        }

        float baseValue = -2 / Mathf.Pi * sumOfTerms;
        targetInputBuffer[i] = SAWTOOTH_NORMALIZATION_FACTOR * baseValue;
        _phase = Mathf.PosMod(_phase + increment, 1.0f);
    }
    }
```

https://github.com/user-attachments/assets/f6c7a730-556f-4915-991f-05e75cbf9779

Yay, now we have implementations of all four wave types!

I also made a module that combines all four wave types into a single *wave module*, where the wave type is a selectable option. I added a slider for rudimentary control of the frequency i.e. pitch.

https://github.com/user-attachments/assets/80ece6fe-2280-4a6f-9ae8-d3bd14f52fbb

Some final comments: the visuals for pretty much everything is kind of prototypal. I don't dislike this art style so far! But I could definitely see the look and feel of the system being overhauled when things are starting to feel pretty feature-complete. The IO ports are definitely the most egregious thing right now, where the IO family and type is not clear at all and is a color convention right now. Here's a brief table of what the port color corresponds to:

| color       | io family/type |
| ----------- | -------------- |
| red         | audio input    |
| light blue  | audio output   |
| orange      | midi input     |
| green       | midi output    |
| purple      | trigger input  |
| dark blue   | trigger output |

I'll omit what MIDI and trigger IO ports are used for until the next entry, where we'll talk about audio envelopes and connecting MIDI input.

#### <<< [\[02\] beginnings](./02_audio-graphs.md) | [\[04\] midi keyboards](./04_midi-keyboards.md) >>>