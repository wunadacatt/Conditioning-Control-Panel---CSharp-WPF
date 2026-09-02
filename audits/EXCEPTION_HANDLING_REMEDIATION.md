# Exception Handling & Log Visibility — Remediation Plan

**Date:** 2026-09-01
**Against:** `main` @ v6.9.0
**Method:** `git grep` over tracked `.cs` (initial counts were directional pattern matches),
plus inspection of the Serilog bootstrap in `App.xaml.cs`.

This is a plan, not a change. It exists to be argued over before anyone touches ~2,500 call
sites.

### Maintainer review — 2026-09-01

Reviewed by the maintainer. Diagnosis confirmed: the `Debug` tier is dead and `LogScrubber`
only runs on bug reports. Corrections folded in below:

- **Exact counts:** 2,538 empty `catch` blocks; ~500 `catch` blocks whose only output is
  `Debug`.
- **OCR is clean** — `ScreenOcrService` does not log recognised text. Removed as a PII
  source.
- **The confirmed live PII leak is `KeywordTriggerService`**, which logs matched *screen
  words* at `Information` — i.e. already in the shipped `app-*.log` and reaching bug reports,
  not a `Debug`-tier problem. This is the first call site to fix.
- **`crash.log` does not go through Serilog at all** — it is written directly by the global
  exception handlers. A Serilog sink decorator will not touch it; Phase 1 must wrap that
  writer separately.
- **P0 + P1 accepted, conditionally:** the scrubber must rewrite `LogEvent` *properties*
  (scalar property values), **not** regex the rendered message string, and must ship with a
  test. **P2** (analyzer pass over the empty catches) is deferred pending Phase 1.

---

## The finding, stated conservatively

| Pattern | Count (tracked `.cs`) |
|---|---|
| `catch (Exception …)` total | ~4,200 |
| fully empty `catch {}` / `catch (…) {}` | **2,538** (maintainer count) |
| `catch` whose only output is `Debug` | **~500** (maintainer count) |
| `Logger?.Debug(` / `Logger.Debug(` call sites (anywhere) | ~2,405 |
| `Log.Debug(` static-Serilog call sites (anywhere) | ~536 |
| `Verbose(` call sites | 0 |

The load-bearing facts:

1. **~3,000 places where a caught exception produces no durable record in a shipped build** —
   either an empty `catch {}` (no log at all) or a `catch` whose only output is `Debug`.
2. **Every `Debug`/`Verbose` call in the app is discarded, in every build** — see §2. So the
   ~500 "logged" swallows are, in the field, indistinguishable from the 2,538 silent ones.
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

## 2. Why the ~500 `Debug` swallows emit nothing — and how much else is affected

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

What lands in the logs is **user-identifying context**, because logging is where the app
records "what is the user doing right now" and there is a lot of it:

- **`KeywordTriggerService`** — logs matched *screen words* at **`Information`**. This is the
  confirmed live leak: it is already in the shipped `app-*.log` and travels into bug reports.
  Not a `Debug`-tier issue — fix this call site first, independently of the pipeline work.
- **Window / tab awareness** — active window titles and browser tab names (the README
  privacy section covers this feature). e.g. a document title or `"Re: … — Gmail"`.
- **Companion chat** — the user's own messages, prompt content, personality config.
- **File paths** — `C:\Users\<real name>\…`. `LogScrubber` carries a regex *specifically*
  for `Users\<name>`, which tells you this was a known leak.
- **Account** — display names, Patreon tier, Discord handle.

`ScreenOcrService` was checked and **does not** log recognised text — it is not a source.

The blanket `MinimumLevel.Information()` change was aimed at the `Debug` tier because that is
where most of this *used to* sit; it neither stops PII being logged nor helps the one source
(`KeywordTriggerService`) that logs at `Information`.

### The `LogScrubber` gap — two write paths, neither covered

`Services/LogScrubber` (regex redaction of home paths, emails, OAuth/bearer/Discord tokens,
`%APPDATA%` vars) is invoked in **exactly one place** — `BugReportService.cs:142` and
`:178`, at the moment a report is assembled. Two consequences:

