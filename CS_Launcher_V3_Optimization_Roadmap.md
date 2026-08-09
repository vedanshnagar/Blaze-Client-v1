# CS Launcher V3 — Complete Performance & FPS Optimization Roadmap

## 1. Executive Summary
This report provides a comprehensive performance and user experience optimization roadmap for CS Launcher V3. Following an in-depth codebase audit, several critical bottlenecks affecting UI responsiveness, input latency, and system overhead were identified. While launcher code rarely affects native Minecraft FPS directly, how the launcher manages background threads, memory allocation, and the rendering bridge does influence overall stability, frame pacing, and thermal throttling. This document outlines exactly what to change, where to change it, and the anticipated impact across low-end to high-end devices.

## 2. Current Performance Architecture
The application runs as a standard Android app bridging into native code via JNI and SDL for Minecraft.
- **Threading:** Heavy use of `Tools.MAIN_HANDLER.post()`, a custom `PojavApplication.sExecutorService` thread pool, and `runOnUiThread()`.
- **Input:** Native touch events handled via `InGameEventProcessor` and `InGUIEventProcessor`, plus SDL/Gamepad polling.
- **Rendering Bridge:** Uses either `SurfaceView` or `TextureView` backed by `MinecraftGLSurface`, forwarding a native surface to OpenGL/Vulkan backends via JNI.
- **Sync/Network:** Real-time Firebase listeners (`FirebaseSyncManager`) and `HttpURLConnection` for downloads.

## 3. Biggest Performance Bottlenecks
1. **Unnecessary `notifyDataSetChanged()` calls:** Widespread usage across multiple RecyclerView adapters (e.g., `RTRecyclerViewAdapter`, `LauncherPreferenceFragment`, `ProfileAdapter`) causes full view rebinding and destroys smooth scrolling.
2. **Main Thread File I/O & Bitmap Decoding:** Explicit file checks (`File.exists()`, `length()`) and `BitmapFactory.decodeFile()` in UI/Main Thread paths, particularly in `MinecraftAccount` skin loading and `CustomCursorRenderer`.
3. **High-Frequency UI Polling:** `CustomCursorRenderer` forces a `postDelayed(16ms)` (60fps loop) to redraw the custom cursor, invalidating the touchpad constantly, leading to excess CPU wakeups and UI thread contention.
4. **Firebase Sync Overhead:** `FirebaseSyncManager` stores massive `JSONObject` instances and parses them repeatedly, leading to memory bloat and potential stuttering when syncing.

## 4. Minecraft FPS Analysis
**Important:** Changing Android UI code will *not* magically increase raw Minecraft FPS, as the game runs within its own JVM and native GL context. However, the launcher *can* negatively affect FPS if it competes for CPU/GPU resources or causes GC pauses.
- **Real FPS Impact:** The custom cursor 60fps loop (`CustomCursorRenderer`) continuously wakes the main thread, stealing CPU cycles from the JVM/native threads.
- **Surface Allocation:** `TextureView` (when selected) introduces an extra compositing step on the GPU compared to `SurfaceView`, reducing performance slightly.
- **GC Interference:** Explicit `System.gc()` calls in `MainActivity` and `ControlLayout` can stall the entire Dalvik/ART VM, interfering with smooth gameplay.

## 5. Frame Pacing Analysis
Frame pacing stutters (micro-stutters) occur when the CPU is blocked right before submitting a frame, or when background tasks consume excessive I/O.
- **Culprit:** The `FpsCounter` tick running every 500ms in `MainActivity` does layout manipulation on the UI thread while the game is rendering.
- **Culprit:** Constant Gamepad polling and Input Event dispatching (`dispatchGenericMotionEvent`) sometimes allocate new `MotionEvent` objects (e.g., `MotionEvent.obtain(event)`), triggering minor GC pauses that disrupt frame pacing.

## 6. Launcher UI Performance
- **RecyclerViews:** The use of `notifyDataSetChanged()` instead of specific item notifications (`notifyItemInserted`, `notifyItemChanged`) forces Android to recalculate the entire list layout, causing severe scroll jank.
- **View Overdraw:** The use of hardware layers for layout animations (e.g., in `ProfileEditorFragment`) can be expensive if not removed after the animation completes.

## 7. Startup Performance
- **Firebase Initialization:** `FirebaseSyncManager.onResume` processes JSON snapshots on the UI thread.
- **Mod/Profile Scanning:** Extensive I/O in profile loading happens partially on the main thread, delaying the time to interactive (TTI).

