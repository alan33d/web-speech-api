# Web Speech API Proposal: SpeechRecognitionResult Timestamps

**Author:** alanding@google.com, evliu@google.com

### Problem

The Web Speech API currently does not expose the start and end timestamps of the source audio corresponding to a given transcription result (`SpeechRecognitionResult`). This limitation creates a major challenge for API clients requiring timeline association use cases:

- **Timeline Association:** Developers cannot readily associate transcribed text with specific segments of the audio source, making it difficult to map generated captions to media timelines, audio tracks, or video frames:
  - **Subtitling & Closed Captions:** Web applications cannot automatically generate synchronized subtitle tracks (e.g. WebVTT / SRT cues) because they lack the exact `[startTime, endTime]` boundaries for each phrase.
  - **Interactive Meeting Transcripts ("Click-to-Seek"):** In recorded meetings, video lectures, and podcast players, applications cannot offer "click-to-seek" navigation—where clicking on a sentence or word in the transcript jumps media playback to that exact moment.
  - **Live WebRTC Video Sync & Lip-Sync:** In real-time video conferencing (e.g. Google Meet), web applications cannot reliably synchronize live captions or translated subtitles with incoming video frames. Because DOM events conflate speech timing with processing and main-thread queuing delays, subtitles either lag behind speaker lip movement or disappear prematurely.
  - **Text-Based Media Editing:** Web-based podcast and video editors cannot allow users to cut, splice, or re-time media segments by editing transcript text without knowing the corresponding audio boundaries.

---

### Proposed Solution

We propose extending the `SpeechRecognitionResult` interface with `speechStartTime` and `speechEndTime` attributes.

#### Web IDL Definition

```webidl
partial interface SpeechRecognitionResult {
    // Start timestamp of the speech segment in seconds relative to the start of the audio stream (0.0s).
    readonly attribute double speechStartTime;

    // End timestamp of the speech segment in seconds relative to the start of the audio stream.
    readonly attribute double speechEndTime;
};
```

### Choice of Naming & Representation

1. **Mirroring `speechstart` and `speechend`:**
   * The names `speechStartTime` and `speechEndTime` mirror the existing `speechstart` and `speechend` events in the Web Speech API, clearly communicating that these timestamps bound the acoustic speech segment corresponding to the transcript hypothesis.

