# Exception Handling & Log Visibility — Remediation Plan

**Date:** 2026-09-01
**Against:** `main` @ v6.9.0
**Method:** `git grep` over tracked `.cs` (counts are directional — pattern matches, not a
semantic audit), plus inspection of the Serilog bootstrap in `App.xaml.cs`.

This is a plan, not a change. It exists to be argued over before anyone touches 3,000 call
sites.

---

## The finding, stated conservatively

| Pattern | Count (tracked `.cs`) |
|---|---|
| `catch (Exception …)` total | ~4,214 |
| fully empty `catch {}` / `catch (…) {}` | ~2,159 |
| `catch { … Logger?.Debug(… }` (first statement is a Debug log) | ~999 |
| `Logger?.Debug(` / `Logger.Debug(` call sites (anywhere) | ~2,405 |
| `Log.Debug(` static-Serilog call sites (anywhere) | ~536 |
| `Verbose(` call sites | 0 |

Overlaps are unmeasured. The load-bearing facts:

1. **~2–3k places where a caught exception produces no durable record in a shipped build** —
   either an empty `catch {}` (no log at all) or a `catch` whose only output is `Debug`.
2. **Every `Debug`/`Verbose` call in the app is discarded, in every build** — see §2. So the
   ~999 "logged" swallows are, in the field, indistinguishable from the ~2,159 silent ones.
3. There is **no gate**: C# emits no diagnostic for `catch {}`, `CA1031` is off by default,
   and there is no CI. Nothing stops the count from rising.

### Threat model (what to actually claim)

Not "3,000 hidden bugs." The app ships and runs, so the overwhelming majority of these
`catch` blocks fire rarely or never. The damage is **diagnostic**: when something *does*
fail, the user sees "the feature just doesn't work," files a bug report, and `logs/crash.log`
is **empty** — because the exception was caught and dropped three frames down. The team is
blind to its own failure modes, and so is every bug report.

### Why it looks the way it does

