# PULSE: Native iOS vs. Web — An Honest Engineering Analysis

*Analysis only. No code was scaffolded. Written 2026-07.*

PULSE today is a browser app built on Tone.js / Web Audio / Web MIDI: step sequencer, synth instruments, live mic → vocal FX rack (reverb/echo/doubler/telephone/grit/pitch/sweep) with tap-latch and momentary "throw" FX, NL beat generation, a deep genre-preset library, arrangement (pattern banks + timeline, key-locked genre switching), sample import, WAV export, with full multitracking planned. The owner **performs over these beats live** — sings, tells stories, punches FX in and out — and wants it to feel pro.

That last sentence is the whole analysis. PULSE is really two products sharing a UI:

1. **A beat-making/ideation tool** — the web is genuinely great at this.
2. **A live vocal performance + tracking instrument** — the web is structurally, not incidentally, bad at this.

The line between "keep on web" and "go native" falls almost exactly on that boundary.

---

## 1. Where the web genuinely limits PULSE

### 1.1 Latency — the one that actually hurts

The killer feature is singing live through the FX rack. That means **input monitoring**: mic → DSP → headphones, in real time. Round-trip latency budget for a vocalist to feel "in the track" is roughly **≤10–12 ms**; past ~20 ms doubling/echo throws feel detached, and past ~30 ms singing in time gets actively hard.