## 8. RAM & Memory Analysis
- **Bitmap Caching:** `MinecraftAccount` frequently decodes raw skin Bitmaps via `BitmapFactory.decodeFile()` and performs manual rounding/resizing (`Bitmap.createScaledBitmap`, `Bitmap.createBitmap`) without memory pooling or `inBitmap` reuse. This causes massive memory churn.
- **JSON Bloat:** `FirebaseSyncManager` caches all settings and announcements as `JSONObject` strings, repeatedly parsing them.

## 9. CPU Analysis
- **The 16ms Custom Cursor Loop:** `CustomCursorRenderer.mAnimationRunnable` runs every 16ms using `postDelayed`. This constantly wakes the CPU, drains battery, and hurts thermal limits.
- **Controllable Mitigation Thread:** `Tools.startControllableMitigation` creates an infinite loop `while (!Thread.currentThread().isInterrupted())` sleeping and checking folders. This busy-wait thread is inefficient.

## 10. GPU/Rendering Analysis
- **SurfaceView vs TextureView:** `TextureView` is less efficient as it uses an FBO and extra composition. Forcing `SurfaceView` is better for raw FPS.
- **FPS Chip Overdraw:** The draggable FPS/Memory chip forces a redraw on top of the native surface, causing Android's HWC (Hardware Composer) to composite layers constantly.

## 11. Battery Efficiency
- **Excessive Wakeups:** The 16ms custom cursor loop and the mitigation threads prevent the CPU from entering deep sleep states, causing severe battery drain even when the user is idling in menus.
- **Network Polling:** If Firebase falls back to polling, it will hold wakelocks.

## 12. File & Storage Performance
- **Synchronous I/O:** `File.exists()` and `File.length()` checks during adapter binding (`ResourceBrowserDialog`, `ModItemAdapter`) block the UI thread.
- **Scoped Storage:** Android 11+ scoped storage overhead is not properly cached, leading to slow repeated directory listings.

## 13. Network/Download Performance
- **Concurrent Downloads:** The adaptive thread pool in `PojavApplication` is good, but downloading many small files (e.g., mod icons) without HTTP Keep-Alive or connection pooling wastes time in TLS handshakes.

## 14. Input/Touch/Controller Performance
- **Touch Event Allocation:** `MinecraftGLSurface.onTouchEvent` and `dispatchGenericMotionEvent` perform heavy logic. The `MotionEvent.obtain(event)` inside the gamepad loop allocates objects, leading to GC pressure during intense gameplay (e.g., fast camera movement).

## 15. Low-End Device Recommendations
- **Disable Custom Cursor Animations:** Completely disable the 16ms `CustomCursorRenderer` loop to save CPU overhead.
- **Force `SurfaceView`:** Ensure `TextureView` is never used.
- **Disable FPS/Mem Chips:** The UI compositing overhead is too high.

## 16. Mid-Range Device Recommendations
- **Optimize Bitmaps:** Implement Glide or Coil for skin/cape loading and caching.
- **DiffUtil for Adapters:** Replace `notifyDataSetChanged()` with `DiffUtil`.

## 17. High-End Device Recommendations
- **120Hz/144Hz Support:** Ensure `MinecraftGLSurface` sets the preferred display mode to the maximum refresh rate, but decouple the UI polling (like the custom cursor) from hardcoded 16ms (60fps) intervals.

## 18. Prioritized Optimization Table

