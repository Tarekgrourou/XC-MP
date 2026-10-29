# **Architecture and Engineering Analysis for a High-Performance Local Audio Player**

## **Engine Paradigm Evaluation**

The selection of a core audio pipeline dictates the fundamental reliability, maintenance overhead, and resource consumption of any desktop media application. For a Windows-first desktop player built within a constrained development budget, the underlying architecture must minimize foreign function interface (FFI) complexity, avoid immense bundling requirements, and guarantee pristine audio delivery. The analysis evaluated pure Rust ecosystem crates, C/C++ bindings, and web-native solutions to determine the most viable path.  
The Rust audio ecosystem has matured significantly by late 2026, offering robust, memory-safe alternatives to legacy C libraries. The pure-Rust library symphonia (version 1.7.0 as of August 20261) provides a highly capable media demuxing and decoding pipeline. Symphonia strictly handles decoding, requiring a separate audio output crate. For output, the cpal crate serves as the standard pure-Rust cross-platform audio I/O library. On Windows, it natively interfaces with the Windows Audio Session API (WASAPI)1. The pairing of Symphonia and CPAL represents a highly decoupled, maintainable architecture.  
Historically, the rodio crate served as the default higher-level abstraction over CPAL. However, reliance on Rodio in 2026 introduces unacceptable project risk. The crate is reportedly undergoing a major rewrite3, leading to instability in production environments. Real-world open-source applications provide VERIFIED evidence of these limitations; the librespot project formally deprecated its Rodio backend due to its failure to honor audio output format requests, opting to interface directly with CPAL4. Another popular audio crate, kira, provides excellent performance but is explicitly designed for game audio (triggering short sound assets, physics integration, and entity-component systems) rather than streaming 100,000-track local libraries from disk5.  
Foreign Function Interface (FFI) wrappers around C/C++ libraries offer an alternative path but severely conflict with the goal of minimizing maintenance overhead. The official Rust SDKs for miniaudio were explicitly abandoned by their maintainers (Picovoice) in July 20257. Relying on unmaintained bindings introduces significant security and compilation risks. Commercial solutions like BASS and FMOD are highly capable but introduce complex, closed-source licensing requirements that complicate distribution8. Exhaustive multimedia frameworks like libmpv, libvlc, and GStreamer require bundling large dynamic libraries (DLLs) alongside the Tauri executable. This approach inflates the installer size by up to 150 MB and requires complex build scripts to orchestrate the C-dependencies, representing significant over-engineering for an audio-only desktop player.  
Executing audio directly within Tauri's WebView2 via HTML5 \<audio\> or the Web Audio API represents the lowest-friction path for UI/UX integration, as WebView2 natively supports FLAC decoding. However, the hard limits of web-native audio fundamentally contradict the project's core requirements. Standard HTML5 \<audio\> elements cannot achieve true, sample-accurate gapless playback; the browser's media pipeline inevitably introduces micro-stutters during track transitions. To achieve gapless playback in JavaScript, developers typically decode entire audio files into AudioBuffer nodes via the Web Audio API. This approach leads to catastrophic memory bloat. A single 24-bit/192kHz FLAC file can consume upwards of 300 MB of uncompressed PCM data in RAM. Queuing multiple high-resolution tracks via this method violates the performance constraints mandated for a lightweight local player. Furthermore, extracting Fast Fourier Transform (FFT) data via the Web Audio API's AnalyserNode ties audio processing to the main thread's JavaScript event loop, which can drop frames and cause audio stutter if the UI is busy rendering complex CSS or WebGL motion graphics.

## **Symphonia Capabilities and Real-World Implementation**

