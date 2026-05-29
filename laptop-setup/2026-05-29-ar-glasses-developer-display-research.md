# AR / Smart Glasses as Developer Display (2026 Research)

Use case: Wearing glasses that show a terminal/browser while on the go, connected
to Claude Code running on a remote machine via phone or iPad.

## Product Comparison

### 1. XREAL One Pro ($599) - BEST FOR THIS USE CASE

- **Display**: 1920x1080 per eye, Sony Micro-OLED, 57-degree FOV, 120Hz, 700 nits
- **Virtual screen**: ~171 inches equivalent
- **Pixel phone (USB-C)**: YES - works with Pixel 6+ (requires DP Alt Mode over USB-C)
- **iPad Pro (USB-C)**: YES - plug and play, especially good with iPadOS 26 windowing
- **Text readability**: Good. No screen-door effect. Terminal text reported readable.
- **Screen anchoring**: Built-in X1 chip provides native 3DoF with 3ms latency. Screen
  stays stable without needing external accessories (unlike older Air models that
  needed the Beam accessory).
- **Comfort**: 84g, no internal battery (draws from connected device). Comfortable for
  extended sessions but drains host device battery.
- **Outdoor use**: Electrochromic dimming (3 levels). Usable in shade/overcast. Direct
  midday sunlight still challenging at 700 nits (need ~1500+ for true sunlight
  readability).
- **Multi-monitor**: NOT supported on One series. Nebula software discontinued for this
  hardware. Single virtual display only, but has ultrawide 32:9 mode (310 inches).
- **Key limitation**: Single display only. No multi-screen like older XREAL models.
  Battery drain on connected device.

### 2. XREAL Air 2 Ultra ($699) - DEVELOPER-FOCUSED BUT OLDER

- **Display**: 1080p per eye, Sony Micro-OLED, 52-degree FOV, 120Hz
- **Pixel phone**: YES (same DP Alt Mode requirement)
- **iPad Pro**: YES via USB-C
- **Text readability**: Good, no screen-door effect
- **Screen anchoring**: Full 6DoF tracking (more capable than One Pro), but needed
  Beam accessory for stable screen anchoring in practice
- **Comfort**: 80g. Head movement while typing can be disorienting without Beam.
- **Multi-monitor**: Supports Nebula for up to 3 virtual screens (unlike One Pro)
- **Outdoor**: No electrochromic dimming - needs clip-on shade accessory
- **Key limitation**: Older model, superseded by One series. Beam accessory adds
  complexity. No built-in dimming.

### 3. VITURE Beast ($549) - STRONG COMPETITOR

- **Display**: 1920x1200 per eye (1200p), Sony Micro-OLED, 58-degree FOV, 120Hz,
  1250 nits
- **Virtual screen**: 174 inches
- **Pixel phone**: YES via USB-C
- **iPad Pro**: Should work via USB-C (listed as compatible with iPhone, Android, Mac, PC)
- **Text readability**: Should be very good given 1200p resolution
- **Screen anchoring**: Built-in 3DoF (VisionPair), 5 viewing modes including Anchor
- **Comfort**: 88g, no battery. 9-level electrochromic dimming.
- **Outdoor**: Best brightness in category at 1250 nits. Still not full sunlight readable
  but better than competitors.
- **Multi-monitor**: SpaceWalker software for multi-screen across iOS, Android, macOS,
  Windows
- **Key limitation**: 1200p/120Hz and 6DoF were not active at launch (April 2026).
  Running at lower specs initially. New product with less field testing.
- **NOTABLE**: Brightest display, widest FOV, multi-platform multi-screen support,
  lowest price. If launch features ship, this may be the best option.

### 4. Apple Vision Pro ($3,499) - OVERKILL BUT POWERFUL

- **Display**: Dual micro-OLED, ~23 million pixels total, ultrawide Mac Virtual Display
  equivalent to two 5K monitors side-by-side
- **Pixel phone**: NO. Apple ecosystem only.
- **iPad Pro**: Not as a display source. Vision Pro IS the computer. iPad apps run
  natively on it. Mac Virtual Display requires a Mac (MacBook, iMac, etc).
- **Terminal apps**: La Terminal (by Miguel de Icaza) - native visionOS SSH client with
  AI command assistance. Also can run any macOS terminal via Mac Virtual Display.
- **Text readability**: Right at the edge. "Individual pixels aren't visible, but coding
  at typical font sizes was uncomfortable" for some users. M5 (2025) improved this
  with 120Hz Mac Virtual Display.