1. The on-disk `app-*.log` (Serilog) is **never scrubbed** — it sits in plaintext with 7-day
   retention; redaction only happens *if* a bug report is filed *and* the assembler runs.
2. **`crash.log` does not go through Serilog at all** — it is written directly by the global
   exception handlers (`DispatcherUnhandledException` / `AppDomain.UnhandledException` /
   `TaskScheduler.UnobservedTaskException`). A Serilog sink decorator cannot see it. That
   writer needs the scrubber applied at its own call site.

### The fix, in order

**1 — primary: stop logging PII, and redact both write paths (a security fix, not hardening).**

- **Fix `KeywordTriggerService` now** — it logs matched screen words at `Information`. This
  is standalone and does not wait on the pipeline work.
- **Policy:** nothing that identifies a person or reflects their screen / files / messages
  goes into a log at *any* level — IDs, counts, enums, status codes only, never content.
  Audit the awareness / companion / path call sites next; they are the known sources.
- **Redact at the source of truth, the `LogEvent` — not the rendered string.** A
  `LogScrubber`-backed `ILogEventEnricher` (or sink wrapper) that walks `LogEvent.Properties`
  and rewrites scalar property *values* in place. String-replacing the final formatted
  message is rejected: it is lossy, order-dependent, and defeats structured querying.
  **Ships with a test** (seed a `LogEvent` carrying a home path / token property, assert the
  written value is redacted and the message template is untouched).
- **Apply the same scrubber to the `crash.log` writer** at its own call site — it bypasses
  Serilog, so the enricher above never sees it.
- With redaction on both write paths, **every level is safe**, and `MinimumLevel` can drop
  back to `Debug`. The ~500 `Debug` swallows start recording with no per-catch edits; only
  the 2,538 *empty* catches still need work.

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

### Phase 1 — fix logging at the root: no PII + redact both write paths (1 PR)

- **`KeywordTriggerService`**: stop logging matched screen words (or log a hash / count).
- **Serilog path**: a `LogScrubber`-backed `ILogEventEnricher` that rewrites scalar
  `LogEvent.Property` values in place — not the rendered string. Applied to the logger
  config so every sink inherits it. **With a test** (seeded PII property → redacted value,
  message template intact).
- **`crash.log` path**: run the same `LogScrubber` at the global exception-handler write
  site — it does not pass through Serilog.
- Audit the remaining known PII sources (window/tab awareness, companion chat, user paths)
  so content is never passed as a log argument in the first place.
- With both paths redacted, `MinimumLevel` returns to `Debug`; add the `LoggingLevelSwitch`
  + a "Verbose logging" Settings toggle for control (not containment).
- Delete or bind the dead `Logging` / `LogLevel` keys in `appsettings.json`.
- Outcome: the ~500 `Debug`-logging catches record again with no catch edits, and the PII
  exposure that motivated `MinimumLevel.Information()` is closed on both write paths.

### Phase 2 — instrument the empty catches (mechanical, days, reviewed per file)

- A Roslyn analyzer + code-fix that rewrites `catch {}` →
  `catch (Exception ex) { App.Diag.Swallowed(ex); }`, run across the tree, reviewed
  file-by-file (start with the worst — see table).
- Genuinely intentional no-ops (e.g. `catch { /* window tearing down */ }`) keep the comment
  and take `App.Diag.Swallowed(ex, note: "...")` or an explicit `// swallow: <reason>` that
  the analyzer allowlists.

**Worst single files by empty-`catch {}` count** (from the initial pattern grep; the
maintainer's whole-repo total is 2,538, so per-file figures may run a little higher):

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
| 1 | ~2–3 days | low–medium (property-rewrite enricher + test, crash.log writer wrap, `KeywordTriggerService` fix, PII call-site audit; verify a bug report is still clean and a seeded PII property is redacted on both paths) |
| 2 | days–weeks, incremental | low per file, reviewed; behaviour unchanged (adds logging only) |
| 3 | ongoing | this is where real bugs get fixed |
| 4 | continuous | social, not technical |

This is a multi-week effort touching hundreds of files. Phases 0 and 1 (≈3 days, maintainer
pre-approved) close the PII exposure on both write paths and convert the app from *blind* to
*instrumented* — 80% of the value. Phase 2 is deferred pending Phase 1.
