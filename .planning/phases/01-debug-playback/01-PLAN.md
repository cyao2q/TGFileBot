# Phase 1: Debug Playback - Implementation Plan

## Objective
Implement Option B (Fast Fail & Retry) to resolve the cold-start connection timeout issue when fetching the first media chunk.

## Requirements Covered
- [BUG-001]: Resolve initial connection timeout in `/stream`.

## Context & Constraints
- The `DownloadChunk` function in `stream.go` currently uses a hardcoded 90-second timeout.
- During a cold start, the TCP connection might be dead, causing the MTProto client to hang for the full 90 seconds.
- The player client gives up and cancels the HTTP request after ~20 seconds.
- A fast fail will allow the existing backoff/retry logic in `stream.go` (which retries up to 6 times for initial chunks) to kick in and re-establish the connection.

## Plan Steps

1. **Edit `stream.go`**:
   - Locate the `Client.DownloadChunk` call at line ~191.
   - Modify the timeout parameter from `90*time.Second` to `10*time.Second` to ensure it fails fast enough before the HTTP client times out.
   - Example change:
     ```go
     content, fileName, err := stream.Client.DownloadChunk(srcCopy, int(task.ContentStart), int(task.ContentEnd), int(stream.ChunkSize), false, stream.Ctx, 10*time.Second)
     ```

2. **Verify Retry Logic**:
   - Ensure that the error returned by the timeout falls into the `default` case of the `err != nil` switch, triggering the exponential backoff block (`time.Sleep(backoff)` and `continue`).
   - The current logic handles standard network errors well, so simply reducing the timeout should naturally harness this mechanism.

## Dependencies
- None. This is an isolated modification within `stream.go`.

## Testing Strategy
- Start the server, wait for a long idle period (e.g., > 1 hour) or simulate a dropped TCP connection.
- Initiate a stream request for a file.
- Observe the logs to ensure a quick timeout followed by a successful retry and subsequent chunk downloads.