- **Comfort**: 1.6 lbs without battery. M5 improved with Dual Knit Band, reviewers say
  multi-hour sessions now feasible but still noticeably heavy.
- **Battery**: 2.5 hours general use, 3 hours video. Can plug in for infinite use at desk.
- **Outdoor**: Works but the headset form factor is socially conspicuous. Good for
  private spaces.
- **Key limitation**: $3,499. Heavy. Requires Mac for best coding experience. No phone
  connection for display. Not portable in the "park bench" sense - you look like
  you're wearing a ski goggle.

### 5. Ray-Ban Meta ($299-$379 / Display version $799)

- **Original (no screen)**: Camera + speakers + Meta AI voice assistant. NO display.
  Cannot show terminal/code. Voice-only interaction via "Hey Meta" (uses Llama 4).
  Cannot run Claude as voice assistant (Meta AI only).
- **Ray-Ban Meta Display (2025)**: Small monocular display, 600x600 pixels, 20-degree
  FOV. This is a notification display, not a coding display. Way too small and
  low-res for terminal work.
- **Pixel phone**: Pairs via Bluetooth for audio/voice. Not a display device.
- **iPad Pro**: No display connection.
- **For this use case**: NOT suitable. No useful display for code. The voice assistant
  is Meta AI, not Claude. Could theoretically be used for voice-only Claude
  interaction via a custom app, but that's not the intended workflow.

### 6. TCL RayNeo X2 ($899)

- **Display**: Binocular full-color Micro-LED optical waveguide, 1000+ nits
- **Form factor**: Standalone AR glasses with built-in Snapdragon XR2, 6GB RAM, battery
- **Pixel phone**: Not designed as phone display. Standalone with own compute. Can
  share phone internet connection.
- **iPad Pro**: Not a display accessory.
- **Text readability**: Waveguide display - significantly lower resolution than
  micro-OLED solutions. Small transparent overlay, not full virtual screen.
- **Comfort**: True glasses form factor (unlike birdbath optics), but noticeably
  larger than regular glasses.
- **For this use case**: NOT suitable for terminal/coding work. The waveguide display
  is designed for HUD-style overlays (navigation, translation, notifications), not
  for reading dense code. Resolution too low, FOV too narrow.

### 7. Rokid Max 2 ($399-$529)

- **Display**: 1920x1080 per eye, Sony Micro-OLED, 50-degree FOV, 120Hz, 600 nits
- **Virtual screen**: 215 inches
- **Pixel phone**: YES via USB-C (requires DP Alt Mode)
- **iPad Pro**: YES via USB-C
- **Text readability**: Decent, but frame design blocks downward view to keyboard
- **Comfort**: 75g - lightest option. Built-in myopia adjustment (0 to -6.00D).
- **Outdoor**: No electrochromic dimming mentioned
- **Multi-monitor**: Possible with Station 2 add-on, but Station 2 is a separate
  Android compute box
- **Key limitation**: Reviewers consistently say "better for media consumption than
  productivity." 50-degree FOV is narrower than competitors. Bottom frame blocks
  keyboard view. No dimming.

### 8. Even Realities G2 ($599) - BEST FOR AI AGENT MONITORING

- **Display**: Green Micro-LED waveguide, small HUD-style display on both lenses.
  NOT a virtual monitor. Shows text overlays on top of the real world.
- **Resolution**: Small display, 576x288 green-only - not for reading full code files.
- **Form factor**: 36-40g, looks like normal glasses, titanium/magnesium frame.
  Prescription lens support (-12.00 to +12.00). IP67 rated.
- **Battery**: 2-day battery life with charging case for 7 charges.
- **Terminal Mode (v2.2.0, April 2026)**: Purpose-built for monitoring AI coding agents.
  Shows agent status (Thinking, Executing, Listening). Tap R1 ring for approvals.
  Hold ring for voice input to agent.
- **Claude Code integration**: Multiple community projects exist:
  - `claude-code-g2`: Voice-first Claude Code via HUD. Tap, speak, tap. Whisper
    transcribes, Claude executes, HUD streams result.
  - `cc-g2`: Claude Code HTTP hooks for approval/rejection from glasses.
- **For this use case**: NOT a replacement for a monitor. You cannot read code on it.
  But it IS the best option for ambient AI agent monitoring - keep an eye on Claude
  Code while walking around, approve/reject actions via ring tap + voice. Different
  paradigm from "stare at virtual screen."

## The Practical Workflow Assessment

### Setup: Glasses + Keyboard + Phone/iPad + Claude Code

