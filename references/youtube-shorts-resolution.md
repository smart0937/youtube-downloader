# YouTube Shorts Resolution & Filter Logic

## The "Shorts Trap"
In `yt-dlp`, resolution is typically handled as `width x height`. 
- **Horizontal Video (Standard)**: 720p is `1280x720` (Height = 720).
- **Vertical Video (Shorts)**: 720p is `720x1280` (Width = 720).

If a filter only specifies `height=720`, it will fail for Shorts and fall back to the next available format (often 360p, where height is 640), leading to lower-than-expected quality.

## Verified Format Selection Logic
To ensure 720p regardless of orientation, use a combined selector:
`"bestvideo[width=720]+bestaudio/bestvideo[height=720]+bestaudio/best"`

### Logic Breakdown:
1. `bestvideo[width=720]+bestaudio`: Matches Vertical 720p (Shorts).
2. `/`: OR operator.
3. `bestvideo[height=720]+bestaudio`: Matches Horizontal 720p.
4. `/best`: Final fallback to the best available if neither specific 720p match is found.

## Example Format List (-F) for Shorts
When running `yt-dlp -F`, look for these patterns:
- **Wrong (360p)**: `360x640` -> Height is 640.
- **Right (720p)**: `720x1280` -> Width is 720.