Symphonia represents the most viable and cost-effective decoding engine for a modern Rust application. Its format maturity is exceptionally high for the requirements of a local music manager.  
Symphonia provides mature, highly optimized pure-Rust decoders for FLAC, ALAC, WAV, MP3, AAC-LC, Vorbis, Opus, WavPack, APE, and DSD1. The library's efficacy is VERIFIED by its adoption in major open-source projects. For example, librespot successfully migrated its audio decoding pipeline from lewton to Symphonia, explicitly citing improved Vorbis sound quality and the ease of future FLAC integration4.  
Gapless playback support is fundamentally reliant on the decoder's ability to expose encoder delay and padding data. Symphonia excels in this area, actively supporting gapless playback information embedding, including parsing LAME Xing headers and Apple's iTunSMPB headers10. It must be noted, however, that Symphonia acts solely as a demuxer and decoder; it exposes the padding sample counts, but the application's audio pipeline must manually trim these leading and trailing samples before pushing the audio buffer to the OS.  
Seek accuracy varies by format within Symphonia. For deterministic container formats like MKV/WebM and FLAC, seeking is mathematically sample-accurate9. However, for lossy formats like MP3, AAC, and Vorbis, achieving sample-accurate seeking requires implementing a seek pre-roll. The application must seek to a keyframe slightly before the target timestamp and silently decode the intervening frames to correctly prime the decoder's internal state before emitting audible samples. Symphonia handles 24-bit/192 kHz FLAC decoding effortlessly on modern CPUs, scaling linearly with available hardware without introducing artificial bottlenecks.

## **CPAL, WASAPI, and the Windows Audio Stack**

Interfacing with the Windows audio subsystem via CPAL requires navigating the specific mechanics of the Windows Audio Session API (WASAPI). The application must choose between Shared Mode and Exclusive Mode, each carrying distinct architectural implications.  
WASAPI Shared Mode allows multiple applications to play audio simultaneously. The OS audio engine (the system mixer) resamples all incoming streams to a common format dictated by the user's Windows Sound Control Panel settings. If the source FLAC file is 96 kHz but the Windows mixer is set to 48 kHz, the audio must be resampled. While CPAL can submit audio to the Shared Mode mixer, relying on the OS for resampling can sometimes introduce artifacts. WASAPI Exclusive Mode bypasses the system mixer entirely, granting the application direct, bit-perfect access to the DAC at the file's native sample rate and bit depth. However, Exclusive Mode actively blocks all other applications from using that audio device11. For a mainstream desktop music player, stealing the audio device prevents browser audio and system notifications from functioning. Consequently, WASAPI Shared Mode is strictly recommended for the v1 architecture, deferring Exclusive Mode to future enthusiast-focused updates. CPAL also supports ASIO on Windows, but this requires integrating the proprietary Steinberg SDK at build time12, which violates the project's constraint for low-maintenance, automated AI-assisted builds.  
Device hot-swapping and sleep/resume states introduce complexity to the CPAL lifecycle. Windows sleep cycles or the physical disconnection of USB headphones frequently invalidate active WASAPI endpoints. CPAL handles device enumeration, but dynamic hot-swapping requires listening to OS-level device state changes. The underlying wasapi crate utilizes the IMMNotificationClient interface to detect when devices are added, removed, or changed13. The audio architecture must be designed to gracefully catch CPAL stream errors (such as DeviceNotAvailable), tear down the current audio stream, query the OS for the new default fallback device, and instantly rebuild the stream without crashing the Tauri application.

## **Reliability Architecture: Threads, Buffering, and Resampling**