Almost certainly defensive accretion, not concealment. The codebase has documented
process-death incidents from unhandled exceptions in fire-and-forget / dispatcher /
event-handler paths (`CLAUDE.md` Known Issues #7/#8/#9). Each `catch {}` was likely added
after one crash. It metastasised because there was never a policy or a review gate.
Wholesale removal or blanket `throw;` would reintroduce the crashes — this must be phased.

---

## 2. Why the ~1,000 `Debug` swallows emit nothing — and how much else is affected

### Root cause

`App.xaml.cs:1515`:

```csharp
Logger = new LoggerConfiguration()
    .MinimumLevel.Information() // Security: Changed from Debug to avoid exposing sensitive data in logs
    .WriteTo.File(Path.Combine(logPath, "app-.log"),
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 7,
        flushToDiskInterval: TimeSpan.FromSeconds(1))
    .CreateLogger();
```

`MinimumLevel.Information()` is a **pipeline-front filter**. Serilog evaluates it *before* an
event is constructed or enriched, so every `Logger.Debug(...)` / `Log.Debug(...)` /
`Logger.Verbose(...)` call is dropped at the door. There is:

- exactly **one sink** (`WriteTo.File`), with **no** `restrictedToMinimumLevel` override;
- **no** `LoggingLevelSwitch` (nothing can raise the level at runtime);
- **no** `#if DEBUG` branch (dev builds are just as blind as shipped builds);
- **no** config binding — `appsettings.json` carries `"Logging"` / `"LogLevel": "Information"`
  keys that **nothing reads** (the level is hard-coded above). `appsettings.json` is also
  stale in other ways: `"Version": "2.1.0"`, `"EnableAutoUpdate": false`.

### Blast radius

This is not a catch-block problem — it is every `Debug` call in the app:

- **~2,405** `Logger?.Debug(...)` + **~536** `Log.Debug(...)` call sites, all discarded,
  always. That is dead code in every build.
- **0** `Verbose(...)` calls — developers have learned not to bother.
- **~10** comments in the code explicitly say some variant of *"logging this at Information,
  not Debug, because Serilog's floor is Information"* — i.e. the team is already routing
  around its own config, promoting breadcrumbs to `Information` so they survive, which
  pollutes the shipped log with noise the `Information` floor was meant to keep out.

The `// Security:` intent was legitimate but the fix was aimed one level too shallow. It
suppressed the *tier* the sensitive data happened to be at, instead of stopping the
sensitive data from being logged and instead of redacting the sink. The cost was the entire
`Debug` tier.

### Why PII is in `Debug` in the first place

Grep of every log level: **raw secrets are not being logged** — tokens are referenced but
never their values ("no access token for {Provider}", "tokens stored successfully"), and the
codebase uses structured templates almost exclusively (0 interpolated `Debug($"…")` vs
~2,239 `Debug("…{X}…", x)`), so redaction *is* mechanically possible.

What lands in `Debug` is **user-identifying context**, because `Debug` is the "what is the
user doing right now" tier and this app records a lot of it:

- **Window / tab awareness** — active window titles and browser tab names (the README
  privacy section covers this feature). e.g. a document title or `"Re: … — Gmail"`.
- **OCR** (`ScreenOcrService`) — screen text scraped for keyword triggers.
- **Companion chat** — the user's own messages, prompt content, personality config.
- **File paths** — `C:\Users\<real name>\…`. `LogScrubber` carries a regex *specifically*
  for `Users\<name>`, which tells you this was a known leak.
- **Account** — display names, Patreon tier, Discord handle.

Someone almost certainly opened an attached `app-*.log` from a bug report, saw a real name
in a path or a tab title, and raised the floor because that context mostly lives at `Debug`.

### The `LogScrubber` gap

`Services/LogScrubber` (regex redaction of home paths, emails, OAuth/bearer/Discord tokens,
`%APPDATA%` vars) is invoked in **exactly one place** — `BugReportService.cs:142` and
`:178`, at the moment a report is assembled. So the on-disk `app-*.log` is **never
scrubbed**; it sits in plaintext with 7-day retention, and redaction only happens *if* a
bug report is filed *and* the assembler runs.

### The fix, in order

**1 — primary: stop logging PII, and redact at every sink (a security fix, not hardening).**

- **Policy:** nothing that identifies a person or reflects their screen / files / messages
  goes into a log at *any* level — IDs, counts, enums, status codes only, never content.
  Audit the awareness / OCR / companion / path call sites first; they are the known sources.
- **Enforcement at the sink:** wrap every Serilog sink with the `LogScrubber` regex set (an
  `ILogEventSink` decorator, or a `Destructuring`/format-time filter). A mistake is then
  redacted on the way to disk, always — not only if a report is filed. This is the change
  that actually makes the security concern go away.
- With scrubbing at the sink, **every level is safe**, and `MinimumLevel` can drop back to
  `Debug`. The ~999 `Debug` swallows start recording with no per-catch edits; only the
  ~2,159 *empty* catches still need work.

**2 — secondary: level control, now an ergonomics choice, not a security control.**

- `MinimumLevel.ControlledBy(new LoggingLevelSwitch(LogEventLevel.Information))`, flippable
  to `Debug` at runtime via a Settings toggle or `--verbose` arg — so support can say *"turn
  verbose on and reproduce"* without a rebuild.
- Optionally a second `debug-.log` sink at `restrictedToMinimumLevel: Debug` with 1–2 day
  retention, purely to keep the primary file small. It is **not** a containment boundary —
  the scrubber above is — so `BugReportService` may include it or not, freely.
- Delete or bind the dead `Logging` / `LogLevel` keys in `appsettings.json`.

---

## Remediation phases

### Phase 0 — stop the bleeding (1 small PR)

- `.editorconfig`: `dotnet_diagnostic.CA1031.severity = warning`.
- `audits/swallow-baseline.txt`: today's counts (empty `catch {}`, `Debug`-only `catch`).
  A pre-push / CI script fails if either rises. The number may only ever go **down**.
- New helper `App.Diag.Swallowed(Exception ex, [CallerMemberName], [CallerFilePath],
  [CallerLineNumber])` — one call, logs at `Warning` with `{Swallowed:true}` +
  file/line, so every intentional swallow is greppable and located automatically.

### Phase 1 — fix logging at the root: no PII + scrub every sink (1 PR)

- Wrap every Serilog sink with the `LogScrubber` regex set (`ILogEventSink` decorator or a
  format-time filter). On-disk logs are now redacted unconditionally, not just on export.
- Audit and fix the known PII sources (window/tab awareness, OCR, companion chat, user
  paths) so content is never passed as a log argument in the first place.
- With scrubbing at the sink, `MinimumLevel` returns to `Debug`; add the
  `LoggingLevelSwitch` + a "Verbose logging" Settings toggle for control (not containment).
- Delete or bind the dead `Logging` / `LogLevel` keys in `appsettings.json`.
- Outcome: the ~999 `Debug`-logging catches record again with no catch edits, and the
  security concern that motivated `MinimumLevel.Information()` is closed properly.

### Phase 2 — instrument the empty catches (mechanical, days, reviewed per file)

- A Roslyn analyzer + code-fix that rewrites `catch {}` →
  `catch (Exception ex) { App.Diag.Swallowed(ex); }`, run across the tree, reviewed
  file-by-file (start with the worst — see table).
- Genuinely intentional no-ops (e.g. `catch { /* window tearing down */ }`) keep the comment
  and take `App.Diag.Swallowed(ex, note: "...")` or an explicit `// swallow: <reason>` that
  the analyzer allowlists.

**Worst single files by empty-`catch {}` count:**

| File | empty catches |
|---|---|
| `Services/Chaos/ChaosModeService.cs` | 91 |
| `App.xaml.cs` | 74 |
| `Views/Deeper/EnhancementPlayerWindow.xaml.cs` | 73 |
| `Views/Deeper/DeeperEditorWindow.xaml.cs` | 61 |
| `Windows/TutorialOverlay.xaml.cs` | 44 |
| `Services/BubbleService.cs` | 43 |
| `Services/Flash/FlashService.cs` | 36 |
| `Services/Chaos/DtrhHostService.cs` | 30 |
| `Services/Possession/PossessionDirector.cs` | 29 |
| `Chaos/ChaosWebViewHost.cs` | 27 |

### Phase 3 — triage what visibility reveals (ongoing)

Once shipped builds report swallows at `Warning`, sort `{Swallowed:true}` events by
frequency. The top ~20 sites will be most of the volume, and each is a *"this operation
fails constantly and nobody knew."* Fix those at the root (handle properly, or fix the cause
so the `try` can't throw). The long tail stays swallowed-but-logged.

### Phase 4 — policy

- `CA1031` stays `warning`; the baseline number only decreases.
- Review rule: no new empty `catch {}`. An intentional swallow requires a one-line reason
  comment **and** an `App.Diag.Swallowed(ex)` call.
- No new `Log.Debug` promoted to `Information` "so it shows up" — verbose logging is a real
  tier again, use it.

---

## Effort / risk

| Phase | Effort | Risk |
|---|---|---|
| 0 | ~half a day | none (config + one helper + a count script) |
| 1 | ~1–2 days | low–medium (sink decorator + PII call-site audit; smoke-test bug report is still clean, verify redaction on a seeded PII line) |
| 2 | days–weeks, incremental | low per file, reviewed; behaviour unchanged (adds logging only) |
| 3 | ongoing | this is where real bugs get fixed |
| 4 | continuous | social, not technical |

This is a multi-week effort touching hundreds of files. It needs maintainer buy-in — it is
not a weekend PR. Phases 0 and 1 alone (≈2 days) close the PII exposure properly and convert
the app from *blind* to *instrumented*, which is 80% of the value.
