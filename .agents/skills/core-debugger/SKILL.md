# Skill: Core Debugger

## Trigger
Activated when handling concurrent async event overlaps, unhandled promise rejections, memory leaks over streams, or dropped websocket connections.

## Objective
Systematically isolate failures specifically within real-time streaming architectures.

## Instructions
1. Always enable raw byte tracing internally before modifying code constraints.
2. Check network `close` bounds against active memory locks (like `isProcessing` booleans in JS architecture).
3. If audio logic fails, verify the precise chunk boundary arrays before concluding the logic is incorrect (Check 16-bit symmetrical compliance vs arbitrary buffer truncations).
