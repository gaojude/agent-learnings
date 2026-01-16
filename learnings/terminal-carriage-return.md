# Terminal \r (Carriage Return) for Progress Indicators

**Date:** 2026-01-16

**Project:** next-evals-oss (CLI tool with parallel eval execution)

## Summary

The `\r` (carriage return) character moves the cursor back to the start of the current line without advancing to a new line. This is commonly used for progress indicators that update in place.

## How It Works

```javascript
// Basic overwrite
process.stdout.write("Hello");
process.stdout.write("\rWorld");
// Output: "World" (overwrites "Hello")

// Progress indicator
process.stdout.write("Loading...");
// ... do work ...
process.stdout.write("\rDone!      ");  // spaces to clear leftover chars
// Output: "Done!      "

// Counter/percentage
for (let i = 0; i <= 100; i++) {
  process.stdout.write(`\rProgress: ${i}%`);
}
```

## Key Differences

- `\r` = carriage return (go to start of line, stay on same line)
- `\n` = newline (go to next line)
- `\r\n` = both (Windows line ending)

## Breaks with Parallel Processes

When multiple processes write to stdout concurrently without coordination:

```javascript
// Process A writes: "Loading A..."
// Process B writes: "Loading B..." (same line: "Loading A...Loading B...")
// Process A finishes, writes: "\rDone A!     \n"
// Result: "Done A!     oading B..." (only partially overwrote)
```

The `\r` only goes back to the start of the current line, but that line now contains concatenated output from multiple processes. Padding spaces may not be enough to fully overwrite.

## Solution for Parallel Output

For parallel execution, either:
1. Print only on completion with `console.log()` (includes newline)
2. Use ANSI cursor positioning (`\x1b[<line>;0H`) to move to specific lines
3. Use a library like `ora` or `listr` that handles concurrent spinners
