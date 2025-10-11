# Analysis of WaterThroughput Problems

The `EnableTestWaterThroughput` command is defined as a write-only service-level message. After enabling service access, issuing the write succeeds, which confirms that the earlier `ERR: element not found` was caused by access-level filtering.

Reading `EnableTestWaterThroughput` is expected to fail because it is not a readable message:

```sh
ebusctl read -f -c hmu EnableTestWaterThroughput
ERR: element not found
```

The remaining problem is with `TestWaterThroughput`:

```sh
ebusctl read -f -c hmu TestWaterThroughput
ERR: invalid position in decode
```

The generated CSV shows that `TestWaterThroughput` sends only the short test-menu read request:

```csv
r,,service,TestWaterThroughput,...,b514,052b,...,IGN:2,...,UIN
```

That means ebusd sends request id `052b` and expects a slave response long enough to decode `IGN:2` followed by a `UIN` value. The error `invalid position in decode` means the returned payload is shorter than expected, so decoding runs out of bytes before the `UIN` can be read.

The locally added `WaterThroughput` works because it is not equivalent on the wire. It sends the selector bytes directly as part of the read request:

```csv
r,,,WaterThroughput,...,b514,052b03ffff,...,IGN:2,...,UIN
```

So the working request id is `052b03ffff`, while `TestWaterThroughput` uses only `052b` after a separate enable write. On this HMU, the split sequence `write EnableTestWaterThroughput` followed by `read TestWaterThroughput` does not produce the same readable response as the single read request containing `03ffff`.

Practical conclusion: for this device, the single-read `WaterThroughput` definition with `@ext(0x2b, 0x03, 0xff, 0xff)` is the functioning definition. The generated `EnableTestWaterThroughput` plus `TestWaterThroughput` pair is currently not wire-equivalent for this use case.

## Context for a Future Codex Session

This file is local scratch/context, not official repo documentation. It exists to preserve the reasoning behind the local `WaterThroughput` addition in `src/vaillant/08.hmu.tsp`.

Relevant local source locations:

- `src/vaillant/08.hmu.tsp`: `EnableTestWaterThroughput` / `TestWaterThroughput` are in the PR-160-derived service-level VWZ test-menu block. They inherit `ws_1` / `rs`, where `ws_1` writes the fixed `03FFFF` payload and `rs` reads only the short `0x2b` selector response.
- `src/vaillant/08.hmu.tsp`: the locally added `WaterThroughput` is near the later `r_9` block and uses `@ext(0x2b, 0x03, 0xff, 0xff)`, which produces the working combined request.
- Local commit `61182a5e4d23b3d3641a692270082ff64a9292a0` added `SourceTempInput` and the local `WaterThroughput`. Its commit message records:
  - `SourceTempInput`: https://github.com/john30/ebusd/issues/335#issuecomment-2585881889
  - `WaterThroughput`: old CSV plus https://github.com/john30/ebusd-configuration/pull/160#issuecomment-2629582474

Upstream history and discussion trail:

- PR #160 is the main source for the VWZ/HMU test-menu definitions: https://github.com/john30/ebusd-configuration/pull/160
- PR #160 was later reworked/merged in commit `6577542f0188cbe081cf0b926f7541d9f9e63d1c`: https://github.com/john30/ebusd-configuration/commit/6577542f0188cbe081cf0b926f7541d9f9e63d1c
- In the reworked CSV from that commit, `WaterThroughput` no longer exists as a single `2B03FFFF` read. It is represented as:
  - `ws,,EnableTestWaterThroughput,Read T.016,...,2B`
  - `rs,,TestWaterThroughput,current heating water flow rate,...,2B,,,UIN,,l/h`
- PR #160 comment by Bernd noting that `WaterThroughput` disappeared after the rework and linking discussion #1476: https://github.com/john30/ebusd-configuration/pull/160#issuecomment-2629582474
- Reply from john30 saying it was moved/renamed to the generated `EnableTestWaterThroughput` / `TestWaterThroughput` definitions: https://github.com/john30/ebusd-configuration/pull/160#issuecomment-2630146159
- Follow-up from Bernd showing confusion about how to use the enable/read pair and initial `ERR: element not found` results: https://github.com/john30/ebusd-configuration/pull/160#issuecomment-2630330350
- Later john30 clarification that the enable message must be written, not read: https://github.com/john30/ebusd-configuration/pull/160#issuecomment-2799981051
- Later Bernd follow-up saying plain `write -c hmu EnableTestWaterThroughput` still returned `ERR: element not found` at that time, and cross-linking discussion #1503 / `BuildingCircuitFlow`: https://github.com/john30/ebusd-configuration/pull/160#issuecomment-3393512208
- Discussion #1503 records another user's `WaterThroughput: element not found` after switching configuration sources, and a claim that the renamed value is `hmu BuildingCircuitFlow`: https://github.com/john30/ebusd/discussions/1503

Important interpretation:

- `ERR: element not found` can mean different things depending on access level and message direction. `EnableTestWaterThroughput` is write-only and service-level, so reading it is expected to fail, and writing it without the needed access flags/config can also fail.
- The current unresolved technical point is not just access. After service access is enabled and the write succeeds, `TestWaterThroughput` still decodes with `ERR: invalid position in decode` on this device. That points to a short response for the split sequence, not simply a missing definition.
- The old/single-read behavior sends `B514 052B03FFFF` in one read request and works locally. The generated PR-160-style behavior sends a write for `B514 052B` with fixed payload `03FFFF`, then reads `B514 052B`; on this HMU, that split behavior has not produced a decodable water-throughput response.
- If revisiting this, compare raw ebusd logs for all three candidates: local `WaterThroughput`, generated `EnableTestWaterThroughput` + `TestWaterThroughput`, and any generated/current `BuildingCircuitFlow` definition. The decisive evidence is the actual sent MS command and returned payload length, not only the message name.
