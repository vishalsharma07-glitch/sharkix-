# Changelog

All notable changes to Eglantine are documented here. Each release's notes are taken from its
`## [X.Y.Z]` section (release candidates use the section of the version they are a candidate for).
New entries go under `## [Unreleased]`; `tools/ci/bump-version.sh` turns them into a version section.

## [Unreleased]

### 9070xt patch (frame-time spikes; RX 9070 XT + Ryzen 5 7600X, Ocarina of Time)

- Native draws no longer freeze the frame while their pipeline compiles: an urgent build starts
  at once on dedicated threads, draws wait at most a per-frame budget (3 ms) and are skipped for
  that frame otherwise. Draws that store to buffers/images or use transform feedback still wait.
  `EDEN_NX2_NATIVE_ASYNC=0`, `EDEN_NX2_NATIVE_ASYNC_BUDGET_US`, `EDEN_NX2_URGENT_PIPELINE_THREADS`.
- Unused pipelines are evicted after 1 h instead of 10 min (`EDEN_NX2_EVICT_PIPELINE_FRAMES`).
- The used-pipeline list is written on the serialization thread, not on the GPU thread mid-draw.
- Texture cache keeps up to 3/4 of the device budget on 12 GB+ cards (sharkiix 3.0 change).
- Windows MSVC job enabled in sharkiix-linux.yml; tuned dist/nx2_switches.txt.

## [0.1.0]

- Initial release candidate
