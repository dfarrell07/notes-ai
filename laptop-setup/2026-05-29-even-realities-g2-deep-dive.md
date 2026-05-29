# Even Realities G2: Deep Dive for Agentic Developer Workflow

The G2 is a different paradigm from virtual-monitor AR glasses (XREAL, VITURE).
It is not a screen replacement. It is an ambient interrupt-handling terminal for
AI coding agents. This document covers everything needed to decide whether to buy.

## 1. Claude Code Integration

Three approaches exist, ranging from official to community-built:

### Official: Terminal Mode (v2.2.0, April 2026)

Built into the Even Realities phone app. Connect to a host machine (Mac) by
entering host info, port, and auth token. Once streaming, the agent session
appears on the glasses HUD. Agent-agnostic -- works with Claude Code, Codex CLI,
or any terminal agent. v2.2.1 added explicit Codex support.

### Community: claude-code-g2 (sam-siavoshian)

Full voice-first Claude Code client. Architecture:

```
G2 Glasses <-> (Bluetooth 5) <-> Phone (Vite+React WebView) <-> (HTTPS via Cloudflare tunnel) <-> Mac running claude CLI
```

Backend spawns `claude` in headless `stream-json` mode. User taps temple to
record, speaks task, taps again. Audio captured via glasses mic, sent to OpenAI
Whisper API (~$0.006/min) for transcription. Transcription shown on HUD for
confirmation. On confirm, sent as user turn to Claude CLI stdin. Events streamed
back via SSE to phone, then to glasses display.

HUD shows: full transcript with collapsed tool calls (`Read(file.ts)`), turn
separators, scroll bar, busy/done/idle indicators, connection loss warnings.
576x288 green monochrome display.

Runs with `--dangerously-skip-permissions` by default. Bearer token auth.
Projects whitelist restricts directories. Anyone with the token can execute
shell commands on the host Mac.

Setup: `git clone`, then `./dev.sh` installs everything and prints QR code.
Requires macOS, Claude Max/Pro, OpenAI API key, Homebrew.

Source: https://github.com/sam-siavoshian/claude-code-g2

### Community: cc-g2 (wmoto-ai)

Different philosophy: minimal interrupt handler, not full client. Uses Claude
Code HTTP hooks instead of headless mode.

Architecture:

```
Claude Code (tmux session) -> HTTP hooks -> Notification Hub server -> Tailscale -> iPhone -> Bluetooth -> G2 glasses
```

When Claude needs permission (run command, write file), HTTP hook sends
PermissionRequest to Hub. Hub holds request, waits for G2 response. Glasses show
summary. Ring tap approves. For denial with feedback, voice comment converts to
"deny + comment" response. Completion notifications show when tasks finish.

The developer who built this (a parent) found it was exactly right for handling
agent sessions while walking or doing chores. Key insight: smart glasses work
best as interrupt-handling terminals, not general-purpose displays.

Quote: "The psychological resistance to leaving the PC was reduced. Ring
operations feel natural. Handling approvals without taking out the smartphone
was more comfortable than I imagined."

Setup: `pnpm add -g github:wmoto-ai/cc-g2 && cc-g2 doctor && cc-g2`

Source: https://zenn.dev/wmoto_ai/articles/claude-code-even-g2-glasses?locale=en

### Also: OpenClaw Bridge

A Claude Code skill connecting G2 to OpenClaw via Cloudflare Worker. Voice
commands to manage AI tasks hands-free.

Source: https://mcpmarket.com/tools/skills/even-realities-g2-openclaw-bridge

## 2. Terminal Mode Details

Released in app v2.2.0 (late April 2026). Purpose-built for the pattern: run
agent, wait, check phone, approve, put phone down, repeat. Eliminates that loop.

### What the HUD shows:

- **Status indicators**: Thinking, Executing, Listening -- quiet and unobtrusive
- **Agent alerts**: Permission requests, approval prompts, error states cause
  the display to shift and signal attention needed
- **Multi-session visibility**: Track several agents simultaneously
- **Minimized mode**: Double-tap to collapse to background view while running
- **Full transcript view**: Scrollable terminal output with session text

### Interaction:

| Action                | Mechanism                              |
|-----------------------|----------------------------------------|
| View session          | Glance up at glasses display           |
| Minimize/expand       | Double tap                             |
| Approve               | Single tap R1 ring                     |
| Reject                | Double tap R1 ring                     |
| Voice instruction     | Tap and hold ring, speak, release      |
| Scroll                | Swipe up/down on ring                  |

### What it does NOT do:

- Does not replace a monitor for reading code
- Markdown, images, and code blocks render poorly on 1-bit monochrome
- Long text is not readable -- better suited for audio TTS
- Not agent-specific (does not know about Claude Code hooks natively)

## 3. Ring Control (R1)

The R1 ring ($249, sold separately or bundled) is a touch-sensitive band worn on
the index finger, operated with the thumb.

### Gestures:

- **Double-tap**: Wake/activate displays
- **Swipe up/down**: Scroll menus, navigate content
- **Single tap**: Select, approve
- **Long press**: Quick dashboard, or voice input in Terminal Mode

### Accuracy (from reviews):

**Good**: Discreet. More natural than tapping glasses arms. Reviewers call it
"the unsung hero" of the G2 experience. After brief learning curve, provides
inconspicuous control.

**Bad**: Sometimes requires multiple double-taps. Overshooting when scrolling.
Sensitivity changes when ring shifts on finger (users rotate it toward thumb
for better contact). Unintentional activations reported. Bluetooth disconnections
from phone/glasses randomly.

**Can you approve a PR with a tap?** Effectively yes, for the Claude Code use
case. In Terminal Mode or cc-g2/claude-code-g2, a single ring tap sends the
approval. The ring is accurate enough for binary approve/reject. Fine motor
scrolling through long content is where it struggles.

## 4. Voice Input

### How it works:

1. Four microphones in the glasses capture speech (up to 10 feet)
2. For Terminal Mode: tap and hold ring, speak, release. Voice goes to agent.
3. For claude-code-g2: tap temple, speak, tap again. Audio sent to OpenAI
   Whisper API (cloud, not local). Transcription shown for confirmation.
4. For cc-g2: hold ring, speak. Custom OpenAI-compatible HTTP server processes
   voice. Transcription sent as deny+comment or new instruction.

### Cloud vs. local:

- Native G2 voice commands: Hybrid. Basic voice-to-text appears to happen
  on-device (G2 sends transcribed text, not raw audio). Advanced features
  (Conversate, translation) require cloud.
- claude-code-g2 project: Explicitly uses OpenAI Whisper cloud API
- cc-g2 project: Custom OpenAI-compatible endpoint (configurable)
- Even Realities does not disclose their ASR engine. The AI backend is described
  as "hybrid" using Gemini, ChatGPT, and Perplexity.

### Quality:

Multiple reviewers report the four-mic array provides "clean, directional
pickup." One reviewer (G1 owner) called the microphones "terrible" for
translation, with accuracy issues even in quiet rooms. Translation accuracy
may have improved with G2 hardware. For short agent commands and approvals,
voice input is generally adequate.

## 5. Prescription Lenses

**Yes, fully supported.** This is a major differentiator from XREAL/VITURE.

### How it works:

Not clip-in inserts. Even Realities bonds a digitally surfaced prescription lens
directly to the waveguide display lens, creating a single ultra-slim optic. No
gap between lenses (no dust, moisture, smudges). Looks like standard eyewear.

### Prescription range:

- Single vision: -12.00 to +12.00 diopters (covers essentially all corrections)
- Astigmatism: Supported
- Progressive lenses: Available through partner opticians only (not online)
- Prism lenses: Even Retail Store only

### Lens options:

- Essential Slim (1.60 index): For +2.00 to -3.00
- Enhanced Slim (1.67 index): For +4.00 to -6.00
- Higher index available for stronger prescriptions

### Cost:

- Prescription lens add-on from $159 at checkout
- FSA/HSA eligible
- Lens replacement available through support

### Ordering:

Enter prescription at checkout (or send later). PD measurement required. Custom
built and shipped. Estimated lead time: 6-12 weeks.

## 6. Battery Life

### Official claim: 2 days typical use

### Real-world testing:

**Trusted Reviews**: "It really does last for two full days." With notifications,
Conversate, and navigation, reviewer ended 14-hour days with 50-60% remaining.
Never worried about battery.