To guarantee zero-stutter playback while simultaneously feeding high-resolution data to a React frontend, the engine must employ a strict triple-thread architecture. Attempting to run decoding, I/O, and OS audio callbacks on a single thread will inevitably result in buffer underruns and audio crackle when reading from slow mechanical disks or network attached storage (NAS).  
The first thread is the Tauri IPC Main Thread, which handles asynchronous commands from the React frontend (e.g., Play, Pause, Seek). It maintains application state and sends lightweight control signals over crossbeam channels to the audio backend. The second thread is the Decoder Worker Thread. This thread executes blocking disk I/O, reads the FLAC file, decodes it via Symphonia into 32-bit floating-point (f32) PCM samples, applies ReplayGain scalar multiplication, and pushes the samples into a shared buffer. The third thread is the CPAL Audio Callback Thread. This is a real-time priority thread spawned and managed by CPAL. Its singular, critical responsibility is to pop samples from the shared buffer and deliver them to the WASAPI endpoint within strict microsecond deadlines.  
The communication mechanism between the Decoder Thread and the CPAL Thread must be entirely lock-free. Mutexes must never be used in the CPAL callback thread; if the OS scheduler preempts the Decoder Thread while it holds a lock, the CPAL thread will block, starving the DAC and producing audible glitches. A lock-free ring buffer (such as those provided by rtrb or crossbeam-queue) ensures the audio thread never waits14. The ring buffer should be sized to hold a minimum of 1.5 to 2 seconds of audio data to provide resilience against slow disk spin-ups or saturated NAS network links.  
Gapless playback is achieved through proactive pre-buffering within the Decoder Thread. The thread must monitor the remaining samples in the current track. Approximately five seconds before the track concludes, the Decoder Thread must seamlessly open the next file in the queue, decode its headers, trim any required padding10, and begin pushing the new samples into the lock-free ring buffer directly contiguous with the tail samples of the current track. Crossfading follows the same logic, but rather than appending samples sequentially, the Decoder Thread mathematically mixes (adds and averages) the tail of track A with the head of track B inside a temporary buffer before pushing to the ring buffer.  
Because WASAPI Shared Mode operates at a fixed system sample rate, the application must perform high-quality software resampling when the source file rate diverges from the OS rate. The pure-Rust rubato crate is the VERIFIED standard for this task15. Rubato offers multiple resampling algorithms. For static sample rate conversion (e.g., converting a 96 kHz file to a 48 kHz output), the synchronous Fft resampler is the optimal choice; it is highly efficient and avoids the immense CPU overhead of sinc interpolation15. The asynchronous Async::new\_sinc mode should be strictly avoided for local file playback, as it is designed for real-time clock-drift correction and consumes excessive CPU cycles15.

## **Visualizer Data Path and DSP Integration**

Extracting Fast Fourier Transform (FFT) data for the React visualizers must be accomplished without disturbing the real-time audio pipeline. Computing FFTs in JavaScript via the Web Audio API is highly inefficient. Instead, the architecture must compute the FFT natively in Rust and stream the results to the frontend.  
The Decoder Thread taps the f32 audio stream just before the samples are pushed into the lock-free ring buffer. Using a crate like rustfft or the pure-Rust FFT implementations in rubato-fft-node16, the engine calculates the magnitude spectrum of the signal. A Hann or Hamming window function must be applied to the sample chunk prior to the FFT calculation to mitigate spectral leakage16.  
Synchronization between the audio heard by the user and the visualizer rendered on screen is a critical UX challenge. Because the lock-free ring buffer holds up to two seconds of audio, FFT frames calculated at the decoder level represent audio that is in the "future" relative to the speakers. Every computed FFT frame must be tagged with a precise playback timestamp. The engine stores these frames in a short-lived memory queue. A lightweight loop queries CPAL for the exact playback cursor position, retrieves the matching FFT frame from the queue, and transmits it to the frontend. To bypass the serialization overhead of Tauri's standard IPC events, sending high-frequency quantized frequency bins (e.g., 64 floats at 60 FPS) via a local WebSocket server bound strictly to 127.0.0.1 provides a zero-overhead data path directly to the WebGL canvas.

## **Engine Scope and Cost-Benefit Analysis**

Defining the exact scope of the v1 audio engine is paramount for budget preservation. The engineering cost of certain audiophile features vastly outweighs their utility for a mainstream application.  
Multi-channel audio (e.g., 5.1 FLAC files) should be aggressively downmixed to stereo in the Decoder Thread. Attempting to negotiate multi-channel output configurations with WASAPI introduces edge cases that crash the CPAL initialization process across different hardware vendors. Similarly, DSD decoding (.dsf / .dff files) should be deferred. While Symphonia supports DSD9, decoding 1-bit high-frequency signals into standard PCM for WASAPI Shared Mode requires heavy low-pass filtering and decimation, consuming massive CPU resources for a format that constitutes less than 1% of typical user libraries.  
CUE sheets, which divide a single large audio file into multiple logical tracks, break the standard 1:1 file-to-track paradigm assumed by most modern library scanners. Supporting CUE sheets requires complex virtual file mapping within the SQLite database and specialized byte-offset seeking within the audio pipeline. This feature should be firmly rejected for the v1 build. ReplayGain, conversely, is trivial to implement. By parsing the REPLAYGAIN\_TRACK\_GAIN tag as a float and applying it via basic scalar multiplication on the f32 buffer, the player gains massive UX value with minimal code.

