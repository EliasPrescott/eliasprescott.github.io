+++
title = "Making Virtual Instruments with SvelteKit - Part 1"
date = 2022-10-03
[taxonomies]
authors = ["Elias Prescott"]
tags = ["Svelte", "TypeScript", "WebAudio API"]
+++

In this tutorial, I will be using SvelteKit and the Web Audio API to make a basic instrument. For the uninitiated, [SvelteKit](https://kit.svelte.dev/) is a framework for building reactive web applications.

Let's get started by scaffolding a basic SvelteKit app. First, we run the standard project generation command:

```bash
npm create svelte@latest blog-instrument-app
```

This command should bring up several prompts. The first prompt was which Svelte app template to use. For this, I chose the Skeleton Project option. For the type checking prompt, I chose "Yes, using TypeScript syntax." I also answered yes to the ESLint and Prettier code formatting prompts. Finally, I answered yes to including Playwright in the project for browser testing. In a future post of this tutorial series, I am planning on developing a full suite of functional tests using Playwright, so I wanted to include that now.

After answering all the prompts, we can navigate into our new project folder and start it up:

```bash
cd blog-instrument-app
npm install
npm run dev -- --open
```

Here is the result of my NPM install:

![Dependency install result](/images/2022-10-03/no-vulnerabilities.PNG)

I am used to large Angular projects that report +50 vulnerabilities every time I update them, so it is refreshing to see 0 vulnerabilities 😊

After running all the commands, you should be greeted by a mostly blank web page. Our first order of business will be researching the [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API) and adding some basic audio generation. To hold our instrument logic, we will create a new file at ```src/lib/instrument.ts```. The goal for this ```instrument.ts``` file is to hold all the Web Audio code, and expose a clean API for our front-end code to utilize.

I don't know very much about audio synthesis, so I will be using [these](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Advanced_techniques) [two](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Advanced_techniques#putting_it_all_together) MDN tutorials to learn the Web Audio basics (AKA: I'm gonna steal code from them). I would recommend reading through these two tutorials and any of the other great resources on the Web Audio API if you are interested in learning more about audio programming.