**VR Expert**: Full working day of continuous professional use.

**Why so good**: No camera, no speakers, no positional tracking. All power goes
to the micro-LED display and Bluetooth. Display technology is efficient.

### Charging:

- Case provides 7 full recharges
- R1 ring battery drains faster than spec suggests -- may need more frequent
  charging than the glasses

### For continuous Terminal Mode HUD:

No one has published specific Terminal Mode battery drain data. Given that
Terminal Mode keeps the display active with streaming text, expect somewhat
faster drain than typical intermittent use. Conservative estimate: probably
a full working day (8-12 hours) of active Terminal Mode before needing the case.

## 7. Android / Pixel Phone Compatibility

**Yes, works with Android including Pixel phones.**

The G2 connects via Bluetooth 5.4 and requires the Even Realities companion app.

### App quality (Google Play):

The Android app has notable quality issues:
- Setup and pairing reported as buggy, with freezes during initial pairing
- Multiple pairing prompts, truncated UI text
- Health metrics disappear, requiring unpair/repair cycles
- Bluetooth disconnections for both glasses and ring

### Improvements:

- Even Realities actively updating ("less clutter and cleaner prompts")
- v2.2.0 and v2.2.1 brought stability improvements
- Bluetooth troubleshooting prompts added in v2.2.1
- Even Hub (app marketplace) launched April 2026 with ~50 apps

### Bottom line for Pixel:

It works. No Pixel-specific incompatibility reported. But expect some software
friction. The app is functional but still being polished. iOS reportedly has a
better experience but Android is actively supported.

## 8. Developer SDK

### Official: Even Hub Platform

Launched April 2026. Open developer ecosystem for G2 apps.

- **Language**: JavaScript/TypeScript (npm package `@evenrealities/even_hub_sdk`)
- **Templates**: 6 starter templates (minimal, dashboard, notes, chat, tracker, media)
- **Publishing**: Build, test, publish directly to Even Hub marketplace
- **Docs**: https://hub.evenrealities.com/docs
- **GitHub**: https://github.com/even-realities (includes EvenDemoApp)

### Community: even-toolkit

Design system and component library by fabioglimb:
- 55+ web components, 191 pixel-art icons
- Glasses SDK bridge with per-screen architecture
- Speech-to-text module
- Light/dark themes following Even UIUX Design Guidelines
- Source: https://github.com/fabioglimb/even-toolkit

### Community: BLE Protocol Reverse Engineering

Full reverse engineering of the G2 Bluetooth Low Energy protocol:
- Custom BLE protocol for data transmission in packets
- Removes restriction to Even app functionality
- Source: https://github.com/i-soxi/even-g2-protocol

### Community: Hub Simulator

Desktop testing environment for G2 apps without physical glasses:
- Source: https://github.com/BxNxM/even-dev

### What you CAN build:

- Custom HUD apps with text display
- Voice input integrations
- Ring gesture handlers
- AI/LLM integrations
- Dashboard widgets
- Notification handlers

### Caution:

A security researcher found a hard-coded DeepSeek API key in the official demo
source that still worked. Engineering quality of official SDK may be uneven.

## 9. Privacy

### No camera. Period.

Not hidden, not disabled, not optional. The lens housing is pure glass. This is
a deliberate architectural decision, not a cost cut.

### Social acceptability:

This is the G2's strongest advantage over Meta Ray-Ban, Rokid, or any
camera-equipped glasses. You can wear these into:
- Courtrooms, doctor's offices, schools
- Corporate boardrooms, government facilities
- Any camera-sensitive environment

No one will think you are recording them. The glasses genuinely look like
regular premium eyewear at 36g.

### However -- audio concerns:

Security researchers raise valid points:
- Four always-available microphones capture conversations
- Even Realities is headquartered in Shenzhen, funded by Chinese VC
- Subject to China's National Intelligence Law
- Audio recordings and voiceprints sent to unnamed third-party providers
- Even Realities uses a proprietary "Even LLM" with undisclosed hosting

No camera does not mean no surveillance. Audio data handling and corporate
governance are legitimate concerns for anyone working with sensitive code
or conversations.

### Privacy claim vs. reality:

Even Realities says: "No voice recordings are stored -- only transcriptions and
conversation summaries are stored on your smartphone."