The target workflow: AR glasses on face, Bluetooth keyboard on lap, Claude Code
running on a remote machine, SSH'd via phone or iPad from a park bench.

**What actually works today (mid-2026):**

1. **XREAL One Pro + iPad Pro + Bluetooth keyboard**: Plug XREAL into iPad USB-C.
   iPad shows as large virtual display. Open terminal app (Blink, Termius, a]Shell).
   SSH to remote machine running Claude Code. Bluetooth keyboard (Moonlander/Voyager)
   pairs to iPad. This works. iPadOS 26 windowing makes it better. Single virtual
   screen, but ultrawide mode gives 310-inch equivalent.

2. **VITURE Beast + iPad Pro + Bluetooth keyboard**: Same setup. Potentially better
   due to SpaceWalker multi-screen support and higher brightness. But newer product,
   less proven.

3. **XREAL One Pro + Pixel phone + Bluetooth keyboard**: Pixel 6+ supports DP Alt
   Mode. Phone mirrors/extends to glasses. Run Termux or full Linux in chroot, SSH
   to remote machine. Bluetooth keyboard pairs to phone. This works but phone screen
   is less useful for windowed workflow than iPad.

4. **Even G2 + phone + Claude Code**: Different paradigm. Claude Code runs on remote
   machine. G2 Terminal Mode shows agent status on HUD. Approve/reject via ring tap.
   Voice commands for detailed instructions. Phone stays in pocket. You walk around
   while Claude works. Most "hands-free" option but not for reading/writing code.

### Bluetooth Keyboard Compatibility

All AR glasses that work as display accessories connect to the host device (phone/
iPad), not directly to the glasses. The keyboard also connects to the host device
via Bluetooth. So:

- Moonlander/Voyager connects to iPad or phone via Bluetooth
- AR glasses connect to iPad or phone via USB-C
- Keyboard input goes to host device, display output goes to glasses
- This just works. No special pairing needed between keyboard and glasses.

### Real Developer Experiences

**"Coding Without a Laptop" (May 2025, holdtherobot.com):**
Two weeks with AR glasses + Pixel 8 Pro + Linux in chroot + folding keyboard.
Conclusion: it works, it's fun, a laptop is still better in almost all ways. The
keyboard is the weakest link. Vision quality has non-uniform sharpness. Walking
causes distracting text bounce.

**Tom's Guide mini PC setup (2025):**
Khadas Mind mini PC + XREAL One + mechanical keyboard + trackball mouse. Became
the author's go-to travel setup. Fits on airplane tray table. Dual/triple virtual
monitors via Nebula.

**Wasabigeek remote work (2024-2025):**
XREAL Air 2 Pro for programming during 2-week trip. Kept using them for travel.
Would not replace 27-inch monitor at home. Beam accessory nearly essential for
screen stability.

**Even Realities Terminal Mode users (2026):**
Developers monitoring Claude Code via G2 glasses while walking, at coffee shops,
during workouts. Not typing code - monitoring and approving agent actions. This is
the "new paradigm" workflow.

## Honest Assessment

### What is practical today:

- **Indoor virtual monitor replacement for travel**: YES. XREAL One Pro or VITURE
  Beast + iPad/laptop gives you a usable large virtual display for SSH/terminal
  work. Good for planes, hotel rooms, coffee shops. Multiple developers do this
  regularly.

- **AI agent monitoring while mobile**: YES. Even G2 Terminal Mode + Claude Code is
  a real workflow. Let Claude work, get notified when it needs approval, tap or
  speak your response. Genuinely useful for the agentic coding era.

### What is semi-practical:

- **Park bench coding**: Possible but compromised. Outdoor brightness is still a
  challenge (need shade). Battery drain on phone/iPad limits sessions. Walking
  causes text bounce. You need a surface for the keyboard. A park bench with shade
  actually works okay. A park bench in direct sun does not.

- **Phone-only setup (no iPad)**: Works but cramped. Android desktop mode (DeX or
  Android 16 desktop) helps. Linux in chroot adds capability. But iPadOS 26
  windowing on iPad Pro is a significantly better experience.

### What is gimmicky:

- **Replacing your desk monitor setup**: No. 50-57 degree FOV is like looking through
  a porthole compared to a 27-inch monitor at arm's length. Eye strain after 2+
  hours. Text clarity is "good enough" not "great." Every reviewer who tries this
  says they would not give up their desk monitor.

- **Coding while walking**: No. Text bounces with head movement. You need your hands
  for a keyboard. You need to see where you're going. Even the Even G2 terminal
  mode is for monitoring, not writing code while walking.