## **Deliverables: Engine Decision Matrix and Test Plan**

| Architecture Approach | Reliability | UI/Motion Performance | Gapless FLAC Support | Maintenance & Implementation Cost | Verdict |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Hybrid (libmpv / GStreamer)** | High | Moderate (IPC overhead) | Excellent | High (C-dependencies, bundling bloat) | Reject |
| **Web Native (HTML5 / WebAudio)** | Low | High | Poor | Low | Reject |
| **Rust Pipeline (Symphonia \+ CPAL)** | High | High (Decoupled from UI) | Excellent | Moderate | **Recommended** |

**Engine Test Plan:**

> 1. **Gapless Verification:** Generate a continuous 440Hz sine wave. Split the file precisely at a zero-crossing into two FLAC files. Enqueue both files. Any audible click, pop, or micro-stutter indicates a failure in Symphonia padding extraction or a stall in the lock-free ring buffer.  
> 2. **Device-Switch Resilience:** Initiate playback through standard desktop speakers. Open the Windows Sound Control Panel and disable the active device. The CPAL callback must catch the invalidation error, instruct the main thread to pause playback, and successfully re-initialize the stream when the device is re-enabled, without requiring an application restart.  
> 3. **Format and Frequency Stress:** Procure the official xiph.org FLAC test vectors to verify standard decoding. Source a 24-bit/192kHz test file and observe CPU utilization during the rubato Fft downsampling process to 48kHz WASAPI Shared Mode.

## **Metadata Standards and Tag Parsing Realities**

A local library containing 100,000+ tracks introduces edge cases that easily crash naive metadata parsers. Modern music managers must gracefully handle decades of shifting, poorly implemented tagging standards.  
FLAC files utilize Vorbis comments, which are well-structured and inherently support multiple values by simply repeating the key (e.g., multiple ARTIST= fields). MP3 files rely on ID3v2 tags, which are historically chaotic. ID3v2.3 separates multiple artists within a single string using a forward slash (/) or a semicolon (;). ID3v2.4 officially standardized the use of null bytes (\\0) for field separation. MP4 files (.m4a) use Apple QuickTime atoms. The library scanner must normalize these disparate systems. Multi-value fields must be explicitly parsed and split into standardized arrays before insertion into the SQLite database, preventing queries from failing on strings like "Artist A / Artist B".  
Real-world tag messiness requires defensive programming. Older ID3 tags often contain text encoded in Shift-JIS or Windows-1252 rather than UTF-8. If the metadata crate cannot transcode the text, it should gracefully fallback to replacing invalid characters rather than panicking the entire scan thread. Malformed dates (e.g., tags containing "2023-04" or text strings instead of a valid four-digit year) must be caught and defaulted to NULL to prevent database type insertion errors.

## **Album Identity and Deterministic Grouping**

To emulate the robustness of industry-standard managers like foobar2000, MusicBee, and beets, track grouping must follow a strict, deterministic cascade. Naively grouping tracks merely by the Album string results in catastrophic library corruption; tracks from "Greatest Hits" by twenty different artists will merge into a single anomalous album, and multi-disc box sets will overwrite each other.

| Grouping Priority | Logic Cascade | Failure Mode Handled |
| :---- | :---- | :---- |
| **1\. MusicBrainz Release Group ID** | If MUSICBRAINZ\_RELEASEGROUPID exists, it acts as the absolute source of truth, grouping tracks regardless of string variations. | Bypasses typos, varying capitalization, and complex box sets. |
| **2\. Album Artist \+ Album \+ Year** | The standard fallback. Groups by the overarching album artist. | Fixes "Greatest Hits" merging by separating them by the distinct Album Artist. |
| **3\. Compilation Flag \+ Album \+ Year** | Checks for TCMP (Compilation) tag or Album Artist \= "Various Artists". | Groups Various Artists compilations correctly even if individual track artists differ. |
| **4\. Folder Path** | If tags are entirely missing, group all files residing in the same physical directory. | Salvages untagged downloaded folders. |