Before we can make a peep with Web Audio, we need to do some setup. First we need to setup a wavetable file for our instrument and import it in ```instrument.ts```. We could use a default waveshape provided by the Web Audio API, but I am using custom wavetable in the hopes that it will sound more like a real instrument. I used the piano wavetable from this [repository](https://github.com/GoogleChromeLabs/web-audio-samples/blob/main/src/demos/wavetable-synth/wave-tables/Piano), but there are many other options for wavetables. To prepare the wavetable for use, I saved it under ```src/lib/wavetables/piano.ts```, and exported the wavetable like so:


```TypeScript
// src/lib/wavetables/piano.ts

let pianoWavetable = {
    // Values omitted for brevity
};

export default pianoWavetable;
```

Now that we have a wavetable set up, we can import it in ```instrument.ts``` and create our basic instrument class:

```TypeScript
// src/lib/instrument.ts

import pianoWavetable from "./wavetables/piano";  

export class Piano {
    audioContext: AudioContext;

    wave: PeriodicWave;

    play(length: number) {
        let osc = new OscillatorNode(this.audioContext, {
            frequency: 380,
            type: "custom",
            periodicWave: this.wave
        });

        this.audioContext.currentTime

        osc.connect(this.audioContext.destination);

        let currentTime = this.audioContext.currentTime;
        
        osc.start(currentTime);
        osc.stop(currentTime + length);
    }

    constructor(audioCtx: AudioContext) {
        this.audioContext = audioCtx;

        this.wave = new PeriodicWave(this.audioContext, {
            real: pianoWavetable.real,
            imag: pianoWavetable.imag
        });
    }
}
```

So, our Instrument API here takes in a reference to the page's ```AudioContext```, and then returns an instance of the ```Piano``` class. To test our initial sound, we are exposing the ```play``` method to be called from our front-end page script.

Now that we have our class ready for testing, we can move to the index page file located at ```src/routes/+page.svelte```. We can replace whatever default content ```+page.svelte``` has with this:

```html
<!-- src/routes/+page.svelte -->

<script>
    import { Piano } from '$lib/instrument';

    let audioContext = new AudioContext();

    let piano = new Piano(audioContext);    
</script>

<button on:click="{_ => piano.play(1000)}">
    Test Sound
</button>
```

If you are not familiar with Svelte/SvelteKit, it allows you to easily handle element events by using a ```on:{eventNameHere}``` attribute. The button click event handler must take in a ```MouseEvent``` instance. Because we don't care about the click information, I am just using an underline to show that the mouse event information is unused.

Assuming you still have your project running with auto-refresh, you should see this error message after saving our changes:

```
AudioContext is not defined
```

Initializing an ```AudioContext``` requires the window to be initialized and ready, which it isn't whenever our script block is currently executing. To ensure we initialize our ```AudioContext``` at the proper time, we can use the ```onMount``` hook provided by the Svelte runtime:

```html
<!-- src/routes/+page.svelte -->

<script lang="ts">
    import { Piano } from '$lib/instrument';
    import { onMount } from 'svelte';

    let audioContext: AudioContext | undefined;

    let piano: Piano;

    onMount(() => {
        audioContext = new AudioContext();

        piano = new Piano(audioContext);
    });
</script>

<button on:click="{_ => piano.play(1)}">
    Test Sound
</button>
```

To use the TypeScript-style type annotations, I had to add ```lang="ts"``` to the script tag.

Before clicking the button, **please** make sure your audio is turned down, preferably to whatever the minimum setting is. The default volume level for the noise we are generating is fairly loud. Even with my speakers set to the lowest possible volume, which is 2 for some reason, the noise comes through at a considerable volume. After checking your audio levels, you should be ready to click the "Test Sound" button and listen to the harsh beep it produces.

Producing a sound is good, but we need to add more control to our ```Piano``` class. For our final piano, I want users to be able to click and hold on the keys to play different notes. This means that we need to stop passing in a pre-determined length for each note. Instead, our page script should be able to start and stop notes freely based on user input.

To accomplish this, we can replace the ```play``` method on our ```Piano``` class with two methods: ```startNote``` and ```stopNote```. Another requirement for our improved instrument logic should be variable frequencies for the notes. Our page script should be able to pass in different frequencies based on the notes that a user is playing. Our new ```startNote``` method should take in a frequency, start the ```OscillatorNode``` instance, and then stash that oscillator node into a list somewhere. The reason we need to store these nodes is because we will need to delete them later on with our ```stopNote``` method. The ```stopNote``` method will also take in a frequency, which it will use as a (kinda) unique ID to identify which stored ```OscillatorNode``` to stop and disconnect. After making these changes, here is our improved ```instrument.ts```.

```TypeScript
// src/lib/instrument.ts

import pianoWavetable from "./wavetables/piano";  

export class Piano {
    audioContext: AudioContext;

    wave: PeriodicWave;

    sweepLength: number = 2;

    oscillators: OscillatorNode[] = [];

    startNote(frequency: number) {
        let osc = new OscillatorNode(this.audioContext, {
            frequency: frequency,
            type: "custom",
            periodicWave: this.wave
        });

        osc.frequency.defaultValue

        this.audioContext.currentTime

        osc.connect(this.audioContext.destination);

        let currentTime = this.audioContext.currentTime;

        osc.start(currentTime);
        
        this.oscillators.push(osc);
    }

    stopNote(frequency: number) {
        let matchingOsc = this.oscillators
            .find(x => Math.trunc(x.frequency.value) == Math.trunc(frequency));

        if (matchingOsc) {
            matchingOsc.stop();
            matchingOsc.disconnect();
            this.oscillators = this.oscillators
                .filter(x => Math.trunc(x.frequency.value) != Math.trunc(frequency));
        }
    }

    constructor(audioCtx: AudioContext) {
        this.audioContext = audioCtx;

        this.wave = new PeriodicWave(this.audioContext, {
            real: pianoWavetable.real,
            imag: pianoWavetable.imag
        });
    }
}
```

The ```Math.trunc()``` calls might not be necessary when we just play our default note, but they will be critical later on when we add more notes. Whenever the frequency is stored in the ```OscillatorNode``` instance, it can change by extremely small amounts. These changes will interfere with our equality checks in ```stopNote```, which would mean that all of our notes would play infinitely. Additionally, checking floats for equality can be a surprisingly intensive operation, so it is good to avoid that whenever possible.

After changing the API on our ```Piano``` class, we need to update ```src/routes/+page.svelte``` to use the new methods:

```html
<!-- src/routes/+page.svelte -->

<script lang="ts">
    import { Piano } from '$lib/instrument';
    import { onMount } from 'svelte';

    let audioContext: AudioContext | undefined;

    let piano: Piano;

    onMount(() => {
        audioContext = new AudioContext();

        piano = new Piano(audioContext);
    });
</script>

<button 
    on:mousedown="{_ => piano.startNote(440)}" 
    on:mouseup="{_ => piano.stopNote(440)}"
    on:mouseleave="{_ => piano.stopNote(440)}">
    Test Sound
</button>
```

This new change gives us a lot more control, but only having one note is pretty boring. Let's add in some more notes:

```html
<!-- src/routes/+page.svelte -->

<script lang="ts">
    import { Piano } from '$lib/instrument';
    import { onMount } from 'svelte';

    let audioContext: AudioContext | undefined;

    let piano: Piano;

    onMount(() => {
        audioContext = new AudioContext();

        piano = new Piano(audioContext);
    });

    let testingNotes = [
        { key: "C", frequency: 261.63, minor: false },
        { key: "Db", frequency: 277.18, minor: true },
        { key: "D", frequency: 293.66, minor: false },
        { key: "Ed", frequency: 311.13, minor: true },
        { key: "E", frequency: 329.63, minor: false },
        { key: "F", frequency: 349.23, minor: false },
        { key: "Gb", frequency: 369.99, minor: true },
        { key: "G", frequency: 392.0, minor: false },
        { key: "Ab", frequency: 415.3, minor: true },
        { key: "A", frequency: 440.0, minor: false },
        { key: "Bd", frequency: 466.16, minor: true },
        { key: "B", frequency: 493.88, minor: false },
    ];
</script>

{#each testingNotes as note}
<button 
    on:mousedown="{_ => piano.startNote(note.frequency)}" 
    on:mouseup="{_ => piano.stopNote(note.frequency)}"
    on:mouseleave="{_ => piano.stopNote(note.frequency)}">
    {note.key}
</button>
{/each}
```

After saving and refreshing these changes, you should be able to click between the buttons and play a nice variety of notes. The buttons still don't look very nice, so I'll add some basic styling to make them look more like piano notes: 

```html
<!-- src/routes/+page.svelte -->

<script lang="ts">
    // Contents omitted for brevity
</script>

<div class="piano">
    <h3>
        Blog Piano
    </h3>
    <div class="pianoNotes">
        {#each testingNotes as note}
        <button class="pianoNote{note.minor ? ' minorPianoNote' : ''}"
            on:mousedown="{_ => piano.startNote(note.frequency)}"
            on:mouseup="{_ => piano.stopNote(note.frequency)}"
            on:mouseleave="{_ => piano.stopNote(note.frequency)}">
            {note.key}
        </button>
        {/each}
    </div>
</div>

<style>
    .piano {
        padding: 1em;
        background-color: rgb(40, 40, 40);
        box-shadow: 0 2px 64px rgba(0,0,0,0.5);
        border-radius: 2em;
        color: whitesmoke;
    }

    .piano h3 {
        width: 100%;
        height: 2em;
        margin: 0;
    }

    .pianoNotes {
        display: flex;
        height: 250px;
        gap: 2px;
        border-radius: 1em;
    }

    .pianoNote {
        border: 1px solid black;
        border-top: none;
        border-radius: 0 0 8px 8px;
        width: 60px;
        height: 100%;
        display: flex;
        align-items: flex-end;
        box-shadow: none;
    }

    .minorPianoNote {
        background-color: black;
        border-color: whitesmoke;;
        color: whitesmoke;
        height: 75%;
        margin-left: -30px;
        margin-right: -30px;
        z-index: 2;
    }
</style>
```

To finish off the styling changes, I created ```static/style.css``` and added this link in ```src/app.html```:

```html
<link rel="stylesheet" href="style.css">
```

Here is the basic style rule in ```static/style.css```:

```css
body {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    width: 100vw;
    margin: 0;

    background-color: rgb(45, 45, 45);
}
```

Here's our final result:

![Piano Instrument App](/images/2022-10-03/final-piano.PNG)

Moving forward, I would like to revisit this project and continue to work on it. Some possible future improvements include more polished sound synthesis, adding control sliders, adding more instruments, adding some support for saving/loading MIDI files, adding functionality tests using Playwright, and converting the app into a desktop application using [Tauri](https://tauri.app). I hope you enjoyed following along!