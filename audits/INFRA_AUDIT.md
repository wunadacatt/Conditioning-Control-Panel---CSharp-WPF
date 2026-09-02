# Infrastructure & Repo Health Audit

**Date:** 2026-09-01
**Against:** `main` @ `99869e98` (v6.9.0 "The Spiral")
**Scope:** build/SDK/toolchain, CI, dependency health, repo weight, compiler-surfaced
latent bugs, localization parity, versioning consistency. Not a security review of app
logic, not a UX/DPI review (tracked separately).

**Method:** `dotnet build` / `dotnet list package --deprecated|--vulnerable|--outdated`,
`git` object inspection, compiler-warning triage, JSON key-parity counts, source grep.

---

## Verdict

Infra is thin (no CI, accreted config) with a visible tail of half-finished migrations
and unaddressed compiler warnings. No deprecated or vulnerable **direct** packages,
version strings are consistent, no secrets are committed. The load-bearing problems are
**repo weight (2.5 GiB pack)**, **localization drift (~10% of the UI untranslated in
non-English locales)**, and **a handful of concrete latent bugs the compiler already
points at**.

---

## P0 — concrete latent bugs (compiler is already flagging these)

| # | Where | Problem |
|---|---|---|
| 1 | `ConditioningControlPanel/Services/StartupManager.cs:19` | `ApplicationPath => Assembly.GetExecutingAssembly().Location.Replace(".dll",".exe")` resolves to `".exe"` in the shipped **single-file** build (`IL3000`). The file also has a correct `Environment.ProcessPath`-first resolver near line 113 — **two implementations**. If the shortcut-writer uses the property, "Start with Windows" is broken in every installed copy. Confirm the live call site. |
| 2 | `ConditioningControlPanel/Views/Deeper/EnhancementPlayerWindow.xaml.cs:35` | `_isScrubbing` is declared, read at line 886 (`if (_isScrubbing) return;`), and **never assigned** (`CS0649`). The scrub/seek re-entrancy guard is dead — the race it was meant to prevent is unguarded. |
| 3 | 9 sites | Fire-and-forget `async` (`CS4014`): `Chaos/ChaosFlashOverlay.cs:218`, `Chaos/ChaosGifCascadeOverlay.cs:413`, `AvatarTube/AvatarTubeWindow.Reactions.cs` ×3, `Views/Controls/Companion/Runtime/ChatThresholdRuntimeVm.cs:395`, `Services/GoonGame/GoonHostService.cs:1037`, `MainWindow/MainWindow.RemoteControl.cs:75`. This is **CLAUDE.md known-issue #7** verbatim; each is a silent unobserved-exception path. |
| 4 | `ConditioningControlPanel/Services/Flash/FlashService.cs:4305` | `FlashWindow.VisualOpacity` **shadows** `Visual.VisualOpacity` (`CS0108`). Shadowing a WPF framework member — bindings/animations can resolve to the base property, not the intended one. |
| 5 | transitive | **2× HIGH-severity CVEs**: `System.Net.Http 4.3.0` (GHSA-7jgj-8wvc-jh57), `System.Text.RegularExpressions 4.3.0` (GHSA-cmhx-cq75-c4mj), dragged in by an old dependency. On .NET 8 the in-box assemblies supersede them so runtime risk is ~nil, but `dotnet list package --vulnerable`, Dependabot, and every SCA scanner flag it. Fix: add explicit direct references to patched versions, or find and bump the offender. |

---

## P1 — infrastructure & process