Security researchers say: The data pipeline passes through cloud services with
undisclosed providers before results are stored locally.

### Future camera plans:

CEO Will Wang has said they may make a camera as a snap-on accessory in the
future, removable when not needed.

## 10. Developer User Reviews

### Positive experiences:

**wmoto-ai (cc-g2 builder)**: "The psychological resistance to leaving the PC
was reduced. Ring operations feel natural. Being able to return minimum
operations without taking out the smartphone was more comfortable than I
imagined." Calls it "an operational layer for an AI coding session."

**Sam Siavoshian (claude-code-g2 builder)**: "Think GitHub Copilot but on your
face, controlled entirely by voice and temple taps."

**JU CHUN KO (OpenClaw Bridge)**: "Walking around and asking questions by voice,
getting answers projected onto your glasses -- it's a fundamentally different
interaction model. The latency is noticeable (~3-5 seconds for short answers),
but acceptable. It feels like having a knowledgeable friend whispering answers
to you."

### Negative/mixed experiences:

**Hacker News community**: Skeptical but intrigued. Hardware praised, software
ecosystem called too restrictive. "The apps don't solve any of my problems."
"I am not putting any hardware on my nose if I can't fully control the software
running on it." One commenter: "The pebble in smart glasses land, simple and
elegant."

**IT professional on Trustpilot**: Returned after two weeks. Called it "a premium
prototype that was rushed to market." Hardware commendable, software ecosystem
falls short.

**G1 owner (jhugo on HN)**: Microphones "terrible" for live translation. "Barely
works even in quiet rooms, on the street it rarely gets even a single word
correct." (Note: G2 hardware is improved over G1.)

**Wired**: "Impressive smart glasses, but the software needs polish."

### Overall developer verdict:

Hardware is genuinely good. Software is actively improving but still rough.
Terminal Mode is the killer feature for the agentic workflow. Community projects
(cc-g2, claude-code-g2) prove the concept works. But expect to be an early
adopter dealing with bugs, disconnections, and evolving APIs.

## 11. Price and Availability

### Pricing:

| Item                          | Price |
|-------------------------------|-------|
| G2 A (Crown Panto frame)     | $599  |
| G2 B (Rectangular frame)     | $599  |
| R1 Smart Ring                 | $249  |
| Prescription lens add-on      | From $159 |
| Bundle discount available     | Varies |

### Where to buy:

- **Direct**: evenrealities.com/store (ships to US)
- **US retailers**: Authorized opticians (store locator on site)
- **Lead time**: 6-12 weeks (made to order)
- **Payment**: Klarna installments, 0% interest plans, FSA/HSA eligible

### Total cost estimate for this use case:

G2 glasses ($599) + R1 ring ($249) + prescription lenses (~$159-$200) = ~$1,050

## 12. Competitors for Agent Monitoring

### Even Realities G2 -- the clear leader for this use case

Only product with a purpose-built Terminal Mode for coding agents. 2-day
battery. Looks like normal glasses. Active Claude Code community. No camera
(social acceptability). Prescription support.

### Rokid Glasses -- broader hardware, less focused

- 49g, 480x398 per eye, 12MP camera, speakers, 1500 nits
- Integrating OpenClaw for agent interaction
- More features but less focused on the developer agent workflow
- Battery: ~6 hours music / 45 min video (much worse than G2)
- Camera makes it less socially acceptable

### Meta Ray-Ban Display -- wrong tool

- 600x600 monocular, 20-degree FOV
- No Terminal Mode equivalent
- Third-party developers cannot access the display yet (promised "this year")
- One developer (Jake Ledner) hacked agent monitoring via WhatsApp messages
  but it is a workaround, not a supported workflow
- Meta AI locked in as voice assistant (not Claude)

### Google Android XR glasses -- vaporware for now

- Announced at Google I/O 2026
- Audio glasses launching fall 2026, display glasses "later" with no date
- Could be significant when it ships (full Android app ecosystem)
- Not available to buy

### Apple -- not for years

- Targeting 2027 at earliest for smart glasses (Project N50)

### Snap Spectacles -- AR focused, not agent focused

- Spun off as Specs Inc.
- Focused on full AR immersion, not ambient HUD
- Developer-only at $99/month subscription