| Priority | Area | Location | Problem | Proposed Fix | FPS Impact | UX Impact | RAM/CPU Impact | Risk | Measurement |
|----------|------|----------|---------|--------------|------------|-----------|----------------|------|-------------|
| 🔴 P0 | CPU/Battery | `CustomCursorRenderer.java` | 16ms `postDelayed` loop hogs CPU and wakes UI thread. | Remove `postDelayed(16)`. Sync animation frames with `Choreographer` or disable during gameplay. | 🟢 Yes (Improves pacing) | 🟡 Medium | 🔴 High (CPU) | Low | Profiler CPU trace |
| 🔴 P0 | UI/Scroll | `*Adapter.java` (Multiple) | Widespread use of `notifyDataSetChanged()`. | Implement `DiffUtil` or specific `notifyItem*()` calls. | ⚪ None | 🔴 High (Jank) | 🟡 Medium (CPU) | Medium | GPU Rendering Profile bars |
| 🟠 P1 | Memory/IO | `MinecraftAccount.java` | Synchronous `BitmapFactory.decodeFile` and `File.exists` on UI thread. | Move to background thread, use memory pool / Glide/Picasso for skins. | ⚪ None | 🟠 High (Stutters) | 🔴 High (RAM/GC) | Medium | Memory Profiler |
| 🟠 P1 | CPU/Pacing | `Tools.java` | `Controllable` mitigation thread runs infinite loop. | Use `FileObserver` instead of a busy `Thread.sleep` loop. | 🟢 Yes (Pacing) | ⚪ None | 🟠 High (CPU) | Low | CPU Thread trace |
| 🟡 P2 | CPU/GC | `ControlLayout.java` / `MainActivity.java` | Explicit `System.gc()` calls. | Remove explicit GC calls; let Dalvik/ART handle memory. | 🟢 Yes (Pacing) | ⚪ None | 🟡 Medium (GC pauses) | Low | GC pause times in Logcat |
| 🟡 P2 | Memory | `FirebaseSyncManager.java` | Caching full `JSONObject` and reparsing on main thread. | Parse JSON on background thread, store as Java POJOs. | ⚪ None | 🟡 Medium | 🟡 Medium (RAM) | Low | Memory Profiler |
| 🟢 P3 | GPU | `MinecraftGLSurface.java` | `TextureView` fallback causes extra GPU composition. | Ensure strict fallback to `SurfaceView` where possible. | 🟢 Yes (Raw FPS) | ⚪ None | 🟡 Medium (GPU) | Medium | Frame time / GPU Profiler |

## 19. Recommended Implementation Order
1. **Remove `System.gc()` calls** (Immediate, 1-line changes).
2. **Fix `CustomCursorRenderer` loop** (Move to `Choreographer` or pause when game runs).
3. **Replace `Controllable` Mitigation Thread** (Convert to `FileObserver`).
4. **Implement `DiffUtil` in Adapters** (Fixes UI scrolling).
5. **Refactor Bitmap Decoding** (Move out of `MinecraftAccount` sync paths).

## 20. Expected Overall Impact
By removing the constant CPU wakeups (16ms cursor, mitigation loop) and GC pressure (explicit GC, Bitmap churn), the JVM will have more uninterrupted CPU cycles. This will directly translate to **more stable frame pacing (fewer micro-stutters)** in Minecraft. The launcher UI will become vastly more fluid by eliminating main-thread I/O and `notifyDataSetChanged()`.

## 21. Measurement/Benchmark Plan
- **Frame Pacing:** Use Android GPU Inspector (AGI) and `systrace` / `perfetto` to track frame rendering times (should see fewer dropped frames).
- **CPU/Battery:** Use Android Studio Profiler (CPU) to confirm the `CSL-Worker` and UI threads are mostly sleeping during gameplay, not waking up every 16ms.
- **Memory:** Track Dalvik Heap to ensure GC pauses (Stop-The-World) drop below 5ms.

## 22. Risks and Things We Should NOT Change
- **DO NOT change:** The JNI bridging logic (`JREUtils.launchJavaVM`) unless strictly necessary, as it is highly fragile across different Android architectures.
- **DO NOT touch:** The native `FpsCounter` bridge. It correctly reads GL swap boundaries; only the Java-side UI updater should be tweaked if needed.
- **DO NOT remove:** The background thread limits (`PojavApplication.sExecutorService`). They are well-tuned for low-end devices.

## 23. Final Roadmap

**PHASE 1 (Critical & Low Risk):**
- Remove all `System.gc()` calls.
- Fix `CustomCursorRenderer` so it does not execute `postDelayed` during active gameplay or use `Choreographer`.

**PHASE 2 (High Impact & Medium Risk):**
- Replace the `Controllable` mitigation polling thread in `Tools.java` with a native Android `FileObserver`.
- Refactor `MinecraftAccount.java` skin loading to use an asynchronous image loading library (or strict background threads) to stop main-thread I/O blocks.

**PHASE 3 (UI Fluidity):**
- Systematically replace `notifyDataSetChanged()` with `DiffUtil` across `RTRecyclerViewAdapter`, `ProfileAdapter`, and modpack adapters.
- Move `FirebaseSyncManager` JSON parsing off the UI thread.

**PHASE 4 (Optional & Advanced):**
- Investigate pooling `MotionEvent` objects inside `MinecraftGLSurface` to reduce GC pressure on high-polling-rate mice/gamepads.
- Default to `SurfaceView` aggressively over `TextureView` to save GPU composition overhead on lower-end devices.
