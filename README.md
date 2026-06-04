# Gemini Fallback Indexing Report

## Purpose

This report explains how the project creates video indexing when **`GEMINI_API_KEY` is not provided**.

## Short Answer

If Gemini is unavailable, the app does **not stop**. It automatically falls back to a local chunking method that splits the transcript into simple time-based blocks.

## Where This Happens

The fallback logic is in:

- [`src/indexer.py`](./src/indexer.py)

The key function is:

- `chunk_semantically_with_gemini(...)`

## What Happens Without a Gemini API Key

When `GEMINI_API_KEY` is missing or invalid:

1. The app checks whether the Gemini SDK is installed.
2. It checks whether `GEMINI_API_KEY` exists in the environment.
3. If the key is missing, the app prints a warning.
4. It then uses the local fallback function:
   - `chunk_segments(...)`

## How the Fallback Indexing Works

The fallback does **not use AI topic detection**.

Instead, it:

1. Reads the Whisper transcript segments.
2. Groups them into blocks of about **60 seconds**.
3. Assigns default titles like:
   - `Section 1`
   - `Section 2`
   - `Section 3`

This gives the video a basic timeline index even without Gemini.

## Transcript Output

The transcript text file is still created next to the video.

If chapter blocks are available, the file includes:

- an `Index` section
- chapter headings
- timestamped transcript lines under each section

If no semantic blocks are available, the file falls back to plain timestamped transcript lines.

## Example Fallback Behavior

Without Gemini:

- `00:00 - Section 1`
- `01:00 - Section 2`
- `02:00 - Section 3`

This is a timing-based index, not a meaning-based chapter split.

## Important Difference

### With Gemini

- The chapter titles are semantic
- Topics are based on meaning and context
- Example: `Introduction`, `Grammar`, `Vocabulary`, `Reading`, `Conclusion`

### Without Gemini

- The chapter titles are generic
- Blocks are based on time intervals
- Example: `Section 1`, `Section 2`, `Section 3`

## Conclusion

Even without a Gemini API key, the project still creates an indexed transcript.
It simply uses a **local fallback chunker** instead of AI-based semantic chapter detection.
