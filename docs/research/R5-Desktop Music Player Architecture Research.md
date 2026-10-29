# **Architectural Validation and Engine Design for a High-Performance Desktop Music Player**

The following comprehensive research report evaluates the architectural feasibility, performance characteristics, and long-term maintainability of developing a dual-module desktop music application (comprising a Player and a Library Manager) tailored for large local music collections (exceeding 100,000 tracks). The analysis rigorously stress-tests the foundational assumptions of utilizing a Windows-first environment, a Tauri 2 application shell, a Rust-based audio engine (symphonia and cpal), SQLite with FTS5, and specific frontend frameworks. Because the development relies on AI coding assistance with a constrained budget, the evaluation heavily prioritizes architectural choices that minimize integration complexity, reduce the surface area for concurrency bugs, and avoid extensive native C/C++ build pipelines that AI models historically struggle to debug. The research cutoff date is October 24, 2026\.

## **Part 1: The Playback Engine Architecture**

### **Evaluation of Audio Engine Paradigms**

Establishing a reliable, gapless audio engine capable of handling high-resolution FLAC files requires evaluating a spectrum of audio pipelines, ranging from pure Rust implementations to embedded browser APIs.  
**Pure Rust Pipelines (symphonia \+ cpal, rodio, kira)** The combination of symphonia (for decoding and demuxing) and cpal (for hardware output) represents the most granular, low-level pure Rust approach. This pipeline avoids external C dependencies, which is highly advantageous for an AI-assisted workflow, as the build process strictly relies on Cargo. rodio wraps symphonia and cpal into a higher-level API (VERIFIED: https\://github.com/rustaudio/rodio, Oct 24, 2026\)1. However, rodio obscures the underlying buffers, making precise, sample-accurate gapless transitions and tap-points for FFT visualizers difficult to implement without rewriting its core traits. kira is another pure Rust audio library, but its architecture is heavily optimized for game audio (triggering short, overlapping sound effects) rather than long-form streaming of large, compressed audio files with complex gapless metadata.  
**C Bindings and Frameworks (miniaudio, GStreamer, libmpv, libvlc)** miniaudio is an exceptional single-file C library used widely in the industry. However, the primary Rust bindings (miniaudio-rs) were archived and abandoned in 2023 (REPORTED: https\://github.com/ExPixel/miniaudio-rs, Oct 24, 2026\)2. Relying on unmaintained FFI (Foreign Function Interface) bindings introduces severe security and compilation risks. GStreamer (via gstreamer-rs), libmpv, and libvlc offer unparalleled format support and maturity (VERIFIED: https\://github.com/GStreamer/gst-plugins-rs/blob/main/CHANGELOG.md, Oct 24, 2026\)3. Nevertheless, they require bundling massive shared libraries (DLLs) and managing complex C-state machines. This drastically increases the installation footprint and violates the constraint of minimizing maintenance complexity for an AI-assisted developer.  
**Proprietary SDKs (BASS, FMOD)** BASS and FMOD are industry-standard, highly performant audio engines. However, they are closed-source and require paid commercial licenses for distribution. BASS, specifically, requires a commercial license from Un4seen Developments for any non-freeware or commercial desktop application (REPORTED: https\://github.com/mtxmiller/LyrPlay, Oct 24, 2026\)4. These licensing hurdles disqualify them for a low-budget, independent project.  
**The HTML5 \<audio\> and Web Audio API Route**  
Attempting to handle audio playback purely within the WebView2 frontend via the HTML5 \<audio\> element or Web Audio API is a structurally flawed approach for audiophile-grade applications. The HTML5 \<audio\> tag lacks native support for true gapless playback; transitioning between tracks always incurs a latency penalty governed by the browser's internal media pipeline. The Web Audio API can achieve gapless playback by stitching AudioBufferSourceNode objects, but this requires decoding the entire audio file into the browser's memory. A 192 kHz / 24-bit FLAC track decoded into 32-bit float PCM can consume over 500 MB of RAM per track, leading to rapid memory exhaustion and garbage collection stutter (UNVERIFIED). Furthermore, Web Audio introduces inherent latency when tapping audio for FFT visualizers and prevents the application from seizing exclusive control of the digital-to-analog converter (DAC).

| Engine Candidate | Paradigm | Maintainability & AI-Readiness | Gapless & FLAC Support | License | Verdict |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Symphonia \+ Cpal** | Pure Rust | High. Native to Rust ecosystem. | Excellent. FLAC native; gapless supported. | MIT/Apache | **Recommended.** Best balance of control, memory efficiency, and low overhead. |
| **Rodio** | Pure Rust (High-level) | Very High. Abstracted API. | Moderate. Lacks granular crossfade/gapless control. | MIT/Apache | **Discard.** Too high-level for custom visualizers and precise gapless buffering. |
| **Kira** | Pure Rust (Game Audio) | High. | Poor for long-form streaming. | MIT/Apache | **Discard.** Optimized for games, not media players. |
| **Miniaudio (FFI)** | C Library \+ Rust Bindings | Low. Bindings archived in 2023\. | Excellent. | Public Domain / MIT | **Discard.** Unmaintained bindings introduce compilation risks. |
| **GStreamer / libmpv** | Native C Framework | Low. Immense complexity; heavy DLLs. | Excellent. Industry standard. | LGPL / GPL | **Discard.** Violates the "low complexity/maintenance" constraint. |
| **BASS / FMOD** | Proprietary SDK | High. Well-documented APIs. | Excellent. | Commercial | **Discard.** Requires paid commercial licenses. |
| **Web Audio API** | Browser/WebView2 | High. Native to JS/TS. | Poor. High memory bloat; no DAC exclusive mode. | N/A | **Discard.** Unsuitable for audiophile playback and large FLAC files. |

### **Symphonia Specifics: Formats, Gapless, and Seeking**

symphonia serves as a pure-Rust audio decoding and demuxing library. Its performance benchmarks indicate that it generally operates within ![][image1] of FFmpeg's decoding speeds, rendering it exceptionally capable of decoding 24-bit/192 kHz FLAC files in real-time with negligible CPU overhead on modern hardware (VERIFIED: https\://github.com/pdeljanov/symphonia, Oct 24, 2026\)7.  
The format maturity of symphonia aligns perfectly with a local music player's requirements. FLAC, WAV, ALAC, and MP3 are fully compliant and play losslessly. AAC-LC and Ogg Vorbis are well-supported. WavPack and APE (Monkey's Audio) are supported via their respective codec flags. DSD (Direct Stream Digital) audio formats (DSF/DFF) are technically supported for demuxing and can output in raw mode (mode 0\) as packed bytes, but Symphonia does not natively convert DSD to PCM audio; this requires a dedicated external conversion layer, making DSD a strong candidate for deferral in a v1 release (VERIFIED: https\://github.com/pdeljanov/Symphonia/pull/503/files, Oct 24, 2026\)7.  
**Gapless Playback:** Gapless playback is natively supported by Symphonia, provided the application architecture utilizes it correctly. Setting enable\_gapless \= true in the FormatOptions instructs the FormatReader to process gapless metadata (such as LAME/Xing headers for MP3s or iTunSMPB for AAC/M4A). The decoders use this trimming information to discard encoder delay at the start of a track and padding samples at the end (VERIFIED: https\://github.com/pdeljanov/Symphonia/discussions/169, Oct 24, 2026\)9.  
**Seeking and Known Bugs:**  
Seek accuracy in Symphonia operates on packet boundaries. To achieve exact sample-accurate seeking, the audio engine must perform a coarse seek to the nearest preceding packet, decode the intermediate packets, and discard the surplus samples until the target playhead position is reached.  
Two notable bugs and architectural quirks must be navigated:

> 1. **LAME Xing Headers in Joint Stereo:** There is a reported issue where Symphonia's joint stereo processing fails if the channels within an MP3 granule do not share the exact same scale factor representation, leading to CRC failures early in the parsing stage (REPORTED: https\://github.com/pdeljanov/Symphonia/issues/556, Oct 24, 2026\)11.  
> 2. **Fragmented MP4s:** When seeking within fragmented MP4 streams (such as those generated by YouTube ripping tools), Symphonia must scan numerous top-level atoms (MOOF/MDAT pairs), triggering massive I/O overhead on the initial seek (VERIFIED: https\://github.com/pdeljanov/Symphonia/discussions/153, Oct 24, 2026\)13. While less relevant for standard local FLAC libraries, it poses a risk if users import web-ripped audio.

### **Cpal on Windows: Hardware Integration and Resampling**

cpal is the defacto standard for cross-platform audio output in Rust. On Windows, it interfaces primarily with WASAPI (Windows Audio Session API).  
**WASAPI Shared vs. Exclusive Mode:**  
By default, cpal opens WASAPI in Shared Mode. In this mode, the Windows OS audio engine intercepts the stream, resamples it if necessary, and mixes it with other application sounds (e.g., system notifications). This is highly reliable but compromises bit-perfect audiophile playback. WASAPI Exclusive mode bypasses the OS mixer, granting the application direct access to the DAC. However, Exclusive mode is brittle; it will hard-crash or return a cpal::Error if the application attempts to push a sample rate or bit depth that the DAC does not natively support at that exact moment.  
**Sample Rate and Bit-Depth Handling:** Recent updates to cpal (post-v0.17) have altered the default behavior to prefer 48 kHz output over the traditional CD-quality 44.1 kHz (VERIFIED: https\://github.com/rustaudio/cpal/releases, Oct 24, 2026\)14. Because local music libraries are an amalgamation of 44.1 kHz, 48 kHz, 96 kHz, and 192 kHz files, sending these directly to a WASAPI Shared endpoint requires a software resampler. The rubato crate provides high-quality, asynchronous sample rate conversion and should be integrated directly after the Symphonia decoder (VERIFIED: https\://github.com/HEnquist/rubato/, Oct 24, 2026\)15.  
**Device Hot-Swap and Sleep Behavior:** Audio device disconnection (e.g., unplugging USB headphones) or system sleep triggers a device failure. Historically, cpal does not gracefully auto-migrate streams. Instead, the audio callback will surface a DeviceNotAvailable or DeviceBusy error (the latter introduced in v0.18). The engine architecture must trap these errors, pause the internal playback state, poll the system for the new default audio device, and manually reconstruct the cpal::Stream (VERIFIED: https\://github.com/RustAudio/cpal/issues/373, Oct 24, 2026\)14.  
**ASIO Support:** While cpal can support ASIO for ultra-low latency, doing so requires the developer to download the proprietary Steinberg ASIO C++ SDK, set a CPAL\_ASIO\_DIR environment variable, and compile against LLVM/Clang (VERIFIED: https\://github.com/RustAudio/cpal, Oct 24, 2026\)18. This violently conflicts with the project's constraint of minimizing complexity and should be categorically abandoned.

### **Reliability Architecture: Threads, Buffers, and Latency**

A robust desktop audio engine cannot rely on synchronous operations or garbage-collected runtimes. To prevent audible glitches (buffer underruns) and maintain UI responsiveness, strict thread isolation is paramount.

> 1. **Thread Model Separation:**  
   * **UI/Main Thread:** Managed by the Tauri shell and WebView2.  
   * **I/O Thread:** Responsible for executing SQLite queries and reading raw bytes from the physical disk or network drive.  
   * **Decoder Thread:** Runs symphonia, demuxing packets and decoding them into raw PCM floating-point arrays. It also applies rubato resampling if the source sample rate differs from the DAC's operating rate.  
   * **Audio Output Thread:** The cpal real-time callback thread. It must *only* read from memory and push to the DAC. It is strictly forbidden from performing I/O, allocating memory, or blocking on mutexes.  
> 2. **Lock-Free Ring Buffers:** Data must transition from the Decoder Thread to the Audio Output Thread via bounded, single-producer, single-consumer (SPSC) lock-free ring buffers (utilizing crates like ringbuf or rtrb). Using a standard std::sync::Mutex to share audio buffers will cause priority inversion; if the audio thread attempts to acquire the lock while the decoder thread is busy allocating memory, the DAC starves, resulting in a crackling sound.  
> 3. **Gapless Pre-buffering and Crossfading:** True gapless playback mandates that the engine never closes and reopens the audio device between tracks. The Decoder Thread must monitor the remaining samples in the current track. Seconds before the track concludes, it must open the next file in the queue, trim its encoder padding, and seamlessly begin pushing the new track's PCM samples into the same continuous ring buffer (VERIFIED: https\://github.com/pdeljanov/Symphonia/discussions/169, Oct 24, 2026\)9. Crossfading requires a more complex architecture: a dual-decoder pipeline feeding an intermediate mixer node that applies linear gain interpolation before pushing to the final ring buffer.  
> 4. **DSP: Volume, EQ, and ReplayGain:** Because the PCM data is floating-point (f32), software volume and ReplayGain are achieved via simple scalar multiplication of the samples before they enter the ring buffer. Software EQ requires implementing biquad filters, which adds complexity but can be executed in the Decoder Thread.  
> 5. **Resilience to Slow Disks:** Local network drives (SMB/NFS) or spinning HDDs can exhibit high latency spikes. The lock-free ring buffer must be sized appropriately (e.g., 2 to 5 seconds of audio data) to absorb I/O jitter without starving the audio thread.

### **Real-World Evidence: Open-Source Rust Players**

Analysis of existing open-source Rust audio players validates the proposed architecture:

* **RSPlayer:** successfully utilizes a pure-Rust pipeline (symphonia \+ cpal) coupled with rubato for automatic resampling. It achieves gapless playback and local library management without native C dependencies (REPORTED: https\://github.com/ljufa/rsplayer, Oct 24, 2026\)19.  
* **Honeycomb:** A macOS/Windows desktop player built precisely on the Tauri \+ React/TypeScript \+ symphonia/cpal stack. Its custom audio engine ("Kaseki") achieves gapless playback and handles 100k+ track virtualization, definitively proving that this exact architectural assumption is viable in production (REPORTED: https\://github.com/n1ji/honeycomb, Oct 24, 2026\)20.  
* **Musicat:** This project initially attempted WebAudio but abandoned it due to the inability to manage gapless transitions seamlessly. The developers pivoted to a symphonia \+ cpal thread, citing that direct control over the raw audio stream via a ring buffer solved the gapless issue. They utilized WebRTC to stream real-time FFT visualization data back to the frontend (REPORTED: https\://github.com/basharovV/musicat/discussions/6, Oct 24, 2026\)21.

### **Visualizer Data Path: The FFT Pipeline**

The application requires high-quality audio visualizers. Processing the Fast Fourier Transform (FFT) in the frontend Web Audio API introduces high latency. The FFT calculation must happen in Rust.

> 1. **Tapping the Audio:** As the cpal output thread consumes PCM samples from the primary ring buffer, it copies a small window of these samples into a *secondary* lock-free ring buffer dedicated strictly to telemetry. This ensures the visualizer tap does not disrupt the DAC.  
> 2. **Computing the FFT:** A low-priority telemetry thread reads this secondary buffer, applies a Blackman-Harris windowing function (to reduce spectral leakage at the edges of the sample window), and computes the FFT using the rustfft crate. rustfft is heavily optimized, caching plans internally for ![][image2] lookups, making it highly performant for real-time applications (REPORTED: https\://gist.github.com/proffesor-for-testing/02321e3f272720aa94484fffec6ab19b, Oct 24, 2026\)22.  
> 3. **Streaming to the Frontend:** The resulting frequency bins must be dispatched to the Tauri WebView2 frontend at 60 Hz.

### **Engine Scope Deferrals**

To maintain the AI coding budget, strict scope boundaries must be drawn.

* **CUE Sheets:** Parsing CUE sheets for single-file album rips adds immense demuxing complexity and complicates the SQLite database schema. Defer or abandon.  
* **Multi-Channel Audio:** Downmixing 5.1/7.1 surround sound to stereo requires complex channel mapping matrices. Restrict v1 to stereo output.  
* **Hi-Res / DSD:** While 24-bit/192 kHz FLAC is natively handled by Symphonia, DSD requires heavy PCM conversion overhead. Defer DSD.

### **Engine Design Deliverables**

**Recommended Minimum-Reliable v1 Engine Design:**  
A single Rust binary utilizing symphonia for decoding, rubato for resampling to 48 kHz, and cpal outputting to WASAPI Shared Mode. Data flows through an rtrb lock-free ring buffer. A parallel telemetry thread computes rustfft arrays for the visualizer.

| Risk | Mitigation |
| :---- | :---- |
| **Buffer Underruns (Crackle)** | Enforce strict lock-free SPSC ring buffers. Never use Mutex in the cpal audio callback thread. |
| **Device Disconnection Crash** | Trap cpal::ErrorKind::DeviceNotAvailable, pause the state machine, poll for the default device, and rebuild the stream. |
| **LAME/Xing Header Parsing Panic** | Wrap symphonia packet decoding in a robust Result match arm; gracefully skip corrupted MP3 packets rather than panicking the thread. |

**Test Plan:**

> 1. *Gapless Validation:* Play a known gapless album (e.g., Pink Floyd's *The Dark Side of the Moon*) split into separate FLAC files. The transition must contain zero audible clicks or silence.  
> 2. *Seek Validation:* Seek to exactly 03:14 in a 192 kHz FLAC file. The audio must resume instantaneously without dropping buffers.  
> 3. *Device Switch Validation:* Begin playback on USB headphones. Unplug the headphones mid-song. The audio must gracefully pause or auto-route to desktop speakers without crashing the Tauri process.

## **Part 2: The Tauri 2 Application Shell**

Tauri 2 provides a lightweight Rust executable that hosts a WebView2 instance. While it promises massive memory savings over Electron, utilizing it for an animation-heavy, performance-sensitive application with 100k+ library management introduces specific architectural challenges.

### **Tauri 2 on Windows: The Webview2 Reality**

WebView2 is distributed via the Evergreen Bootstrapper, meaning it relies on the underlying Microsoft Edge runtime installed on the user's machine. Tauri handles this gracefully via its NSIS or MSI installer configurations (VERIFIED: https\://github.com/tauri-apps/tauri/blob/dev/crates/tauri-utils/CHANGELOG.md, Oct 24, 2026\)24.  
**Memory Footprint and Startup:**  
Tauri applications idle between 30–60 MB of RAM, and startup times are typically under 400ms. This heavily outperforms Electron, which carries the initialization overhead of an embedded Node.js environment and a bundled Chromium instance.  
**Background Throttling (The Critical Risk):** Chromium (and by extension WebView2) aggressively throttles background processes to conserve system resources. When a Tauri window is minimized, requestAnimationFrame drops to 1 Hz, and JavaScript timers (setTimeout, setInterval) are severely throttled to a minimum execution time of 1000ms. If the music player relies on a frontend JavaScript timer to track the playhead, request the next track in the queue, or compute visualizer states, the player will effectively stall when minimized (VERIFIED: https\://github.com/tauri-apps/tauri/issues/5250, Oct 24, 2026\)25. *The implication is profound:* The Rust backend must independently manage the queue, state machine, and gapless transitions. The frontend must act strictly as a "dumb terminal" that blindly renders whatever state the backend dictates when the window is in focus.  
**IPC Performance and Custom Protocols:** Rendering an animation-heavy interface with album artwork requires optimizing the Inter-Process Communication (IPC) bridge. Sending base64 encoded images over Tauri's standard invoke commands causes severe JSON serialization blocking on the main thread, leading to dropped frames. Tauri 2 allows the registration of custom protocols (e.g., asset://). The Rust backend intercepts asset:// HTTP requests and serves the image bytes directly from SQLite or the disk, leveraging Chromium's native network stack and image caching capabilities (VERIFIED: https\://github.com/tauri-apps/tauri/blob/dev/crates/tauri-utils/CHANGELOG.md, Oct 24, 2026\)24. For high-frequency visualizer data (the FFT arrays), Tauri 2 supports raw binary payloads via IPC channels. Passing a Uint8Array over these channels bypasses string serialization entirely.

### **Frontend Performance: Virtualized Lists and Frameworks**

Displaying a Library Manager containing 100,000+ tracks requires strict DOM virtualization—rendering only the 30-50 rows currently visible on screen and swapping the data as the user scrolls.  
**React vs. Svelte vs. SolidJS:** The initial assumption proposed React and TypeScript. However, React's Virtual DOM reconciliation overhead scales poorly with high-frequency telemetry (such as 60Hz FFT visualizers updating canvas elements) and heavily virtualized grids. React's tendency to re-render entire component trees unless strictly memorized poses a significant risk to the "UI/UX quality and motion polish" priority. Conversely, Svelte 5 and SolidJS offer fine-grained reactivity without a Virtual DOM, executing DOM updates in fractions of a millisecond. Svelte ships a significantly smaller payload (![][image3] KB gzipped compared to React's ![][image4] KB) and is highly regarded for performant, animation-heavy Tauri applications (REPORTED: https\://www\.intuz.com/best-frontend-frameworks/, 2026\)27. If the AI coding assistant's training data allows, pivoting to Svelte 5 will drastically reduce CPU utilization. If React is strictly required, integrating TanStack Virtual is non-negotiable for the library grid.  
**Visualizers and Animations:**  
For audio visualizers, the HTML5 Canvas 2D API is highly performant for drawing FFT bars. WebGL or WebGPU is excessive for standard visualizers and introduces context-loss edge cases that AI assistants struggle to debug. For UI motion (page transitions, mini-player morphing), utilizing the browser's native View Transitions API or CSS/WAAPI is vastly superior to heavy JavaScript physics engines like Framer Motion or GSAP, which tax the WebView2 thread.

### **Windows Integration: SMTC and Media Keys**

To feel like a native desktop application, the player must integrate with hardware media keys (Play, Pause, Next) and the Windows 11 lock screen / volume overlay. This requires registering with the System Media Transport Controls (SMTC).  
The historical assumption in the Rust ecosystem was to use the souvlaki crate. However, souvlaki has largely fallen out of maintenance and lacks features like shuffle/repeat states and accurate positioning. The modern, maintained alternative is playwire. playwire exposes a unified API that wraps SMTC on Windows, MPRIS v2 over D-Bus on Linux, and MPNowPlayingInfoCenter on macOS (VERIFIED: https\://github.com/SFG5453/playwire, Oct 24, 2026\)28.  
Crucially, an open issue in the Stremio project highlights that applications relying solely on legacy WM\_APPCOMMAND messages will be starved of media keys if modern browsers (like Chrome or Edge) hijack the SMTC session (REPORTED: https\://github.com/Stremio/stremio-shell-ng/issues/84, Oct 24, 2026\)29. Integrating playwire guarantees that the Tauri application formally registers an SMTC session tied to the HWND window handle, ensuring media keys function even when the app is out of focus.

### **The Two-Module Structure: SQLite and Cargo Workspaces**

The requirement dictates two modules: a Player and a Library Manager. Compiling two separate binaries introduces significant complications regarding inter-process communication, file locking, and single-instance enforcement.  
The optimal architecture is a single Tauri binary that spawns two separate Webview windows. Both windows communicate with a shared Rust backend state. **SQLite Sharing and WAL Mode:** The backend maintains a single SQLite connection pool (rusqlite \+ r2d2). Because the Library Manager (writing metadata and tags) and the Player (reading track locations and writing play counts) operate simultaneously, the SQLite database must be configured to use PRAGMA journal\_mode=WAL; (Write-Ahead Logging). WAL mode allows concurrent readers and writers without database locking bottlenecks, preventing the Library Manager's mass-tagging operations from starving the Player's ability to fetch the next track (VERIFIED: https\://github.com/mvmcode/elves, Oct 24, 2026\)30.

### **Alternatives Sanity Check**

* **Electron:** Offers superior native API depth and ecosystem maturity. However, its memory consumption (![][image5] MB idle) and massive package size violate the strict performance constraints for large libraries. Tauri fundamentally wins on size, memory, and startup speed (REPORTED: https\://brainsroot.com/blog/why-tauri-over-electron/, 2026\)34.  
* **WinUI 3 / WPF (+ NAudio):** Offers the highest ceiling for native Windows integration. However, AI coding assistants struggle heavily with verbose XAML data-binding and C\# interop compared to the massive corpus of web development data available for React/Svelte.  
* **Qt / QML:** Powerful, but the C++ ecosystem is notoriously difficult for AI code generation due to complex memory management and pointer lifecycles.  
* **Slint / egui:** Pure Rust GUI frameworks. They are highly performant, but styling them to achieve the requested "UI/UX quality and motion polish" is extraordinarily difficult and time-consuming. They lack the vast, easily generated CSS animation libraries available in the web ecosystem.

### **Cross-Platform Cost**

If macOS or Linux support is required in the future, the architectural choices hold up well:

* **macOS:** Tauri uses WKWebView, which has stricter CORS and media autoplay policies than WebView2. However, the custom asset:// protocol bypasses this. The playwire crate seamlessly handles MPNowPlaying. cpal utilizes CoreAudio instead of WASAPI.  
* **Linux:** WebKitGTK struggles with WebGL hardware acceleration depending on proprietary Nvidia drivers, reinforcing the decision to stick to Canvas 2D for visualizers. playwire utilizes MPRIS for media keys.

### **Tauri Shell Deliverables**

**Stack Verdict Table**

| Component | Initial Assumption | Validation Verdict | Recommended Modification |
| :---- | :---- | :---- | :---- |
| **Shell** | Tauri 2 | **Valid.** | Utilize binary IPC channels to prevent JSON serialization blocking. |
| **Frontend** | React \+ TypeScript | **Caution.** | Strongly recommend **Svelte 5** or **SolidJS** for high-frequency FFT updates. If React is strictly used, TanStack Virtual is mandatory. |
| **Database** | SQLite \+ FTS5 | **Valid.** | Enforce PRAGMA journal\_mode=WAL; to allow simultaneous Player reading and Manager writing without locking. |
| **Media Keys** | (Implicit / None) | **Required.** | Integrate the playwire crate for SMTC integration, discarding the unmaintained souvlaki crate. |
| **Visualizer** | Rust FFT \-\> Canvas | **Valid.** | Execute via secondary lock-free ring buffer and rustfft Blackman-Harris windowing. |

**Ranked Risk List & Mitigations**

> 1. **Risk:** Background Throttling stalling playback.  
   * *Mitigation:* The Rust backend must autonomously manage track advancement and the queue state machine.  
> 2. **Risk:** UI Freezing on 100k+ List Scrolls.  
   * *Mitigation:* Absolute strict reliance on DOM virtualization. Pre-cache SQLite query pagination in Rust. Use a custom asset:// protocol for album art to leverage Chromium caching.  
> 3. **Risk:** IPC Bottlenecks dropping visualizer frames.  
   * *Mitigation:* Transmit FFT data as raw Uint8Array binary payloads over Tauri IPC channels, avoiding base64 or JSON stringification.

**Spike Tests (Validation Experiments)**  
Before committing the AI coding budget, execute these isolated experiments with strict pass/fail criteria:

> 1. **The Throttling Spike:** Build a minimal Tauri app with a JS setInterval updating a counter. Minimize the window for 5 minutes. *Pass criteria:* Understand the exact throttling curve and prove that Rust must own the timeline.  
> 2. **The FTS5 100k Spike:** Generate a mock SQLite database with 100,000 rows. Expose an FTS5 search query over Tauri IPC. *Pass criteria:* Search queries return rendered results to the virtualized UI grid in ![][image6]ms.  
> 3. **The IPC Telemetry Spike:** Send a simulated array of 1024 f32 floats from Rust to the frontend at 60 Hz. *Pass criteria:* The UI renders a Canvas without dropping frames, and CPU usage remains under 5%.  
> 4. **The Media Key Spike:** Run the Tauri app in the background. Open a YouTube video in Google Chrome (forcing Chrome to register SMTC). Press the hardware 'Play/Pause' button on the keyboard. *Pass criteria:* Using playwire, the Tauri app intercepts the command successfully.

## **Implications for my project**

The initial architectural instincts driving this project are remarkably accurate and structurally sound. The combination of Tauri 2, symphonia, cpal, and SQLite represents the bleeding edge of high-performance, low-footprint local desktop application design.  
However, because you are a freelance designer utilizing an AI coding assistant, minimizing rework is the paramount constraint. The most critical implication of this research is a necessary paradigm shift in how you prompt the AI. You must instruct the AI to build the architecture as a **"Headless Audio Server"** written in Rust, wrapped in a lightweight GUI, rather than a "web app that happens to play audio."  
If the JavaScript frontend is tasked with managing the play queue or gapless transitions, the application will catastrophically break the moment a user minimizes the window due to Chromium's background throttling. The Rust backend must be designed as an autonomous, multithreaded daemon. It receives discrete commands (PLAY\_PLAYLIST(id)), accesses the SQLite database, manages the gapless pre-buffering, decodes the audio, outputs the telemetry, and updates the SMTC presence. The frontend (whether React or Svelte) is merely a remote control monitoring the daemon's state.  
Furthermore, you must rigorously enforce lock-free programming concepts in the AI's Rust generation. AI assistants frequently default to wrapping shared state in Arc\<Mutex\<T\>\> to quickly satisfy the Rust compiler's borrow checker. If the AI places a Mutex on the audio buffer, the moment the Library Manager module queries the SQLite database and locks the main thread, the audio thread will starve, resulting in severe audible crackling.

## **What would change this conclusion**

> 1. **A hard requirement for Web Audio visualizers:** If you discover that implementing Canvas visualizers via binary IPC arrays is too difficult for the AI, and you wish to use the native Web Audio AnalyserNode logic, you would have to abandon the pure-Rust symphonia pipeline. You would be forced to route audio via a localhost HTTP server to the WebView2 \<audio\> tag, permanently sacrificing gapless playback and drastically increasing memory consumption.  
> 2. **Budget exhaustion on thread safety:** Lock-free multithreading in Rust is notoriously difficult, even for advanced AI models. If the AI repeatedly generates deadlocks, race conditions, or complex lifetime compilation errors during the ring buffer implementation, abandoning the dual-module single-binary approach in favor of two entirely separate applications (a Player binary and a Manager binary) communicating via a local TCP socket might become necessary to artificially separate memory concerns.  
> 3. **Adding DRM or Streaming Integrations:** This architecture assumes local, DRM-free files. If you pivot to supporting encrypted streaming services (Spotify, Apple Music), symphonia cannot decrypt Widevine or FairPlay. You would be forced to use the WebView2 EME APIs, effectively discarding the entire Rust audio engine.

#### **Works cited**

> 1. GitHub \- RustAudio/rodio: Rust audio playback library, [https\://github.com/rustaudio/rodio](https://github.com/rustaudio/rodio)  
> 2. GitHub \- ExPixel/miniaudio-rs: Rust bindings for miniaudio C library., [https\://github.com/ExPixel/miniaudio-rs](https://github.com/ExPixel/miniaudio-rs)  
> 3. CHANGELOG.md \- GStreamer/gst-plugins-rs \- GitHub, [https\://github.com/GStreamer/gst-plugins-rs/blob/main/CHANGELOG.md](https://github.com/GStreamer/gst-plugins-rs/blob/main/CHANGELOG.md)  
> 4. GitHub \- mtxmiller/LyrPlay: Squeezebox player for iOS with native, [https\://github.com/mtxmiller/LyrPlay](https://github.com/mtxmiller/LyrPlay)  
> 5. NVGT Documentation \- The Nonvisual Gaming Toolkit, [https\://nvgt.dev/docs/](https://nvgt.dev/docs/)  
> 6. Licensing, [http\://www\.un4seen.com/license.php?l=0](http://www.un4seen.com/license.php?l=0)  
> 7. pdeljanov/Symphonia: Pure Rust multimedia format demuxing, tag, [https\://github.com/pdeljanov/symphonia](https://github.com/pdeljanov/symphonia)  
> 8. feat: added support for wavpack by pierreaubert · Pull Request \#503, [https\://github.com/pdeljanov/Symphonia/pull/503/files](https://github.com/pdeljanov/Symphonia/pull/503/files)  
> 9. Implementing gapless playback \#169 \- pdeljanov Symphonia \- GitHub, [https\://github.com/pdeljanov/Symphonia/discussions/169](https://github.com/pdeljanov/Symphonia/discussions/169)  
> 10. How faithful is vorbis decoding to libvorbisfile? · pdeljanov ... \- GitHub, [https\://github.com/pdeljanov/Symphonia/discussions/435](https://github.com/pdeljanov/Symphonia/discussions/435)  
> 11. MP3: Decoding fails on joint stereo frames with different block\_types, [https\://github.com/pdeljanov/Symphonia/issues/556](https://github.com/pdeljanov/Symphonia/issues/556)  
> 12. mp3: Xing/Info tag rejected on CRC-protected frames, VBR ... \- GitHub, [https\://github.com/pdeljanov/Symphonia/issues/516](https://github.com/pdeljanov/Symphonia/issues/516)  
> 13. Best way to implement audio streaming? \#153 \- Symphonia \- GitHub, [https\://github.com/pdeljanov/Symphonia/discussions/153](https://github.com/pdeljanov/Symphonia/discussions/153)  
> 14. Releases · RustAudio/cpal \- GitHub, [https\://github.com/rustaudio/cpal/releases](https://github.com/rustaudio/cpal/releases)  
> 15. Rubato \- An asyncronous resampling library written in Rust \- GitHub, [https\://github.com/HEnquist/rubato/](https://github.com/HEnquist/rubato/)  
> 16. Event or callback for arrival/removal of devices · Issue \#373 \- GitHub, [https\://github.com/RustAudio/cpal/issues/373](https://github.com/RustAudio/cpal/issues/373)  
> 17. cpal/UPGRADING.md at master · RustAudio/cpal \- GitHub, [https\://github.com/RustAudio/cpal/blob/master/UPGRADING.md](https://github.com/RustAudio/cpal/blob/master/UPGRADING.md)  
> 18. RustAudio/cpal: Low-level cross-platform audio I/O library in Rust, [https\://github.com/RustAudio/cpal](https://github.com/RustAudio/cpal)  
> 19. RSPlayer \- Diy-friendly Hi-Fi Music Player \- GitHub, [https\://github.com/ljufa/rsplayer](https://github.com/ljufa/rsplayer)  
> 20. n1ji/honeycomb: A fast, library-focused desktop music player (Tauri), [https\://github.com/n1ji/honeycomb](https://github.com/n1ji/honeycomb)  
> 21. Gapless playback \+ moving away from WebAudio to native Rust, [https\://github.com/basharovV/musicat/discussions/6](https://github.com/basharovV/musicat/discussions/6)  
> 22. RuView Quality Engineering Analysis \- Full Report (10 documents), [https\://gist.github.com/proffesor-for-testing/02321e3f272720aa94484fffec6ab19b](https://gist.github.com/proffesor-for-testing/02321e3f272720aa94484fffec6ab19b)  
> 23. madtunebk/TempRS \- GitHub, [https\://github.com/madtunebk/TempRS](https://github.com/madtunebk/TempRS)  
> 24. tauri/crates/tauri-utils/CHANGELOG.md at dev \- GitHub, [https\://github.com/tauri-apps/tauri/blob/dev/crates/tauri-utils/CHANGELOG.md](https://github.com/tauri-apps/tauri/blob/dev/crates/tauri-utils/CHANGELOG.md)  
> 25. config.schema.json \- GitHub, [https\://github.com/tauri-apps/tauri/blob/dev/crates/tauri-schema-generator/schemas/config.schema.json](https://github.com/tauri-apps/tauri/blob/dev/crates/tauri-schema-generator/schemas/config.schema.json)  
> 26. \[feat\] Allow disable background throttling · Issue \#5250 · tauri-apps, [https\://github.com/tauri-apps/tauri/issues/5250](https://github.com/tauri-apps/tauri/issues/5250)  
> 27. Best Frontend Frameworks in 2026: A Production-Tested Guide \- Intuz, [https\://www\.intuz.com/best-frontend-frameworks/](https://www.intuz.com/best-frontend-frameworks/)  
> 28. playwire \- A Modern Cross Platform System Media Rust Crate \- GitHub, [https\://github.com/SFG5453/playwire](https://github.com/SFG5453/playwire)  
> 29. Bug: Media keys ignored when other apps register SMTC (Proof of, [https\://github.com/Stremio/stremio-shell-ng/issues/84](https://github.com/Stremio/stremio-shell-ng/issues/84)  
> 30. tcsenpai/mycelium: Simple, structured and collaboration ... \- GitHub, [https\://github.com/tcsenpai/mycelium](https://github.com/tcsenpai/mycelium)  
> 31. mvmcode/elves: You don't do the work. You have elves for that., [https\://github.com/mvmcode/elves](https://github.com/mvmcode/elves)  
> 32. GitHub \- Capataina/Mnemosyne: A local-first, AI-powered image, [https\://github.com/Capataina/PinterestStyleImageBrowser](https://github.com/Capataina/PinterestStyleImageBrowser)  
> 33. exodium/CLAUDE.md at main \- GitHub, [https\://github.com/tvollstaedt/exodium/blob/main/CLAUDE.md](https://github.com/tvollstaedt/exodium/blob/main/CLAUDE.md)  
> 34. Tauri vs Electron (2026): The Honest Comparison \- Brainsroot, [https\://brainsroot.com/blog/why-tauri-over-electron/](https://brainsroot.com/blog/why-tauri-over-electron/)

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADYAAAAVCAYAAAANfR1FAAACjElEQVR4Xu2Wz4uNURjHv0JJJAtk/M6PWJAfKWUjkRrRkB8lTJkaCqMkEslKWZkhLAxRDCU2slAWLBRhJVZmI2U75Q8Y3+/7nOOe99wzd+5779s0i/utT/ee95z3ec9zzvM85wAttdSsppFZZGLcEUh90+OH41k7yUtyhvSTOfnuTHLqNDkRd9SrGWQ3mRB31NAasiV+SE0hB8gq91+Tm0eOkIVuzHzyhmxy7f3kOzlHlsKc3EoGyBPYzjYkGerD6AY2kJPkNRkip/LdmbRIr8hwwF9yhUx2Y7aRj6TNtReTq2QR2UX2kXbYTmoBG1YRx3aQ7eQn0o7JxiPylnwgvbD3wmjQe3LMh1/q+0fRRAh6pQzX0lrUdky2UjnjpfwKHdPOXYeFrrSS3EL98xlRY+3YcvKOrHNt5dNx91/OaZebCkGvsh27C8sZheMX8gCVwiEpLBVmL8gh8hCVfNPzwiE4ldwhnyK+kl/kc6LvfPZmXqM5pkrWAXNAXCDvyYJgnLQElrP+nNIuaX5+gbWzl8leVApPIZW5Y3JEEw2LhUJuEFbOR5K+Lad8CK6HVVe1D5NuFDuOMpXpWEp+/DNY1KQUhuAkcptcdG3lnYrLXNeuW2U6don8Rv7w9uOfI/2NOARnw46K0H4XzE4hlemY7PxB3jEfijdQHU5xCEqaj46D0P4xsjFo16VGHdM9L9Ye2P3OO6DfHvIDljexFH46jEMpR3WP9I7Jhm4uK/6PiDSTPEZ15atVFc9mb5o6YWN1nfLXJd3xNAnd0iVVr2vkHjlIbpJvsGtULO2Szix/MIeSU/dh9lQ9lWOpcWMqrfAy2L1vM9IT0jMtgCadkvqVr1q0p2R1vrullsad/gEg9359JGm8pgAAAABJRU5ErkJggg==>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACsAAAAVCAYAAADfLRcdAAACXElEQVR4Xu2Wz6tOQRjHv3KVhd/yI93FayMWhBVKJHXdboqysGIrxIaFbERKNlhQ7FAW1L0SK90oPyIhxMJaWSgbfwDfr2fGnfO8Z+ac9827UL71ae47z9wz3zlnnmcG+K9/R7PITN9Z0Awyx3f2owVkK9lJVpDplWi3VpML6G1ymT1NxnygjaaRTeQ+uUn2BG6RJ2Td1NCKlpHbZKUPBOlt7yVzfQD2Um6Q9T5QklZ5kjxHtynFzpNPZI2LaYF6O8dc/3yyi1yE/d8LsqQyYkrbYC9E26hRMnOOfER+hdoKH8hZmMGoVeRxaFPJ7Chs4VdQNqutc5fs9oE67SffQ5vTPPKATJKFSf9hch3lxLqEslnpBLlGhnwg1XLymjwii10sVTSbTiqDMno0DsqojdntsDHa/1lpr/2EraykDnmD6qRq9Xsk/M6pjdm15BW68+WPtKHvkG9ki4t5Ka5x42R26NMEb8mGOCijNmYbFx4HKHGUQCWdgX2BNOtl9mVoS+rFrEplreKApgcNw+rse1Rr6SDMHvCBKGW1srv0IJWpI+QHuh80CLPZbaAycZl8QX7fqe6qqOtQUD1O1YFVEmVySW3MdtDiWTqRZEbnujezmbwjp1BfR+OX2ecDTjKrTC+VJVWBZ2jOnd8Dn8LuBPE+oLvBQ5jh9MRKpX4tUsnntQhWOT7DElN8hZmuO3w0pypTqyNXtyqtSrcs7ZulyJtMpSNSi9Sh0a/idvT3i78u3ZomYJeRfqVT9F5oB64d5Crq93WT9PWOk0Ph74FLkxwM9DrhRtgFpnhp/wWvGWzt+TBqpAAAAABJRU5ErkJggg==>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAVCAYAAAAAY20CAAAByElEQVR4Xu2WzSsFURiHX6GElIRCoiRLkpKPjFKkhC1SZOFracNKFCWy8rGxsFOysWCjJBuFpYUiFsrCRvkD+P2cme6ZmZx7j+69LOappzv3PW9zzzvnPeeOSERERKrJhP2wMjgQhwzYCJfhKmxxY2khB3bDJXgDH2C9L8NMHtyEe7AW1sET6Gg5KYUFdMFWOCd2BfAps/AjWODGJuEnnPWS0gl/1KYA5jF/VIuVwXlYpcXShm0BXLEX2AzzYamoFf0zbArgRPdF5S/ANVHtcwnHxLCJOTAgasO9wh1Y7suIkQtHgkEDNgXwiR+K6vd1mOXG2+A97HG/h3BEVV4Di+EwPIdNWo5HCRwPBg38poA32KHF2UZXouYYaidWyQ3CielUw2M4JOosJ/ycgp1eUgLYFMDVPZBwvlcA5bWPIjgdDLoUwl14ATfgKVyB2XpSHGwKIGydYL6xAE4mFNTg/mBr9Ym6qbcaiWIqgPfiP7TeFoPwWdQp5GFsoVTDAh5hQ3BA1F77gFsS27Dcg2fi/9OKu4mTDXt5G96JOlHoO7yFi7G079eNJzgj/iOyHV6LOkon3Gsepz8eo/8Rvg85sBdW+IciIpLGF+T9TrjkR7M/AAAAAElFTkSuQmCC>

[image4]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACoAAAAVCAYAAAAw73wjAAABuklEQVR4Xu2UzytFQRTHj1CiFEIRUbKyECmyebZ+ZSlE2fiRKGUhQhbq+bGQQhbKQqwspCxlociPnYViY2Or/AF8v29m3LnjvTybKXU/9em+e+Z0O++cmRGJiIgwtMN5NwhK4TRchTOwOrzsl0p4BTedeAM8gi2wDh7AdzgKM6w8L2TDdfgh4UJz4D7sg5k6VgjP4DOs1zFvdMNFeCvhQjnya/gkqpuGKfgpqqve4Mi592pFFWUXyk4vw0NRRRsmRBXKpxdYyAJslKB77h51yYJ78A22OmsJuHF74J2opB1YHsoIyIUDbjAJnRIcinQL5Z/i/uQU+Ed/EBN14mpgMeyHF7DJyjGUwGE36FAB12C+fk+nUOYewy2Y56wlYLtnRRVgw/vsVMKnks8x2GaSksDvzYnqjuG3Qtm9OFwSdRskpQiOu0FNAdyFl3ADnsMVSTEWDSdyAm8s70Xdj6/6feg7OyhyUoKG8PDFTIKBifapc+Ee45boEnW3mY/9hVQd5be5j0f0b8Mg7LDevVEm6h7l/jMF8dkLX/SaPYFH2KzzvMAbYlvUyHk3UhbB0Zsum7jtA6ySiIh/xBdmdU/79KtMaAAAAABJRU5ErkJggg==>

[image5]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEUAAAAVCAYAAAAQAyPeAAAChklEQVR4Xu2XS2sUQRSFj6ggoqIRIyjRCOIqC0UUHxsVSQIhmOBC8LHQlUbBjS5cKEFFURBBQUPAhbtAHgQfGMHHCCqKioTgTiE7t4I/QM/hdjllTVfPTGLMDPSBj+m5uTOde7rurRogV65cuepLc8k+sib8Ayy2n6yG5S0gLeRIcu00h2wml8hV0grLryupoDZykXwkX8nGvzJMW8kk+eWhXBXtJENOkiGyljSQ2+Q6me/l1bxkyl6yk5xB3BTFXpACeUrOkpV+AiznM9nmxZrJO7LHi6VJq+5CGKwFnUK2KZfDYKBzMAN8sxaTEXITtpJiquT7Z0XTMUUr7j5KTVlEBsljstSLhyr3/bOmcqbcI7fIa/Ie1m5uyLriY6aE8VDTMkVLsAs2FL+Tu7AdIU0LyeEwmKFypjwk65L3GqKPUByiKliFh8X/F1N2wZbperKCHCIvyRYvx6mRHAuDGcoyRYXLZF+nYTuS7q17vUFp8aEpeqgyVNc+u8mNlLjwt/wSzYMNM/0DvvT0HpCDKJ4J9HoCdrNKlWVKmpSvrfk4Sot3CuN6kOdhK8xHrVlIiQvtjlEtJz1hMNEy0kdewRx/Qq6guvNBzBQ9WQ3KZ7CinJwpetUD60fcFO1A2olimnL7uN6NSUtTbdUJu0m1J8mYKbqnig1NUfv8JO3Je31+gmz4k2EP8jnshJulKZsy01JR38imIK5VcA12fHdaQobJQHItqY0/kG6XRG0n40ifeb5qyhQNzzvkC4rH9x/kE+ktpqEJ1gKaBxruY2SUrPJypA7yFvab6ADsFHwU2Qc3qaZMqUZq3x2w9lSLxNpTM0i/p4SuK1HdmjKT0tzyf1zmyvUP9Rtq53pyl3aYqwAAAABJRU5ErkJggg==>

[image6]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACoAAAAVCAYAAAAw73wjAAAB9klEQVR4Xu2WzyttURTHl1DykF54JcUrTPTkRwxEDOTHCxEGMpCBUkRGUgyEkkiKwkwGilKUTHhvJKRMZCKlTA39AXy/7XPcffa55zj3yujeb3265661z73ftfdZex+RuOKKXXWAapAGEkA26AV/9EFQCugGS2AaFDnT36tEsAneDLZAhjaO13tgUlRBLOIfaNPGRK1UkGwGw2gZnIFrsA2aRBWgaxScgEwt1gP+gxwtFpG4JJyRdZBl5MJpHpSZQU00R5NrRrwcPIAWI+4rPluVYB8sgDxn2lefGS0Gd+I2ynsewZQRDysuUR04BDPgpzMdSCxsFpyCG3AESrW8bcjLqBl3iAZbwTGYAOnOdERaBWMSei7Z8fegwvreLKrBTEO+RtkcXaKqHwI/nOmoxCL15skVNbMbIAk0gldxG/I1WgsuwYCofe079AtcgQtRHe1lyCv+IX1WR+Rry94HXkQVbss2SnhdAG7Fbcg2Om7EXbKfU24d0TYS90cuq27UXvoDUZs74fWOOFexHjxbn4Fkbk2chaCqEnWPfjB0gidxnjpsMB4I+dZ3/idPKTazfoIFEm8uAbtgRUI/6ifeMyyqyH4wJ2rPHLRytljIojXuryiT5+J+H4hYNMkXh99mwkM8IGigQbxniMYLQTuokWDHc1yxo3eKuFVNeS00fwAAAABJRU5ErkJggg==>