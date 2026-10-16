# xcode-run

Build an iOS app from the current git worktree and run it on your Mac, a
connected iPhone or iPad, or a simulator, without opening Xcode. The app's
output shows in a console in your terminal that you can filter, copy, and save.

```
$ run
Run MyApp on:
> Mac        My Mac            Mac Catalyst · macOS 27.0
  iPhone     Ada's iPhone      iPhone 17 Pro · iOS 27.0 · USB
  Simulator  iPhone 17         iOS 27.0 · Booted
```

It works well with git worktrees: each worktree builds into its own folder, and
the first build in a new worktree copies the Swift packages from the main
checkout instead of downloading them again.

## Requirements

- macOS with Xcode
- [fzf](https://github.com/junegunn/fzf) 0.63.0 or newer
- [jq](https://jqlang.org)

```sh
brew install fzf jq
```

## Install

```sh
npm install -g xcode-run
```

This installs two names for the same command: `run` and `xcode-run`. Use
`xcode-run` if another tool already uses `run`.

### Tab completion

To complete destinations with Tab in fish, add this line to
`~/.config/fish/config.fish`:

```fish
xcode-run completion fish | source
```

In zsh, add this line to `~/.zshrc`, after `compinit`:

```sh
eval "$(xcode-run completion zsh)"
```

## Use

In a git repository with an iOS app:

```sh
run                  # pick a destination, then build, run, and show the console
run iphone 17        # build and run on the destination that matches, without the list
run --no-build       # run the last build again on the same destination
run setup            # answer the setup questions again
run completion fish  # print the Tab completion for fish (or zsh)
run --help
run --version
```

### First run

The first time you use `run` in a repository, it asks a few questions. Type to
filter each list, press Enter to pick, or press Esc to cancel.

1. **Project**: the Xcode workspace or project. The ones with a scheme that
   builds an app come first.
2. **Scheme**: the scheme that builds and launches the app.
3. **Extra xcconfig**: an optional `.xcconfig` file that overrides build
   settings in `run` builds only. `run` asks only if the project's folder has
   an `.xcconfig` file.
4. **Mac Catalyst**: whether the app runs on your Mac. The default comes from
   `SUPPORTS_MACCATALYST` in the project.

The app name and the build configuration come from the scheme.

### Destinations

The list shows My Mac (if the app supports Mac Catalyst), then connected
iPhones, then connected iPads (USB before network), then iOS simulators (booted
ones first, then the newest iOS versions). The list starts on the destination
that you picked last time.

### Name the destination

To skip the list, name the destination after `run`:

```sh
run mac               # My Mac
run iphone 17         # Ada's iPhone, an iPhone 17 Pro, before any simulator
run ada               # Ada's iPhone
run iphone 17 sim     # the iPhone 17 simulator, not iPhone 17 Pro
run iphone 17 sim 26  # the iPhone 17 simulator with iOS 26
run ipad 9th gen      # an iPad (9th generation)
```

Each word must match the start of a word that the list shows for the
destination, in any case. Then:

- If the words match My Mac or a connected iPhone or iPad, simulators don't
  count. Add `sim` to the words to pick a simulator.
- If the words include all of the words of a destination's name, or of the
  model of an iPhone or iPad, only such destinations count. That's why
  `run iphone 17 sim` doesn't count iPhone 17 Pro.
- If the destinations that count are all simulators with the same name, `run`
  takes the booted one, or else the one with the newest iOS. Add the iOS
  version to pick another one.
- If one destination counts, `run` builds and runs the app there.
- If more than one counts, `run` says how many and lists only them. If none
  does, it lists all of the destinations.

The destination that you name counts as your pick, so the list starts on it
next time.

With [Tab completion](#tab-completion), Tab after `run` offers the
words that narrow down the destinations, with the destinations that each word
matches. It leaves out words that all of those destinations have, so each word
picks fewer of them. A version such as `27.0` is one word.

```
$ run iphone 17 <Tab>
17e        -- iPhone 17e
27.0       -- iPhone 17, iPhone 17 Pro, iPhone 17 Pro Max
ada  usb   -- Ada's iPhone
max        -- Ada's iPhone, iPhone 17 Pro Max
pro        -- Ada's iPhone, iPhone 17 Pro, iPhone 17 Pro Max
simulator  -- iPhone 17, iPhone 17e, iPhone 17 Pro +1 more
```

### Build

The build shows as a live view of the newest steps, with errors in red and
warnings in yellow. Press `v` to read all of the output so far in `less`.
Press Ctrl-C to stop the build.

### Console

After the build, `run` installs and launches the app with the launch arguments
and environment variables that its scheme enables in Xcode. Then it shows the
app's output, newest line at the bottom.

| Key | Action |
| --- | --- |
| Type | Filter the lines. `!word` hides lines, `a \| b` matches either, `^word` matches the start. |
| Esc | Clear the filter. |
| Up, PgUp | Scroll back. New lines don't move the view while you read. |
| Down, PgDn, End | Scroll forward. On the newest line, the view follows new lines again. |
| Ctrl-Y | Copy the lines that match the filter (or the full log) to the clipboard. |
| Ctrl-S | Save the same lines to a file on your Desktop. |
| Ctrl-B | Build the app again while it runs, then stop it and launch the new build. |
| Ctrl-R | Stop the app and launch the same build again, without building or installing it. |
| Ctrl-C | Stop the app and end the session. |
| ? | Show all of the keys. |

### Crashes

If the app crashes, the console shows what happened below the app's last
lines: the exception, and the stack of the thread that crashed (or the
backtrace of an uncaught exception), with source lines. The stack ends after
its last frame in the app.

```
── MyApp crashed: EXC_BAD_ACCESS (SIGSEGV) · KERN_INVALID_ADDRESS at 0x0000000000000010 ──
Thread 0 crashed · com.apple.main-thread
  0  MyApp              PhotoLoader.decode(_:) (PhotoLoader.swift:42)
  1  MyApp              PhotoLoader.load(_:) (PhotoLoader.swift:18)
  2  libdispatch.dylib  _dispatch_client_callout + 16
  …
Crash report: ~/Library/Logs/DiagnosticReports/MyApp-2026-10-14-160138.ips
```

`run` takes the summary from the crash report that the destination writes. It
comes a few seconds after the crash on the Mac and in a simulator, and can take
up to about 15 seconds from an iPhone or iPad. A report from a device has no
source lines, so `run` looks them up in the build with `atos`. `run` copies
reports from devices to `crash-reports` in the build folder. Open a report in
Console to see all of its threads.

If iOS ends the app on a device because of its memory use, the console shows
that and how much memory the app used. That leaves no stack.

### Build and run again

To build the app again without leaving the session, press Ctrl-B in the
console. The build shows in the same live view as the first one, and the app
keeps running while it builds. When the build is done, `run` stops the app,
installs the new build, and launches it. The console comes back with the new
output under a `── Built … ──` line, and the filter stays.

If the build fails, the app keeps running, and the console shows the errors at
the end of the log. Fix them and press Ctrl-B again. Ctrl-C in the build stops
it and goes back to the console. If the new build can't launch, the session
ends and shows why.

### Run again without building

To launch the app again without building it, press Ctrl-R in the console. `run`
stops the app and launches the copy that the destination already has. The new
output goes on in the same console, under a `── Relaunching ──` line, and the
filter stays.

To start a new session without building, use `run --no-build`. It launches
the last build of the worktree on the destination that it was built for,
without the list. On an iPhone, an iPad, or a simulator, it installs the build
first, because another worktree can install a different build of the same app
there.

### Locked devices

An app can't launch on a locked iPhone or iPad. If the device is locked when
the app is ready, `run` waits until you unlock it. It shows a desktop
notification and rings the terminal bell, so you know that it's waiting.
The notification uses OSC 777, which terminals such as
[Ghostty](https://ghostty.org) support. In tmux, the notification gets through
only if `allow-passthrough` is on.

Ctrl-R in the console doesn't wait. If the device is locked, the app keeps
running, and the console asks you to unlock the device and press Ctrl-R again.

## Settings

`run setup` saves the settings in `.git/xcode-run.conf` (in the repository's
main git folder). All worktrees of the repository share them, and git doesn't
track them. You can also edit the file:

| Setting | Meaning |
| --- | --- |
| `project` | The Xcode workspace or project, relative to the worktree root. |
| `scheme` | The scheme to build. |
| `configuration` | The build configuration, such as `Debug`. |
| `app` | The name of the app that the scheme builds, without `.app`. |
| `xcconfig` | An optional `.xcconfig` file that overrides build settings. |
| `mac_catalyst` | `yes` adds My Mac to the destinations. |
| `derived_data` | The build folder, relative to the worktree root. By default, `DerivedData-CommandLine` next to the project. Make sure git ignores it. |

The build folder also keeps `last-build.log` (all `xcodebuild` output),
`last-build-progress.log`, `last-console.log` (all app output), `last-run`
(the last build and its destination, for `run --no-build`), and
`crash-reports` (the crash reports that `run` copies from devices).

## Environment

| Variable | Meaning |
| --- | --- |
| `DEVELOPER_DIR` | The Xcode to use. Without it, `run` uses the Xcode that `xcode-select` points to, or else the newest `/Applications/Xcode*.app`. |
| `XCODE_RUN_SAVE_DIR` | The folder for Ctrl-S. The default is `~/Desktop`. |
| `XDG_CACHE_HOME` | `run` remembers the last destination in `$XDG_CACHE_HOME/xcode-run` (default `~/.cache/xcode-run`). |

## Notes

- Builds use `-onlyUsePackageVersionsFromResolvedFile`, so `run` never changes
  `Package.resolved`.
- On the Mac, `run` quits a running copy of the app first, like Xcode does.

## Uninstall

```sh
npm uninstall -g xcode-run
```

Delete `.git/xcode-run.conf` in each repository and `~/.cache/xcode-run` to
remove the settings too.

## License

MIT
