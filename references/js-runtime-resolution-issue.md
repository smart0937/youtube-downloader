# JS Runtime & Resolution Cap Issue

## Symptom
When downloading YouTube videos, the resolution is capped at 360p (or lower) even when the user requests 720p or higher, and `yt-dlp` outputs the following warning:
`WARNING: [youtube] No supported JavaScript runtime could be found. Only deno is enabled by default... YouTube extraction without a JS runtime has been deprecated, and some formats may be missing.`

## Root Cause
YouTube has moved higher-resolution format manifests behind JavaScript-encrypted endpoints. Without a JS runtime (Node.js or Deno) installed in the environment, `yt-dlp` cannot "unlock" these manifests during the automatic `bestvideo` selection process.

## Resolution Workflow (Precision Strike)
Since the automatic selection is blind to these formats, the agent must manually identify and request them:

1. **List All Formats**: Run `yt-dlp -F [URL]`.
2. **Identify IDs**: Scan the `ID` column for the desired resolution (e.g., `720x1280`).
   - Example: Video ID `298` (720p60 mp4), Audio ID `140` (m4a).
3. **Force Download**: Use the explicit ID combination:
   `yt-dlp -f "298+140/best" [URL]`

## Example Log Evidence
- **Wrong result**: Requested 720p $\rightarrow$ Got 360p $\rightarrow$ JS Runtime warning present.
- **Correct result**: `yt-dlp -F` $\rightarrow$ Found ID 298 $\rightarrow$ `-f "298+140"` $\rightarrow$ Got 720p.