- **Web Audio round-trip** is typically **30–80+ ms**: browser input buffering + `getUserMedia` processing (even with `echoCancellation`/`autoGainControl`/`noiseSuppression` forced off, which not all browsers honor) + the audio graph quantum (128 samples, ~2.7 ms — fine) + **output buffering the browser controls and you don't**. `latencyHint: "interactive"` is a *hint*, not a contract. There is no ASIO/JACK equivalent, no exclusive device mode, no way to request a 64-sample I/O buffer.
- **Native iOS (Core Audio / AVAudioEngine)** with `AVAudioSession` set to `.playAndRecord`, `.measurement`/`.voiceChat`-free config and `setPreferredIOBufferDuration(0.005)` routinely delivers **~6–12 ms round-trip** on wired output. This is why every serious mobile vocal FX app (VoiceJam-type apps, ToneStone, Koala, GarageBand's audio recorder with monitoring) is native.
- Caveat that cuts both ways: **Bluetooth headphones ruin latency on both platforms** (AAC/SBC adds 100–200 ms). Native doesn't fix physics; it fixes everything else. Wired/USB-C monitoring on native iOS is genuinely gig-ready; on web it's marginal on a good day.

**Tone.PitchShift specifically**: it's a granular pitch shifter with an inherent window delay (tens of ms) *on top of* I/O latency. Fine as a "throw" effect on playback; not viable for monitored pitch FX on your own voice in-browser.

### 1.2 No real-time pitch correction worth shipping

"Real autotune" (Antares-style retune: robust pitch detection + formant-preserved PSOLA/phase-vocoder resynthesis, or an ML model like CREPE/DDSP-style) needs low, *deterministic* per-block compute. In the browser you get AudioWorklet + WASM (+SIMD), which is workable for detection, but: no guaranteed real-time thread priority, GC pauses in the main/worker realm can starve the graph, SharedArrayBuffer requires COOP/COEP headers, and WebGPU/WebNN inference is nowhere near reliably real-time-safe. On iOS: **Core ML on ANE** or plain vDSP/Accelerate C++ in the render callback, real-time thread guaranteed by Core Audio. This feature is effectively native-only if it has to run on the monitored path.

### 1.3 Everything else, itemized

| Limitation | Web reality | Native iOS reality |
|---|---|---|
| **Background / lock screen** | iOS Safari suspends the tab; audio dies when you switch apps or the screen locks. Fatal for "play the beat while I walk around the room and sing." | `AVAudioSession` + background-audio entitlement: playback and even recording continue locked; Now Playing / lock-screen controls via `MPNowPlayingInfoCenter`. |
| **Tab/thermal throttling** | Backgrounded tabs get timer throttling; long sessions can be deprioritized; no control over sample-rate/route changes mid-set. | You own the session; route-change and interruption notifications let you recover gracefully. |
| **AUv3 / plugin hosting** | Nonexistent. No third-party FX or instruments, ever. | AUv3 hosting via `AVAudioUnitComponentManager` — users bring their own compressors, autotune (e.g., commercial AU plugins), synths. |
| **MIDI** | Web MIDI: no iOS Safari support at all (as of 2026, still gated/absent), flaky Bluetooth MIDI elsewhere, no virtual ports. | Core MIDI: Bluetooth LE MIDI pairing built-in, virtual endpoints, network MIDI, rock-solid. |
| **Ableton Link** | Not available to browsers (UDP multicast; no socket access). | LinkKit drops in; sync with every other Link app on the Wi-Fi. Big deal for performing with other people/gear. |
| **Mic permission friction** | Re-prompted per origin, sometimes per session; iOS Safari drops mic on backgrounding; users must tap to unlock the AudioContext every load. | Ask once. Session persists. |
| **Multitrack recording reliability** | `MediaRecorder`/Tone.Recorder gives you lossy-or-WAV blobs with no sample-accurate alignment guarantees across takes; dropped-buffer glitches are silent failures. | `AVAudioFile` writes from the render thread; sample-accurate punch-in/out; simultaneous file writes for N tracks is routine. |
| **Storage** | OPFS/IndexedDB is workable but quota-capped, evictable under storage pressure, and awkward for multi-GB stem libraries. Export = download-a-blob. | Real files, Files-app integration, iCloud Drive, share sheet. |
| **Inter-app audio / export flow** | Copy a WAV out via download. | Share sheet straight to Ableton Note/Logic/Koala/DAW, AUv3-as-instrument inside other hosts, audio route sharing. |
| **Offline** | PWA offline is doable but fragile on iOS (cache eviction). | Just works. |
| **Discovery & monetization** | SEO + links; payments via Stripe (better margin!). | App Store search intent for "vocal fx" / "beat maker" is huge; IAP/subscription infrastructure free; 15–30% tax. |

Honest counterpoint: several of these have *partial* web mitigations (AudioWorklet + WASM DSP, OPFS, PWA install). None of them mitigate the monitoring-latency problem, which is the product's core.

---

## 2. Which PULSE features benefit from native — ranked

| # | Feature | Native win | How, concretely |
|---|---|---|---|
| 1 | **Live vocal monitoring + punch-in FX rack** | Existential. This is the difference between a toy and an instrument. | `AVAudioEngine` input node → custom DSP (C++/Accelerate or AudioKit nodes) → main mixer; `AVAudioSession(.playAndRecord)`, preferred buffer 128 samples @48k (~2.7 ms/block), wired monitoring. Throw/latch FX = parameter ramps on the render thread, sample-accurate. |
| 2 | **Real autotune / pitch correction** | Only shippable natively (on the live path). | Pitch tracking via YIN/pYIN in vDSP or a small Core ML model on the Neural Engine; correction via TD-PSOLA in a C++ AudioUnit. Formant preservation is table stakes for the "pro" feel. |
| 3 | **Multitracking: record + play many audio layers** | Reliability and alignment. | `AVAudioEngine` with N `AVAudioPlayerNode`s + tap-to-`AVAudioFile` recording; sample-accurate scheduling (`play(at: AVAudioTime)`); punch-in against host time. Disk streaming for long stems. |
| 4 | **Background / lock-screen operation** | Removes the single most annoying daily failure ("my beat stopped"). | Background audio entitlement + `MPRemoteCommandCenter`. Trivial natively, impossible on web. |
| 5 | **Bluetooth / hardware MIDI** | Pads and keyboards become first-class. | Core MIDI + `CABTMIDICentralViewController` for BLE pairing. Map pads to throw-FX triggers. |
| 6 | **AUv3 hosting** | Ecosystem leverage: users add pro FX you didn't build. | Host AUv3 in the FX rack slots; also *ship PULSE's FX rack as an AUv3* so it works inside Logic/GarageBand — great marketing. |
| 7 | **Ableton Link** | Perform with other musicians/apps in sync. | LinkKit; quantized transport start against Link beat time. |
| 8 | **Share-sheet / inter-app export** | Frictionless "get my track into Logic." | `UIActivityViewController`, Files, AUv3, maybe AudioBus. |
| 9 | Visualizations at 120 fps | Nice, not necessary. | Metal/MetalKit; but Canvas/WebGL is honestly fine. |

Items 1–4 are the product. Items 5–8 are the moat.

---

## 3. What's fine — or better — on the web

- **Beat sketching & the step sequencer.** Sequenced (non-monitored) audio doesn't care about round-trip latency; Tone.Transport lookahead scheduling is sample-accurate on output. The current sequencer is not the problem.
- **NL beat generation.** It's a server/LLM round-trip either way. Web is arguably better: iterate on prompts and preset logic without App Store review cycles.
- **Genre preset library & sound design iteration.** Shipping a new Jersey Club or amapiano preset is a deploy, not a release. Keep the preset schema data-driven so both platforms consume the same JSON.
- **Arrangement/timeline editing.** Mouse + big screen beats a phone for timeline work. Key-locked genre switching is transport/music-theory logic, platform-agnostic.
- **Sharing.** A link that plays a beat instantly, no install, is a distribution superpower no app has. "Listen to what I made" → URL. Keep this forever.
- **Cross-platform reach + instant access.** Android, desktop, a friend's laptop. Zero-install demo is your top-of-funnel.
- **The design/UX layer.** The visual identity, the pad UI, the FX-throw interaction design — all portable ideas. Prototyping interaction on web is faster than SwiftUI iteration.
- **WAV export of *composed* (non-live) material.** `OfflineAudioContext` renders faster than real time and is deterministic. Offline bounce on web is actually good.

The web version isn't a lesser PULSE — it's the **writing room**. Native is the **booth and the stage**.

---

## 4. Recommended architecture

**Recommendation: hybrid, two deliberately different apps sharing a project format and preset library — not one codebase contorted to do both.**

- **PULSE Web (exists):** ideation, sequencing, NL generation, arrangement, sharing, playback of published tracks. This is also the marketing site.
- **PULSE iOS (new): the performance + tracking app.** Fully native audio: Swift/SwiftUI shell, `AVAudioEngine` graph, custom C++/Accelerate DSP for the FX rack and (later) pitch correction, Core MIDI, LinkKit, AUv3 hosting, background audio. Projects sync both ways via a **shared JSON project schema** (patterns, kit/preset refs, arrangement, FX-rack state) + audio stems as files.

### Options considered

| Approach | Verdict | Why |
|---|---|---|
| **Capacitor/Cordova wrapper around current app** | **No.** | You inherit WKWebView's Web Audio stack — *same latency, same suspension behavior* — plus App Store risk for thin wrappers. You'd ship the web app's weaknesses with an icon. Only worth it if the goal were merely App Store presence. |
| **React Native / Expo + native audio module** | Meh. | The audio engine is native anyway, so RN only buys you UI reuse — but PULSE's UI is a performance surface (pads, latching, gesture timing) where the JS bridge and gesture latency show. Acceptable if you have a JS-heavy team and no Swift skills; otherwise it's complexity without saving the hard part. |
| **Shared Rust/C++ DSP core → WASM + iOS static lib** | **Attractive, but not yet.** | The "one DSP, two platforms" dream is real (see Surge, Pd/libpd, RNBO exports). But today the web DSP is Tone.js graphs, not custom kernels — there's nothing to share yet. Adopt this **when** you build custom DSP (autotune, doubler, saturation): write those kernels in C++/Rust once, run as AudioWorklet-WASM on web (non-monitored/preview) and in the AU render callback on iOS. Don't rewrite the whole engine into a shared core up front; that's a 6-month detour before any user feels a difference. |
| **Tauri-mobile** | No. | Same WebView-audio problem as Capacitor, smaller ecosystem. |
| **Fully native Swift + AVAudioEngine (+ AudioKit)** | **Yes — for the iOS app.** | AudioKit gets you a sequencer, mixer, taps, and many FX in weeks, with escape hatches to raw AUs/C++ where it's not good enough (it won't give you pro autotune — plan to hand-roll or license that DSP). SwiftUI for everything except the latency-sensitive pad surface (UIKit gesture recognizers there). |