Failure to account for Disc Number and Track Number during the sorting phase is a common error. Tracks tagged as 1/12 for Disc 1 and 1/10 for Disc 2 must be sorted by Disc *then* Track, otherwise the album view will interleave the tracklists.

## **Tag Libraries, Write Safety, and Concurrency**

Modifying local audio files is an inherently dangerous operation that risks permanent data loss if interrupted. The architecture must prioritize absolute write safety.  
The pure-Rust crate lofty (v0.25+ as of late 202617) is the premiere choice for reading and writing audio metadata. It completely avoids the C-bindings associated with older tools like TagLib, fitting the project's low-maintenance constraint perfectly.  
Writing tags to FLAC files introduces specific structural risks. FLAC files contain a pre-allocated padding block located immediately after the metadata headers19. This space is reserved so that small tag modifications do not require rewriting the entire audio payload. However, if a user embeds a large high-resolution PNG album cover, the new metadata may exceed the padding block's capacity. When this occurs, the library (e.g., lofty) must rewrite the entire file, byte-for-byte, to shift the audio frames down19.  
If the application writes directly to the source file and a power failure or OS crash occurs during this operation, the FLAC audio payload is irreversibly destroyed. To guarantee write safety, the application must employ an atomic write pattern:

> 1. Copy the source file to a temporary file in the same directory (e.g., song.flac.tmp).  
> 2. Instruct lofty to write the new tags to the .tmp file.  
> 3. Call fsync to ensure the OS flushes the temporary file's buffers to physical disk.  
> 4. Execute an OS-level atomic rename operation, overwriting the original file with the .tmp file.

Furthermore, Windows strictly enforces file locking. If the CPAL audio engine is currently reading a track for playback, the OS places a read-lock on the file. The Library Manager must attempt a non-blocking write check; if an Access Denied error occurs, the UI must immediately inform the user that the file is in use, rather than retrying indefinitely or crashing.

## **Scanning Performance and File Watching Anomalies**

Saturating disk I/O without freezing the React UI is mandatory for a 100,000-track library. The initial cold scan should utilize a parallel thread pool (e.g., the rayon crate) scaled to the logical CPU core count.  
Subsequent boots rely on incremental detection. A full re-read of 100k FLAC headers takes minutes, even on modern NVMe storage. The SQLite database must store the exact mtime (modified time) and file size (in bytes) for every tracked file. Upon application boot, the scanner rapidly stats the directory tree; it only passes files to lofty for metadata extraction if their mtime or size diverges from the database record.  
Real-time file watching on Windows relies on the notify crate, which wraps the native ReadDirectoryChangesW Windows API21. The implementation of this API is notoriously fraught. ReadDirectoryChangesW utilizes a fixed-size kernel buffer (typically 64KB) allocated from non-paged memory to queue filesystem events23. If a user pastes a massive folder containing thousands of tracks, or if the application is monitoring a network attached storage (SMB/CIFS) drive, the event queue rapidly outpaces the application's ability to read it, and the kernel buffer overflows21.  
When this overflow occurs, Windows drops all pending change notifications. The notify crate detects this state and emits a specific EventKind::Other (or Rescan) event21. This is not a bug in notify, but a fundamental limitation of the OS. The architecture must explicitly catch this exact error variant, debounce it (to wait for the heavy disk I/O to subside), and seamlessly trigger a silent, background incremental directory scan to reconcile the database state. Failure to implement this fallback will result in the Library Manager permanently losing track of files moved by the user.

## **SQLite Design: FTS5, Schemas, and Pagination**

A well-tuned SQLite database is the backbone of the application. The architecture must balance highly normalized structural data with denormalized text required for instant search.  
**Schema Design and Concurrency:**

* tracks: id, path (UNIQUE), mtime, size, title, track\_num, disc\_num, duration, album\_id.  
* albums: id, title, album\_artist\_id, year, artwork\_cache\_path.  
* artists: id, name.  
* track\_artists: A join table resolving the many-to-many relationship for multi-artist tracks.

