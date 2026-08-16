# lean-entity-packet

The Lean 4 + Plausible source of truth for the fabric's 100-byte entity packet (`XRGridEntityPacket`). The wire is fully integral — no floats — so it models exactly in Lean, and Plausible roundtrip properties find codec gaps in seconds rather than an engine rebuild.

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

Position int64 μm is the integral twin of the `precision=double` large-world coordinate and matches the Lean-proved predictive BVH's int64-μm AABB space, which now lives in `lean-spatial-oracle` and is kept in sync. Velocity shares that BVH's `V_MAX` scale.

## The C codec

`xr_grid_entity_packet.h` is emitted from `EntityPacket/Codec.lean` and committed at the root, so a repository that vendors this one gets the codec without running Lean. It is plain C needing nothing beyond `stdint.h`: `xr_grid_entity_packet_t`, an encode, a decode, and the offsets as macros.

It exists because `transport-fanout/src/fanout.h:23` has included `gen/xr_grid_entity_packet.h` since it was written and nobody had emitted it. Do not edit the header — edit `Codec.lean`, where the offsets are named once and read by the encoder, the decoder, and the emitter, then re-run `packet_emit`. Copying a generated header by hand puts one decision in two places.

## Verify

```sh
lake exe packet_demo   # Plausible roundtrip + size, 50000-vector sweep
lake exe packet_emit   # writes packet_golden.csv and xr_grid_entity_packet.h
diff packet_golden.csv build/packet_golden.csv              # committed copies are current
diff xr_grid_entity_packet.h build/xr_grid_entity_packet.h
cc -std=c11 -I build test/golden.c -o build/golden && ./build/golden build/packet_golden.csv
godot --headless --script packet_diff.gd   # the engine's C++ decode must match
```

Both differentials print `PACKET DIFFERENTIAL PASS`. Last run: Plausible clean, 50000/50000 roundtrip, C++ decode matching the spec on 64 golden vectors, emitted C encoder matching all 64.