### The shared contract (this is the actual architecture decision)

Version a **project schema** now, while it's cheap: tempo/key, pattern banks, step data, synth/kit preset IDs, arrangement timeline, FX-rack config (effect type + params + latch state), stem references. Web and iOS are then just two renderers of the same document. This is what makes "sketch on the train in the browser, track vocals at home on the phone" real, and it's ~zero incremental cost today versus a painful retrofit later.

Two codebases is a real cost — roughly 1.5–2× ongoing feature work for anything that must exist on both. Contain it by ruthlessly *not* duplicating: NL generation, preset authoring, sharing, and account/library stay web/server-only; the iOS app consumes them via API.

---

## 5. Phased plan

### Phase 0 — now, on web (0 extra cost)
- Keep proving: do people use the FX-throw interaction? Which genres/presets get used? Does NL generation produce keepable beats? Web analytics answer these cheaper than TestFlight ever will.
- Extract the **project JSON schema** and make save/load/share round-trip through it.
- Measure and log real round-trip latency (`AudioContext.outputLatency` + loopback test) per browser/device — you want data, not vibes, for the go-native argument.
- Squeeze the web ceiling honestly: force off `echoCancellation/noiseSuppression/autoGainControl`, `latencyHint: 'interactive'`, move any custom DSP to AudioWorklet. If monitored latency still lands >25 ms on iOS Safari (it will), that's your documented trigger.