The database must be initialized with Write-Ahead Logging enabled (PRAGMA journal\_mode=WAL)25. WAL mode is strictly required because it allows concurrent readers and writers. Without WAL, if the background scanner is inserting new metadata into the database, the React frontend will hang entirely when attempting to query the tracklist for UI rendering.  
**Full-Text Search (FTS5):** For instantaneous, typo-tolerant search across 100k tracks, standard LIKE '%query%' statements are unacceptably slow. The application must leverage SQLite's FTS5 extension. A virtual FTS5 table should mirror the primary data, combining track\_id, title, artist\_name, and album\_title. Crucially, the FTS5 table must be configured to use the trigram tokenizer rather than the default stemmer25. Trigram tokenization breaks text into 3-character sequences, which allows the search engine to perform lightning-fast substring matching and gracefully handle diacritics and non-Latin scripts. Because contentless FTS5 tables do not support the REPLACE conflict resolution mechanism, the application logic must manually execute a DELETE followed by an INSERT against the FTS5 index whenever a track's metadata is updated25.  
**Pagination Mechanics:**  
Rendering massive lists in React requires UI virtualization (e.g., @tanstack/react-virtual). Virtualization relies on fetching small data chunks rapidly as the user scrolls. Using standard OFFSET pagination (e.g., LIMIT 50 OFFSET 50000\) forces SQLite to sequentially scan and discard 50,000 rows before returning the target data, causing severe scroll lag. The API must implement Keyset Pagination. By tracking the last seen identifier (e.g., WHERE title \> ? AND id \> ? ORDER BY title, id LIMIT 50), SQLite utilizes its B-tree indexes to jump instantly to the exact row, maintaining sub-millisecond query times regardless of library size.

## **Artwork Management and Enrichment Sources**

Parsing embedded artwork in real-time during UI scrolling will instantly saturate the disk and freeze the render loop. During the initial library scan, the rust backend must utilize the image crate to extract embedded images from the audio files. These raw images should be decoded, downscaled to a standard UI resolution (e.g., 512x512 pixels), re-encoded into the highly efficient WebP format, and saved to a dedicated local application cache folder. The SQLite albums table only stores the filepath to this cached WebP.  
Dominant-color extraction—used to dynamically colorize the UI background based on the current album—should run asynchronously against the small, cached WebP image rather than the original massive embedded PNG, ensuring the extraction completes in milliseconds.  
For external metadata enrichment (e.g., querying MusicBrainz or the Cover Art Archive), the application must implement a strict token-bucket rate limiter. MusicBrainz enforces a strict limit of 1 request per second globally per IP address. Exceeding this limit results in temporary IP bans. The Rust backend must queue all outgoing HTTP requests through this centralized rate limiter, prioritizing user-initiated explicit searches over background batch enrichment.

## **Listening History and Analytics Event Model**

Implementing listening history requires modeling events rather than simply incrementing integers. Naively updating a play\_count column destroys temporal data, preventing the generation of analytics like "Top tracks this month."  
The schema requires a play\_history table: id, track\_id, timestamp. To maintain compatibility with standard import/export formats like ListenBrainz and Last.fm, the application must adhere to their official scrobble thresholds: a track counts as "played" only if the user listens to exactly 50% of its total duration, or 4 minutes of uninterrupted playback, whichever occurs first. If a user skips a track before this threshold is met, the event is discarded. The React frontend tracks the listening session and issues an IPC command to the Rust backend to insert the row into play\_history the moment the threshold is crossed.

## **Deliverables: Library Manager Specifications and Benchmarks**

**Realistic Benchmark Targets:**  
*(Assuming modern NVMe SSD and 8-core CPU logic)*

| Operation | Target Time (100k Tracks) | Architectural Requirement |
| :---- | :---- | :---- |
| **Initial Cold Scan** | \< 45 Seconds | rayon thread pool, lofty header parsing, batched SQLite inserts. |
| **Incremental Boot Scan** | \< 3 Seconds | OS-level stat for mtime verification against the database. |
| **FTS5 Trigram Search Query** | \< 15 Milliseconds | Dedicated FTS5 virtual table bypassing standard LIKE queries. |
| **UI Virtualized Scroll Fetch** | \< 5 Milliseconds | Keyset pagination using indexed WHERE clauses (no OFFSET). |

