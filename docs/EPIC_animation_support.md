# Epic: Animation Support in ArtStash Live Preview

## Overview

The ArtStash fork of Online3DViewer (`artstash/Online3DViewer`) now supports animation playback for FBX, glTF, and Collada models. This epic covers integrating that capability into the ArtStash live preview system (React) and the puppeteer-based snapshot pipeline.

The engine changes are complete and backward-compatible — no existing functionality is affected. This epic focuses on frontend integration and UX.

---

## Background

### What Changed in the Engine

The upstream Online3DViewer discards all animation data during import. Our fork preserves it by passing the original Three.js scene (with SkinnedMesh, bones, and AnimationClips intact) through to the viewer when animations are detected.

**New `EmbeddedViewer` API methods:**

| Method | Returns | Description |
|--------|---------|-------------|
| `GetAnimationClips()` | `THREE.AnimationClip[]` | List of animation clips in the loaded model |
| `PlayAnimation(index)` | `void` | Play clip by index |
| `PauseAnimation()` | `void` | Pause current playback |
| `ResumeAnimation()` | `void` | Resume paused playback |
| `StopAnimation()` | `void` | Stop and reset playback |
| `SetAnimationTime(seconds)` | `void` | Seek to a specific time |
| `IsAnimationPlaying()` | `boolean` | Whether animation is currently playing |

These methods are no-ops on non-animated models, so existing code needs no changes.

### How to Install the Fork

**Option A — npm from GitHub (recommended for React projects):**
```bash
# Remove upstream package if installed
npm uninstall online-3d-viewer

# Install from ArtStash fork
npm install artstash/Online3DViewer
```

Import usage remains the same:
```js
import * as OV from 'online-3d-viewer';
```

**Option B — Bundled script tag:**
```bash
# Build the engine bundle
cd Online3DViewer
npm install && npm run build_engine_dev
```
Then reference `build/engine_dev/o3dv.min.js` (dev) or run `npm run build_engine` for `build/engine/o3dv.min.js` (production). The global `OV` namespace is available as before.

---

## Tasks

### 1. Update Online3DViewer Dependency

**Goal:** Replace the upstream `online-3d-viewer` package with the ArtStash fork.

- Update `package.json` to point to `artstash/Online3DViewer`
- Verify the existing live preview still works identically with non-animated models
- Run existing tests / smoke test the preview with known 3D assets

**Acceptance criteria:** All current 3D preview functionality works unchanged.

---

### 2. Detect Animated Models on Load

**Goal:** After a model loads, check if it contains animations and surface that to the UI.

```jsx
// In your viewer component's onModelLoaded callback:
const clips = embeddedViewer.GetAnimationClips();
if (clips.length > 0) {
  setAnimationClips(clips.map((clip, i) => ({
    index: i,
    name: clip.name || `Animation ${i + 1}`,
    duration: clip.duration
  })));
  setHasAnimations(true);
}
```

**Acceptance criteria:** The UI knows whether the loaded model has animations and has access to clip metadata (name, duration, count).

---

### 3. Build Animation Playback Controls (React Component)

**Goal:** Create a `<AnimationControls>` component that appears when an animated model is loaded.

**Suggested UI elements:**
- Play / Pause toggle button
- Stop button (resets to first frame)
- Clip selector dropdown (if multiple clips)
- Time scrubber / progress bar showing current position
- Duration label (e.g., "2.40s / 14.00s")

**Suggested component API:**
```jsx
<AnimationControls
  clips={animationClips}        // Array of { index, name, duration }
  isPlaying={isPlaying}
  onPlay={(clipIndex) => embeddedViewer.PlayAnimation(clipIndex)}
  onPause={() => embeddedViewer.PauseAnimation()}
  onResume={() => embeddedViewer.ResumeAnimation()}
  onStop={() => embeddedViewer.StopAnimation()}
  onSeek={(time) => embeddedViewer.SetAnimationTime(time)}
/>
```