### Phase 1 — trigger check
**Go native when any two of these are true:**
1. You (or early users) are actually performing/recording vocals through PULSE more than ~weekly and the latency/monitoring complaints are the top issue.
2. Multitracking is being built — building it twice later is worse than building it native first.
3. You want autotune/pitch-correction on the monitored path (native-only, see §1.2).
4. Retention data shows mobile users bouncing off Safari's suspend/permission behavior.

Given the owner's own use case (performing over beats, punch-in FX, "Neptunes proud"), condition 1 is arguably already true. Don't wait for a big audience to justify it — this app's first pro user is the owner.

### Phase 2 — iOS v1: "the booth" (focused, not a port)
Scope: beat **playback** of synced projects + live mic monitoring + the full FX rack with tap-latch/throw + record vocal takes to stems + background audio + share sheet. Explicitly *not* v1: the sequencer editor, NL generation UI, arrangement editing (view/play only).
- **Effort: ~3–4 months for one senior iOS/audio dev** (AudioKit-assisted), of which the FX rack DSP parity and session/route-change hardening are the long poles. Add ~1 month if the pad/latch UI needs real polish. Solo owner-dev learning Swift audio: closer to 6–9 months, honestly.

### Phase 3 — deepen
Autotune/pitch correction (license or ~2–3 months of hard DSP work — do not underestimate this one), Core MIDI + BLE MIDI (~2 weeks), Ableton Link (~1–2 weeks), AUv3 hosting (~1 month), full multitrack editing (~2 months). Start the shared C++/Rust DSP-kernel library here so new FX land on web (preview quality) and iOS (live quality) from one source.

### Phase 4 — leverage
Ship the vocal FX rack as a standalone **AUv3** (works in GarageBand/Logic/AUM — cheap distribution into exactly the right audience), App Store subscription, web stays free-tier funnel.

---

## Bottom line

PULSE's identity — a performer singing live through punch-in vocal FX over pro-feeling beats — sits directly on the one capability the web platform structurally cannot deliver: low-latency monitored audio with reliable background operation. Everything else (sequencing, NL generation, presets, arrangement, sharing) is not just fine on the web, it's *better* there, because iteration and distribution are instant. So don't port and don't wrap: keep the web app as the writing room and funnel, define a shared project format now, and build a focused native Swift/AVAudioEngine iOS app that does one thing at pro quality — live vocal monitoring, the FX rack, and multitrack vocal capture — expanding into autotune, MIDI, Link, and AUv3 from that beachhead. It's roughly a one-senior-dev-quarter to a credible v1, it's the only path to the "Neptunes/Daft Punk proud" bar for the live half of the product, and given that the owner is the app's first professional user, the trigger for starting it has effectively already fired.
