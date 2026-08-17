# contract-entity-packet

The Lean 4 + Plausible source of truth for the fabric's 100-byte entity packet (`XRGridEntityPacket`). The wire is fully integral — no floats — so it models exactly in Lean, and Plausible roundtrip properties find codec gaps in seconds instead of an engine rebuild.

## Layout (100 bytes, all integral)

| offset | field | encoding |
| --- | --- | --- |
| 0 | global_id | u32 |
| 4 | position x/y/z | **int64 absolute micrometers** (no origin shift) |
| 28 | velocity x/y/z | i16, scaled to ±`PBVH_V_MAX_PHYSICAL_DEFAULT` (500000 μm/tick) |
| 40 | hlc | u32 (frame<<8 \| counter) |
| 44 | class\|owner | u32 |
| 48 | sub_index | u32 |
| 52 | rotation | i16 swing-twist ×3 |
| 58 | payload | 42 bytes userdata (cmd/action/state/name) |

Position int64 μm is the integral twin of the `precision=double` large-world coordinate, and matches the Lean-proved predictive BVH's int64-μm AABB space (`interactor-spatial-oracle`, kept in sync). Velocity shares the BVH's `V_MAX` scale.

## The emitted codecs

`xr_grid_entity_packet.h` and `xr_grid_entity_packet.py` are emitted from `EntityPacket/Codec.lean` and committed at the root, so a repository that vendors this one gets a codec without running Lean. The C is plain, with no dependency beyond `stdint.h`. The Python writes a field at a time at a named offset rather than through one `struct` format string, which would state the layout a second time as positions that no longer name the offsets they stand for.

It exists because `transport-fanout/src/fanout.cpp` has included it since it was written and nobody had emitted it — that repository's `CMakeLists.txt` names the missing header as the reason it has never built, and says why copying one in by hand would be wrong: "copying the generated headers here would put one decision in two places."

The Python exists because RFD 0123 puts a second implementation of the WebTransport contract on `pywebtransport`, and a second implementation that retyped this layout would test whether two people can copy a table.

Do not edit either. Edit `Codec.lean`, where the offsets are named once and read by the encoder, the decoder and both emitters, then run `packet_emit` again.

## Verify

`VERIFY.md` has the commands, the two differentials that hold the C and the engine to this spec, and what the last run reported.