**Acceptance criteria:** Users can play, pause, stop, and scrub through animations on supported models. Controls are hidden for non-animated models.

---

### 4. Update Time Display During Playback

**Goal:** Keep the time scrubber and duration label in sync while animation plays.

The animation runs inside the viewer's `requestAnimationFrame` loop. To read the current time, poll from the component:

```jsx
useEffect(() => {
  if (!isPlaying) return;
  const interval = setInterval(() => {
    const viewer = embeddedViewer.GetViewer();
    if (viewer.animationMixer) {
      const clip = clips[currentClipIndex];
      const time = viewer.animationMixer.time % clip.duration;
      setCurrentTime(time);
    }
  }, 100); // 10fps update is sufficient for UI
  return () => clearInterval(interval);
}, [isPlaying, currentClipIndex]);
```

**Acceptance criteria:** Time scrubber updates smoothly during playback. Seeking via scrubber works in both playing and paused states.

---

### 5. Handle Animation in Puppeteer Snapshot Pipeline

**Goal:** Enable the puppeteer-based renderer to capture frames from animated models for rotational videos.

**Approach:** Use the new API to seek to specific animation frames before capturing screenshots:

```js
// In puppeteer evaluation context:
async function captureAnimationFrames(viewer, clipIndex, frameCount) {
  const clips = viewer.GetAnimationClips();
  const duration = clips[clipIndex].duration;
  const frames = [];

  for (let i = 0; i < frameCount; i++) {
    const time = (i / frameCount) * duration;
    viewer.SetAnimationTime(time);
    // Wait one frame for Three.js to update the pose
    await new Promise(r => requestAnimationFrame(r));
    const dataUrl = viewer.GetViewer().GetImageAsDataUrl(width, height, false);
    frames.push(dataUrl);
  }
  return frames;
}
```

**Acceptance criteria:** Puppeteer can generate a sequence of frames from an animated model at specified time intervals, suitable for composing into a video.

---

### 6. Add Animation Badge / Indicator to Asset Cards

**Goal:** When browsing assets in ArtStash, indicate which 3D models contain animations.

**Approach options:**
- Detect animation clips at upload/ingest time and store as asset metadata
- Or detect on first preview load and cache the result

Display a small badge (e.g., "🎬 Animated" or a play icon) on the asset card thumbnail.

**Acceptance criteria:** Users can visually identify animated vs. static 3D assets before opening the preview.

---

### 7. Test with Target Formats

**Goal:** Verify animation playback works with the file formats ArtStash users commonly upload.

**Test matrix:**

| Format | Source | Expected |
|--------|--------|----------|
| FBX (binary) | Mixamo character export | Skeletal animation plays |
| FBX (embedded textures) | Blender FBX export | Animation + textures render |
| glTF / GLB | Sketchfab download | Keyframe animation plays |
| Collada (.dae) | Unity export | Animation plays |
| Static FBX (no animation) | Any non-animated FBX | Renders normally, no controls shown |
| Multi-clip FBX | Mixamo with multiple animations | Clip selector works |

**Acceptance criteria:** All formats in the test matrix work correctly. Non-animated models show no animation controls.

---

## Technical Notes

- **Backward compatibility:** The animation API methods are no-ops when no animation data is present. Existing code paths are unaffected.
- **Performance:** The `requestAnimationFrame` render loop only runs during active playback. When paused or stopped, rendering returns to the original on-demand mode (only on user interaction).
- **Texture handling:** Some FBX files reference external textures that may fail to load. The engine automatically cleans up broken texture references and falls back to diffuse color.
- **Upstream sync:** The `origin` remote tracks `kovacsv/Online3DViewer`. To pull upstream fixes: `git fetch origin && git merge origin/master`.

## Reference

- **Fork repo:** https://github.com/artstash/Online3DViewer
- **Test page:** After building (`npm run build_engine_dev`), open `sandbox/animation_test.html` via a local server to test with any animated model.
- **Commit:** `e1ba523` on `master` — "Add animation support for FBX, glTF, and Collada models"
