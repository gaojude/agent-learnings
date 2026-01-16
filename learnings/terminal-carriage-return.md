# Terminal Cursor Control for Progress Indicators

**Date:** 2026-01-16

## Basics

```javascript
\r   // carriage return - go to start of CURRENT line (can't go up)
\n   // newline - go to next line
```

## Overwrite Current Line

```javascript
process.stdout.write("Loading...");
process.stdout.write("\rDone!      ");  // \r goes to start, spaces clear leftover
```

Multiple `\r` doesn't help - it just keeps going to start of same line.

## ANSI Escape Codes (for more control)

```javascript
\x1b[nA   // move UP n lines
\x1b[nB   // move DOWN n lines
\x1b[nD   // move LEFT n chars
\x1b[nG   // move to column n
\x1b[2K   // clear entire line
```

Example - update 2 lines up:
```javascript
process.stdout.write("\x1b[2A\x1b[2K\rNew content\x1b[2B");
```

## Breaks with Parallel Processes

Multiple processes write to same line → `\r` overwrites garbage:
```
"Loading A...Loading B..." → "\rDone A!     \n" → "Done A!     oading B..."
```

**Fix:** Use `console.log()` (prints on completion with newline) for parallel output.
