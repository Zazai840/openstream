# Manual check: settled dictation pipeline

Issue [#105](https://github.com/Nabzx/openstream/issues/105) covers the macOS boundaries that CI cannot drive: global key events, a real microphone, menu bar and overlay feedback, and insertion into another application.

## Run the wizard

From the repository root:

```bash
./scripts/verify-dictation-pipeline.sh
```

The wizard checks the build, then walks through seven stages:

1. Run the automated test suite and typecheck on an Apple Silicon Mac.
2. Start OpenStream and grant Microphone, Input Monitoring, and Accessibility permissions.
3. Use `Cmd+Shift+D` outside OpenStream and inspect the tray, push-to-talk overlay, and sound meter.
4. Dictate into TextEdit and confirm the finished text is inserted once.
5. Inspect both model-server listeners and sample their TCP connections during a dictation. Ports 8178 and 8179 must stay on `127.0.0.1`.
6. Measure three warm dictations from key release to confirmed insertion. Every measurement must be below 1000 ms.
7. Write a Markdown report and optionally post it to issue #105.

The report and application log are written under `${TMPDIR:-/tmp}`. The wizard does not save audio. The application log contains dictated text, so delete it when the check is finished.

## What the timing means

The main process records a monotonic timestamp when the global hotkey reports key-up. It carries that timestamp with the completed recording and logs the elapsed time after the Accessibility helper confirms insertion:

```text
[dictation] release-to-insertion: 742.3ms (within 1000ms budget)
```

This includes WAV finalization, whole-recording transcription, rules cleanup, optional paragraph break placement, context detection, and delivery. Failed or held dictations have no insertion latency because no text arrived at the cursor.

## Troubleshooting

A source build may appear as Electron in System Settings. Check that entry under all three Privacy & Security sections. Quit and relaunch after changing Input Monitoring or Accessibility.

On a cold start, the resident transcription model server can spend 15 to 20 seconds loading Metal shaders. The wizard waits up to 120 seconds for each local listener.

Use the prefixes in the wizard's application log to locate failures:

- `[hotkey-helper]` for Input Monitoring and global key events
- `[accessibility-helper]` for context detection and insertion
- `[transcription model server]` for port 8178 and transcription
- `[rewrite model server]` for port 8179 and paragraph placement
- `[dictation]` for pipeline outcomes and latency