- **Apple Vision Pro as portable dev machine**: Technically the best display, but
  $3,499, 1.6 lbs on your face, 2.5 hour battery, requires a Mac nearby for Mac
  Virtual Display. It's a desk/couch device, not a park bench device.

## Recommendation for This Use Case

**Primary setup: XREAL One Pro ($599) + iPad Pro**
- Plug and play via USB-C
- iPadOS 26 windowing is a good fit
- 57-degree FOV, 1080p, 120Hz, 700 nits
- Bluetooth keyboard pairs to iPad
- SSH to remote machine running Claude Code
- Good for travel, coffee shops, indoor spaces with controlled lighting
- Ultrawide mode for more screen real estate

**Alternative: VITURE Beast ($549)**
- If the 1200p/120Hz firmware update ships and works well
- Better brightness (1250 nits) for outdoor use
- Multi-screen via SpaceWalker
- Slightly wider FOV (58 degrees)
- Newer product with less proven track record

**Complement: Even Realities G2 ($599) for ambient monitoring**
- Wear these when NOT actively coding
- Monitor Claude Code agent status via Terminal Mode
- Approve/reject via ring tap, give voice instructions
- Looks like normal glasses, 2-day battery, 36g
- The "check on Claude while getting coffee" device

## Sources

- [XREAL One Pro review with iPad Pro](https://9to5mac.com/2025/07/02/xreal-one-ar-glasses-review-versatile-with-m4-ipad-pro/)
- [XREAL One Pro shop page](https://us.shop.xreal.com/products/xreal-one-pro)
- [XREAL compatible phones](https://wearablexp.com/smart-glasses/xreal-compatible-phones/)
- [XREAL Air 2 for programming](https://wasabigeek.com/blog/using-the-xreal-air-2-pros-for-remote-work-programming/)
- [VITURE Beast launch](https://vr.org/articles/viture-beast-xr-glasses-review-april-2026)
- [VITURE Beast specs](https://www.viture.com/product/viture-beast-xr-glasses)
- [VITURE compatibility list](https://www.viture.com/compatibility)
- [Apple Vision Pro M5 review](https://apple.gadgethacks.com/news/apple-vision-pro-m5-review-finally-ready-for-work/)
- [Coding on Vision Pro](https://vincelwt.com/visionpro)
- [La Terminal SSH for Vision Pro](https://www.producthunt.com/products/la-terminal-ssh-client-for-vision-pro)
- [Vision Pro Mac Virtual Display](https://support.apple.com/guide/apple-vision-pro/use-mac-virtual-display-tan357ede966/visionos)
- [Vision Pro M5 comfort](https://www.macrumors.com/review/vision-pro-m5-chip/)
- [Ray-Ban Meta Display](https://www.meta.com/ai-glasses/meta-ray-ban-display/)
- [TCL RayNeo X2 review](https://www.uploadvr.com/rayneo-x2-standalone-ar-glasses-review/)
- [Rokid Max 2 review](https://www.seriousinsights.net/rokid-max-2-ar-glasses-review-wired-ar-glasses-prove-better-as-media-consumption-companion-than-a-work-display/)
- [Even G2 Terminal Mode](https://www.evenrealities.com/terminal)
- [Even G2 + Claude Code (GitHub)](https://github.com/sam-siavoshian/claude-code-g2)
- [Even G2 + Claude Code (blog)](https://zenn.dev/wmoto_ai/articles/claude-code-even-g2-glasses?locale=en)
- [Even G2 Terminal Mode announcement](https://www.engadget.com/2161082/even-realities-terminal-mode-ai-agent/)
- [Coding without a laptop - AR glasses blog](https://holdtherobot.com/blog/2025/05/11/linux-on-android-with-ar-glasses/)
- [Coding without a laptop - HN discussion](https://news.ycombinator.com/item?id=43985513)
- [Mini PC + AR glasses setup](https://www.tomsguide.com/computing/i-ditched-my-laptop-for-a-pocketable-mini-pc-and-a-pair-of-ar-glasses-heres-what-happened)
- [Mini PC + AR glasses travel setup](https://www.tomsguide.com/computing/i-paired-a-pocketable-mini-pc-with-a-pair-of-ar-glasses-and-even-i-was-surprised-when-it-became-my-go-to-travel-setup)
- [XREAL for remote work](https://dillonbaird.io/blog/xrealair/)
- [Outdoor brightness requirements](https://www.risinglcd.com/news/How-Many-Nits-Do-You-Need-to-See-a-Screen-in-Full-Sunlight.html)