**Edge Case Testing Roster:**

> 1. *Multi-Value Splitting:* Scan an ID3v2.3 MP3 file with Artist tagged as "AC/DC / Iron Maiden". Ensure the database splits this on the / but handles the internal / in "AC/DC" correctly (this requires advanced delimiter logic or preferring ID3v2.4).  
> 2. *File Lock Violation:* Play a track. Simultaneously attempt to write a new tag to that track via the UI. Ensure a clean File In Use exception is raised to the user without crashing the playback thread.  
> 3. *Buffer Overflow Simulation:* Over a mapped network drive, run a script that creates 10,000 empty .flac files in 2 seconds. Verify that the notify crate catches the EventKind::Other overflow, and the application recovers by quietly running an incremental database rescan.

## **Implications for my project**

Based on the synthesis of late-2026 software constraints, hardware realities, and the mandatory AI-coding budget, your initial assumption of a Windows-first, Tauri 2, Rust (symphonia \+ cpal), and SQLite FTS5 stack is **highly accurate, thoroughly validated, and architecturally sound.**  
To succeed with minimal rework, you must aggressively defend architectural boundaries. The Rust backend must possess absolute ownership of the filesystem, the database, and the real-time audio thread. The React frontend should act strictly as a decoupled display terminal—transmitting user intent (Play, Pause, Search) and receiving state (Current Time, Playback Status, FFT floats). Do not attempt to script audio buffering, resampling, or tag writing in JavaScript.  
Furthermore, minimizing complexity requires strategic feature deferral. Explicitly instruct your AI coding assistant to avoid WASAPI Exclusive mode and DSD parsing for the v1 release. The engineering cost of handling exclusive device hijacking and decimation filters heavily outweighs their value for a solo developer. Crucially, you must explicitly prompt your AI assistant to handle the notify crate's EventKind::Other buffer overflow on Windows. Without this specific implementation, your library manager will permanently desynchronize whenever the user performs bulk file operations.

## **What would change this conclusion**

The conclusions drawn in this report are optimized for a desktop-centric, maintenance-averse environment. Two primary shifts in scope would invalidate this architecture:

> 1. **A Pivot to Mobile Parity (iOS/Android):** While CPAL and Symphonia function on mobile platforms, Tauri's mobile bindings introduce heavy IPC overhead. Attempting to run a highly active real-time CPAL thread alongside Tauri's WebView on a mobile device may lead to severe battery drain and thermal throttling. In a mobile-first paradigm, it is vastly more efficient to push audio decoding and playback logic entirely into the native OS layers (e.g., ExoPlayer on Android or AVFoundation on iOS) rather than executing a pure-Rust pipeline.  
> 2. **Audiophile Demand for Exclusive DAC Control:** If user feedback demands bit-perfect DSD passthrough and WASAPI Exclusive mode functionality, the pure-Rust pipeline becomes exponentially more complex. CPAL's exclusive mode is notoriously brittle regarding sample-rate negotiation across diverse DAC hardware. Meeting this demand would require abandoning the pure-Rust pipeline to wrap a mature C-library like libmpv or GStreamer, sacrificing memory safety and installer size for established, battle-tested exclusive-mode audio pipelines.

#### **Works cited**