**6. No CI.** `.github/` does not exist — nothing builds, tests, lints, or scans on push
or PR. The `NETSDK1151` solution-build break (fixed in PR #1) sat unnoticed for exactly
this reason. One GitHub Actions workflow (`dotnet build` + `dotnet test` +
`dotnet list package --vulnerable`) would catch #1, #2, #5 and any future NETSDK break.

**7. 171 build warnings, `TreatWarningsAsErrors` unset.** Permanent noise floor; new
warnings disappear into it. ~90 are `CS8602`/`CS8604` nullable-dereference with a single
root cause (see #17). Baseline the existing set so *new* warnings fail the build.

**8. Two migrations abandoned in place** (the `CS0618` spam):
- `ContentMode` → "use `App.Mods` / `ActiveModId`": still referenced in `Models/AppSettings.cs`
  ×6, `Models/Session.cs` ×3, `Services/Quiz/IntakeNiche.cs` (some may be the compat shim
  itself — needs triage).
- `IAiService.GetXxxAsync` legacy one-shot calls, "kept … through the Train 1 migration":
  still called from `AvatarTube/AvatarTubeWindow.ChatInput.cs` ×3, `.Reactions.cs` ×6,
  `Services/AutonomyService.cs`, `Services/KeywordTriggerService.cs`,
  `Services/Commands/GetBackToMeCommand.cs`. `App.Brain` / `CompanionBrain` is the intended
  path.

Each `[Obsolete]` shim is debt with a half-life: finish the migration or drop the attribute.

**9. No `Directory.Build.props`.** `TargetFramework` / `Nullable` / `WindowsSdkPackageVersion`
are copy-pasted across three `.csproj` and have **already drifted**:
`Tools/GenerateAwarenessSounds` is bare `net8.0` with no SDK-package pin and carries
`ImplicitUsings` the other two lack.

**10. `.gitattributes` has no EOL normalization** (three LFS lines only). CLAUDE.md
known-issues #12/#13/#14 are all line-ending/encoding failures. `* text=auto`,
`*.json text eol=lf`, `*.cs text` would enforce what currently depends on each machine's
`core.autocrlf`.

**11. Missing release notes for 6.8.5, 6.8.6, 6.9.0.** Those versions shipped
(`btn_v6_8_5_is_out` / `btn_v6_8_6_is_out` / `btn_v6_9_0_is_out` loc keys exist; CLAUDE.md
cites the 6.8.6 font crash). The `/release` workflow bumps version strings and loc keys but
the hand-written `release-notes/notes-vX.Y.Z.txt` step keeps being skipped.

---

## P2 — repo weight & hygiene

**12. 2.5 GiB pack.** Large media is committed as plain git blobs, not LFS:
`pack-previews/*/preview/*.gif` (22 MB, 20 MB, 14 MB, 12 MB, 11 MB …),
`ConditioningControlPanel/Resources/spiral.gif` (8.6 MB),
`ConditioningControlPanel/Resources/Modassets/drone/logo.png` (6.7 MB), achievement PNGs
(~5 MB each), the Twemoji set (~19 MB / ~4000 files), avatar-emote GIFs. `.gitattributes`
LFS-tracks only `*.ccpmod` / `*.mp4` / `*.mov`. Every historical revision of every GIF is
in the pack, so a fresh clone is ~2.5 GB.
- **Now:** add `*.gif` / `*.png` / `*.webp` to `.gitattributes` so new additions go to LFS.
- **Scheduled:** history rewrite (`git lfs migrate import --include='*.gif,*.png,*.webp' --everything`
  or BFG). Disruptive — every clone must be re-made — so it is a deliberate, announced call.

**13. `.gitignore` is ~4 accreted "build outputs" blocks.** `bin/` / `obj/` / `publish/` /
`release/` are repeated, `[Rr]elease/` appears twice, and there are three separate
contradictory negation stacks (`redist/` → `!redist/` → `redist/*` → `!redist/…`; same
shape for `tools/` and `DroneMod/`). It functions, but it is not reason-about-able. One-time
dedup.

**14. `nul` entry in `.gitignore`** — fingerprint of a stray `nul` file created by a
`> nul` redirect run under bash on Windows. `git rm` it if tracked, drop the ignore line.

**15. `installer-content-deletions.iss` (878 KB) committed** — confirm it is regenerated per
release and not hand-drifting.

---

## P3 — architecture (shape of future pain, not this-week work)

**16. God files, growing fast:** `App.xaml.cs` 4,995 LOC (2,720 in June 2026 — doubled in
3 months), `Models/AppSettings.cs` 8,147, `Services/Video/VideoService.cs` 8,436,
`Services/Settings/ProfileSyncService.cs` 5,072, `Services/BubbleService.cs` 5,014.
`MainWindow` is **84 partial files**. Partials keep the compiler happy; there is no
component / view-model extraction, so every feature bolts onto `MainWindow` and `App`.

**17. Service-locator via nullable static `App.X`.** The ~90 `CS8602` warnings *are* this:
`App.Flash?`, `App.Patreon?` everywhere because the fields are typed `T?` and are null until
startup finishes. A non-null accessor (`App.Require<T>()`, or non-null field assignment at
init) removes most of the nullable noise and turns "service used before init" into a
compile error.

---

## Localization drift (its own bucket)

| file | keys | vs `en.json` |
|---|---|---|
| `en.json` | 5,409 | — |
| `zh-CN.json` | 5,352 | −57 |
| `de` / `fr` / `ko` / `pt-BR` / `ru` | 4,848 | **−561** |
| `es` / `ja` | 4,841 | **−568** |

The eight non-English files are 60–570 keys behind English — roughly 10% of the UI renders
English text to every non-English user. Per-key English fallback (`LocalizationManager`,
CLAUDE.md #13) means no crash, only untranslated strings. The two-tier clustering
(4,848 vs 4,841) indicates two past bulk-translation passes and nothing since.

---

## Already clean — do not spend time here

- Version strings consistent at **6.9.0** across `ConditioningControlPanel.csproj`,
  `Services/Update/UpdateService.cs` (`AppVersion`), `installer.iss` (`MyAppVersion`),
  `build-installer.bat` (`VERSION`).
- All 9 `Localization/Languages/*.json` parse as **strict JSON** (the 2026-07-29 fix held).
- **No secrets** in tracked files; `Services/LogScrubber.cs` actively redacts tokens/PII
  from bug-report payloads.
- **No deprecated or vulnerable direct packages** (only the two stale `System.*` transitives
  in P0 #5).
- Solution builds clean; `global.json` present (added 2026-09-01).
- 12 `TODO`/`FIXME`/`HACK` comments across ~200 KLOC.

---

## Every direct package is behind

Not all need bumping (majors carry risk), but flagged for a deliberate pass. Priority:
`Microsoft.Web.WebView2` (security-sensitive, load-bearing, ~1600 builds behind),
`Microsoft.WindowsAppSDK` (`1.1.1` → `2.4.0`, and it carries the `ExcludeAssets="all"`
workaround), plus the two CVE transitives above.

| Package | Requested | Latest |
|---|---|---|
| CommunityToolkit.Mvvm | 8.2.2 | 8.4.2 |
| Hardcodet.NotifyIcon.Wpf | 1.1.0 | 2.0.1 |
| LibVLCSharp.WPF | 3.8.5 | 3.10.1 |
| MahApps.Metro | 2.4.10 | 2.4.11 |
| MahApps.Metro.IconPacks | 5.0.0 | 6.2.1 |
| Microsoft.ML.OnnxRuntime | 1.20.1 | 1.29.0 |
| Microsoft.Web.WebView2 | 1.0.2535.41 | 1.0.4191.47 |
| Microsoft.WindowsAppSDK | 1.1.1 | 2.4.0 |
| NAudio / NAudio.Wasapi | 2.2.1 | 3.0.1 |
| OllamaSharp | 5.4.16 | 5.4.30 |
| OpenAI-DotNet | 8.6.2 | 8.8.8 |
| OpenCvSharp4 (+ runtime.win) | 4.9.0.20240103 | 4.13.0.20260627 |
| org.k2fsa.sherpa.onnx (+ runtime) | 1.13.3 | 1.13.5 |
| QRCoder | 1.6.0 | 1.8.0 |
| Serilog | 3.1.1 | 4.4.0 |
| Serilog.Sinks.Console | 5.0.0 | 6.1.1 |
| Serilog.Sinks.File | 5.0.0 | 7.0.0 |
| SharpVectors | 1.8.4.2 | 1.8.5 |
| SIPSorcery | 10.0.14 | 10.0.16 |
| SkiaSharp.Views.WPF | 2.88.8 | 4.151.1 |
| System.Security.Cryptography.ProtectedData | 8.0.0 | 10.0.11 |
| VideoLAN.LibVLC.Windows | 3.0.21 | 3.0.23.1 |
| XamlAnimatedGif | 2.3.0 | 2.3.2 |