2. **Seconds as `double` (Consistency with Adjacent Media APIs):**
   * In adjacent W3C media specifications, media-local timelines are universally represented in **seconds** as a `double`:
     * **Web Audio API:** [`BaseAudioContext.currentTime`](https://webaudio.github.io/web-audio-api/#dom-baseaudiocontext-currenttime) (seconds)
     * **HTML Media Elements:** [`HTMLMediaElement.currentTime`](https://html.spec.whatwg.org/multipage/media.html#dom-media-currenttime) (seconds)
     * **AudioParam Scheduling:** [`AudioParam.setValueAtTime()`](https://webaudio.github.io/web-audio-api/#dom-audioparam-setvalueattime) (seconds)
   * Using seconds ensures seamless interoperability when routing audio between media elements, Web Audio graphs, and `SpeechRecognition`, avoiding repetitive unit conversions ($1000\times / \div 1000$).

3. **Semantic Inaccuracy of `DOMHighResTimeStamp`:**
   * Under [W3C High Resolution Time Level 3](https://www.w3.org/TR/hr-time-3/#sec-domhighrestimestamp), `DOMHighResTimeStamp` represents milliseconds relative to document uptime (`performance.timeOrigin`). Because `speechStartTime` and `speechEndTime` represent media-local stream offsets ($t = 0.0\text{s}$ at stream start), using `DOMHighResTimeStamp` would be semantically inaccurate.

---

### Proposed Behavior & Examples

The `speechStartTime` and `speechEndTime` properties represent the exact acoustic bounds (in seconds) of the recognized speech, measured from the start of the audio stream consumed by the recognizer ($t = 0.0\text{s}$, marked by `audiostart`).

#### Example 1: Automated Subtitling & "Click-to-Seek" Navigation

In this example, speech recognition results from an audio/video element are converted into WebVTT subtitle cues and interactive "click-to-seek" transcript links:

```javascript
const mediaElement = document.querySelector('video');
const track = mediaElement.addTextTrack('captions', 'English', 'en');
track.mode = 'showing';

const recognition = new SpeechRecognition();
recognition.continuous = true;
recognition.interimResults = false;

// Transcribe audio track from HTMLMediaElement
const audioStream = mediaElement.captureStream();
recognition.start(audioStream.getAudioTracks()[0]);

recognition.onresult = (event) => {
  for (let i = event.resultIndex; i < event.results.length; ++i) {
    const result = event.results[i];
    if (!result.isFinal) continue;

    const transcriptText = result[0].transcript;

    // 1. Create a synchronized WebVTT caption cue
    const cue = new VTTCue(result.speechStartTime, result.speechEndTime, transcriptText);
    track.addCue(cue);

    // 2. Build interactive transcript element ("Click-to-Seek")
    const transcriptItem = document.createElement('p');
    transcriptItem.textContent = `[${result.speechStartTime.toFixed(1)}s] ${transcriptText}`;
    transcriptItem.onclick = () => {
      mediaElement.currentTime = result.speechStartTime;
      mediaElement.play();
    };
    document.getElementById('transcript-container').appendChild(transcriptItem);
  }
};
```

---

#### Example 2: Synchronizing Captions with Live WebRTC Video Frames

In live video conferencing, video frames delivered via `HTMLVideoElement.requestVideoFrameCallback()` contain timestamps relative to the document timeline. Developers can map `speechStartTime` and `speechEndTime` to the document timeline to render captions on the exact video frames when the speaker was talking:

```javascript
const recognition = new SpeechRecognition();
recognition.continuous = true;
recognition.interimResults = true;

let audioOriginTimeMs = 0;

// Capture the audio stream's start timestamp on the document timeline
recognition.onaudiostart = (event) => {
  audioOriginTimeMs = event.timeStamp;
};

const activeCues = [];

recognition.onresult = (event) => {
  const result = event.results[event.resultIndex];

  // Convert stream offsets (seconds) to document timeline (milliseconds)
  const startDocTimeMs = audioOriginTimeMs + (result.speechStartTime * 1000);
  const endDocTimeMs   = audioOriginTimeMs + (result.speechEndTime * 1000);

  // Update or add subtitle cue
  activeCues[event.resultIndex] = {
    text: result[0].transcript,
    start: startDocTimeMs,
    end: endDocTimeMs,
    isFinal: result.isFinal
  };
};

// Frame-accurate rendering loop synchronized with WebRTC video playback
function renderVideoSubtitleFrame(now, metadata) {
  const currentVideoDocTimeMs = metadata.expectedDisplayTime; // document timeline

  const currentCue = activeCues.find(
    cue => currentVideoDocTimeMs >= cue.start && currentVideoDocTimeMs <= cue.end
  );

  subtitleOverlay.textContent = currentCue ? currentCue.text : '';
  videoElement.requestVideoFrameCallback(renderVideoSubtitleFrame);
}
videoElement.requestVideoFrameCallback(renderVideoSubtitleFrame);
```

---

### Security and Privacy Considerations

#### Fingerprinting Risk
Exposing sub-millisecond acoustic timing can enable hardware profiling (measuring CPU execution speed, thermal throttling, and system load), creating a potential tracking vector for cross-origin user fingerprinting.

#### Mitigation Strategy
To mitigate potential side-channel and fingerprinting vectors:
* **Precedent:** Implementations (such as Chromium) follow the security posture established by [`HTMLMediaElement.currentTime`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/currentTime), coarsening raw engine timestamps to discrete resolution buckets (e.g. 2ms resolution) in non-isolated execution contexts.
* **Specification Precedent:** In accordance with [W3C High Resolution Time Level 3](https://www.w3.org/TR/hr-time-3/#privacy-security), the specification does not mandate a hardcoded quantization value, allowing user agents to adjust timer resolution or introduce jitter according to their security and cross-origin isolation policies.

---

### Alternatives Considered

- **Splitting `result` into `resultstart` and `resultend` Events:**
  Splitting the single `result` event into separate `resultstart` and `resultend` DOM events (each carrying individual `event.timeStamp` values) was considered. However, localizing timestamps directly within `SpeechRecognitionResult` and `SpeechRecognitionResultList` is architecturally superior:
  1. **Fragile Client State Management:** Requiring developers to correlate separate, asynchronous start and end events across streaming interim updates introduces a complex, race-prone state machine into web applications.
  2. **Decoupling from Cumulative Results (`SpeechRecognitionResultList`):** In continuous recognition mode (`continuous = true`), the API maintains a cumulative list of all results generated during the session. If timing information is only emitted on transient events, historical results stored in `event.results` would have no localized timestamps. Developers would be forced to manually maintain a secondary timeline mapping in application code.
  3. **Unnecessary Event Overhead for Limited Utility:** Adding yet another pair of events that client applications must process adds event-loop overhead and API surface complexity with limited practical use cases compared to self-contained result properties.
  4. **Self-Contained Data Model:** Attaching `speechStartTime` and `speechEndTime` directly to `SpeechRecognitionResult` ensures that each transcript hypothesis is self-contained and atomically bound to its exact temporal audio segment, allowing results to be passed, cached, and manipulated independently of DOM event lifecycles.

- **Using Existing VAD (`speechstart` / `speechend`) and `result` Event Timestamps:**
  Relying on existing Voice Activity Detection (VAD) events (`speechstart` / `speechend`) and `result` event timestamps was considered. While `speechend.timeStamp` (or the proposed start and end timestamps on `SpeechRecognitionResult`) could potentially be used for basic processing latency tracking, existing events are fundamentally insufficient for timeline association:
  1. **Cannot Capture Start Times in Continuous Recognition:** Neither VAD events nor `result` events capture the acoustic start timestamp of interim or final results that occur in the middle of a continuous recognition session. In continuous recognition mode (`continuous = true`), `speechstart` fires only once at the beginning of the entire session, leaving all subsequent phrases without an acoustic start boundary.
  2. **No Start Timestamp on `speechend` or `result` Events:** `speechend` only marks when speech activity ended, and `result.timeStamp` only indicates when the DOM event was dispatched. Neither provides the acoustic start time needed to compute utterance duration or construct `[startTime, endTime]` subtitle cues.
  3. **No Support for Interim Results:** While a speaker is actively talking mid-sentence, `speechend` cannot fire, leaving streaming captions without timing information.
  4. **Endpointer Trailing Silence Skew:** Voice Activity Detection (VAD) endpointers only fire `speechend` after observing 500ms–1500ms of trailing silence. This introduces non-speech padding into the timestamp, degrading audio alignment.