> 1. Rust Web Audio API \- Lib.rs, [https\://lib.rs/crates/web-audio-api](https://lib.rs/crates/web-audio-api)  
> 2. listening-to-the-radio-with-rust/research/Rust Crates.md at ... \- GitHub, [https\://github.com/t-eckert/listening-to-the-radio-with-rust/blob/main/research/Rust%20Crates.md](https://github.com/t-eckert/listening-to-the-radio-with-rust/blob/main/research/Rust%20Crates.md)  
> 3. What's everyone working on this week (41/2026)? : r/rust \- Reddit, [https\://www\.reddit.com/r/rust/comments/1wz6czn/whats\_everyone\_working\_on\_this\_week\_412026/](https://www.reddit.com/r/rust/comments/1wz6czn/whats_everyone_working_on_this_week_412026/)  
> 4. Releases · librespot-org/librespot \- GitHub, [https\://github.com/librespot-org/librespot/releases](https://github.com/librespot-org/librespot/releases)  
> 5. Kira — Rust audio library // Lib.rs, [https\://lib.rs/crates/kira](https://lib.rs/crates/kira)  
> 6. poro/nAIVE \- GitHub, [https\://github.com/poro/nAIVE](https://github.com/poro/nAIVE)  
> 7. GitHub \- Picovoice/picovoice: On-device voice assistant platform, [https\://github.com/picovoice/picovoice](https://github.com/picovoice/picovoice)  
> 8. Authoring automation \- Browse — Game Audio Tools Index, [https\://gameaudiotools.dev/browse/?category=authoring](https://gameaudiotools.dev/browse/?category=authoring)  
> 9. GitHub \- M0Rf30/rmpd: Modern, high-performance MPD (Music, [https\://github.com/M0Rf30/rmpd](https://github.com/M0Rf30/rmpd)  
> 10. oxiaudio/TODO.md at master \- GitHub, [https\://github.com/cool-japan/oxiaudio/blob/master/TODO.md](https://github.com/cool-japan/oxiaudio/blob/master/TODO.md)  
> 11. conversationaly/CLAUDE.md at main \- bykof \- GitHub, [https\://github.com/bykof/conversationaly/blob/main/CLAUDE.md](https://github.com/bykof/conversationaly/blob/main/CLAUDE.md)  
> 12. ModulaserApp/laser-dac-rs \- GitHub, [https\://github.com/ModulaserApp/laser-dac-rs](https://github.com/ModulaserApp/laser-dac-rs)  
> 13. Wasapi bindings for Rust \- Audio \- Lib.rs, [https\://lib.rs/crates/wasapi](https://lib.rs/crates/wasapi)  
> 14. phosphor-tui \- crates.io: Rust Package Registry, [https\://crates.io/crates/phosphor-tui](https://crates.io/crates/phosphor-tui)  
> 15. rubato \- Rust \- Docs.rs, [https\://docs.rs/rubato](https://docs.rs/rubato)  
> 16. rubato-fft-node-darwin-arm64 \- npm.io, [https\://npm.io/package/rubato-fft-node-darwin-arm64](https://npm.io/package/rubato-fft-node-darwin-arm64)  
> 17. sonora/THIRD-PARTY.md at main \- GitHub, [https\://github.com/sonorahq/sonora/blob/main/THIRD-PARTY.md](https://github.com/sonorahq/sonora/blob/main/THIRD-PARTY.md)  
> 18. Audio — list of Rust libraries/crates // Lib.rs, [https\://lib.rs/multimedia/audio](https://lib.rs/multimedia/audio)  
> 19. GitHub \- craft-and-code/FlacCompagnon: A Rust-powered cross, [https\://github.com/craft-and-code/FlacCompagnon](https://github.com/craft-and-code/FlacCompagnon)  
> 20. Changelog \- kōan, [https\://koan.rocks/docs/changelog/](https://koan.rocks/docs/changelog/)  
> 21. File watching in Rust desktop apps with notify \- gpui-starter, [https\://gpui-starter.freeoxide.com/blog/integrating-notify-rust-file-watching/](https://gpui-starter.freeoxide.com/blog/integrating-notify-rust-file-watching/)  
> 22. c\# \- FileSystemWatcher and windows 7 \- Stack Overflow, [https\://stackoverflow.com/questions/3250343/filesystemwatcher-and-windows-7](https://stackoverflow.com/questions/3250343/filesystemwatcher-and-windows-7)  
> 23. FileSystemWatcher across the network \- PC Review, [https\://www\.pcreview.co.uk/threads/filesystemwatcher-across-the-network.2909709/](https://www.pcreview.co.uk/threads/filesystemwatcher-across-the-network.2909709/)  
> 24. change notifications over SMB from Windows, macOS and Linux, [https\://github.com/borco/rehuco/issues/502](https://github.com/borco/rehuco/issues/502)  
> 25. SQLite FTS5 Extension, [https\://sqlite.org/fts5.html](https://sqlite.org/fts5.html)