### Bottom line:

The Even Realities G2 is the only shipping product that explicitly targets the
"monitor your AI coding agent while mobile" use case. Everything else either
lacks a display, lacks the right software, or does not exist yet.

## Assessment: Should You Buy?

### Buy if:

- You run Claude Code agents frequently and want to stop babysitting your laptop
- You value the ability to approve/reject agent actions while walking, at coffee,
  or doing other things
- You need prescription glasses anyway (replaces a pair you would buy)
- You accept early-adopter software quality
- You are comfortable with the privacy tradeoffs (Chinese company, audio in cloud)

### Do not buy if:

- You want to read or write code on the glasses (this is not that)
- You need rock-solid, polished software today
- You are uncomfortable with audio data going through undisclosed cloud providers
- You do not use AI coding agents regularly
- You primarily use Windows or Linux (Terminal Mode requires Mac currently)

### The honest take:

The Even G2 with Terminal Mode is the most interesting product in the agentic
developer workflow space. It solves a real problem: the developer-as-conductor
is still screen-tethered while the agent does the work. The G2 breaks that
tether. But it is early. The software is unfinished. The community integrations
are more capable than the official Terminal Mode. If you are the kind of person
who enjoys being on the bleeding edge and contributing to an ecosystem, this
is worth the ~$1,050 investment. If you want something that "just works," wait
6-12 months.

## Sources

- [claude-code-g2 (GitHub)](https://github.com/sam-siavoshian/claude-code-g2)
- [cc-g2 developer blog (Zenn.dev)](https://zenn.dev/wmoto_ai/articles/claude-code-even-g2-glasses?locale=en)
- [Even G2 Terminal Mode (official)](https://www.evenrealities.com/terminal)
- [Terminal Mode announcement (Engadget)](https://www.engadget.com/2161082/even-realities-terminal-mode-ai-agent/)
- [Terminal Mode announcement (Digital Trends)](https://www.digitaltrends.com/wearables/even-realities-smart-glasses-bring-the-coding-terminal-to-your-eyeball/)
- [v2.2.0 release (Even Realities on X)](https://x.com/EvenRealities/status/2048702198297829684)
- [v2.2.1 release (Even Realities on X)](https://x.com/EvenRealities/status/2052011171876893052)
- [Even Hub SDK docs](https://hub.evenrealities.com/docs)
- [Even Realities GitHub](https://github.com/even-realities)
- [BLE protocol reverse engineering](https://github.com/i-soxi/even-g2-protocol)
- [even-toolkit (community SDK)](https://github.com/fabioglimb/even-toolkit)
- [Hub Simulator](https://github.com/BxNxM/even-dev)
- [OpenClaw Bridge skill](https://mcpmarket.com/tools/skills/even-realities-g2-openclaw-bridge)
- [Trusted Reviews (battery/ring)](https://www.trustedreviews.com/reviews/even-realities-g2)
- [Tom's Guide review](https://www.tomsguide.com/computing/smart-glasses/even-realities-g2-smart-glasses-review)
- [Gizmodo review](https://gizmodo.com/even-realities-even-g2-review-2000687632)
- [9to5Google review](https://9to5google.com/review-even-realities-g2-smart-glasses/)
- [Hacker News discussion](https://news.ycombinator.com/item?id=45917752)
- [Prescription lenses (official)](https://www.evenrealities.com/prescription-smart-glasses)
- [Privacy investigation (Security Boulevard)](https://securityboulevard.com/2026/03/internal-analysis-even-realities-g2-smart-glasses-security-privacy-investigation/)
- [Camera-free design (The Gadgeteer)](https://the-gadgeteer.com/2026/04/04/7-reasons-these-camera-free-smart-glasses-keep-winning/)
- [Even Realities shop](https://www.evenrealities.com/store)
- [Android app (Google Play)](https://play.google.com/store/apps/details?id=com.even.sg)
- [Rokid + OpenClaw](https://www.shashi.co/2026/04/the-agent-you-can-wear-rokid-puts.html)
- [Smart glasses comparison 2026](https://treeview.studio/blog/best-smart-glasses)
- [Even G2 + OpenClaw Bridge blog](https://blog.juchunko.com/en/even-realities-g2-openclaw-bridge/)
