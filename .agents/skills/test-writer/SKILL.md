# Skill: Test Writer

## Trigger
Activated when the codebase needs unit or integration test coverage, especially concerning streaming interfaces or Websockets.

## Objective
Establish high-coverage testing mechanisms to ensure streaming pipelines (Audio, LLMs, WebSockets) handle parallel I/O securely without drops.

## Instructions
1. Define clear mocked endpoints for external providers (e.g. Deepgram API, Groq).
2. Test both browser-side blobs and native hardware byte streams dynamically.
3. Inject simulated race conditions (like an interrupt mid-speech) and assert that audio buffers are cleared and states reverted.

## Rules
- Isolate IO operations.
- Avoid introducing flaky test patterns that depend on network timing.
