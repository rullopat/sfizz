# MPE fork — `rullopat/sfizz`

This fork adds full **MIDI Polyphonic Expression (MPE)** support to sfizz: per-channel pitch bend, per-channel CC modulation, per-channel aftertouch, channel-aware voice stealing, plus the public API needed to drive it from a host.

It is the runtime engine for a branded rompler product line targeted at expressive MPE controllers (Roli Seaboard, LinnStrument, and specifically the Expressive E Osmose). The first commercial product is a Moog Messenger clone for Osmose performance — without per-finger pitch / CC / aftertouch, a polyphonic chord on Osmose collapses to a single shared bend / shared filter, which is not a usable MPE instrument.

The fork lives at <https://github.com/rullopat/sfizz>. The branch this submodule is pinned to is `mpe-shipping-on-<base-sha>`; an upstream-targetable copy lives on `mpe`.

---

## Why a fork (not a wrapper, not stock sfizz)

Stock sfizz 1.2.3 has no MPE support: pitch bend lives as a single global `MidiState::pitchEvents` vector, all CC events are applied across every voice regardless of source channel, and `Voice` has no per-note pitch state. Per-note pressure can be faked from outside via `polyAftertouch(note, value)`, but there is no equivalent path for per-note pitch or per-note CC — the engine itself collapses them globally before any voice sees them. Wrapping or post-processing won't fix this; the change has to happen *inside* sfizz.

A 30-minute spike on the upstream tracker confirmed:

- **Open MPE feature request: [sfztools/sfizz#1313](https://github.com/sfztools/sfizz/issues/1313)** (April 2025), describing exactly this problem. **Zero maintainer comments. Zero PRs. Sat untouched for over a year.**
- **No in-flight upstream MPE work.** No design decisions made; we are free to design as we see fit.
- Adjacent precedent ([#1137 — MTS-ESP](https://github.com/sfztools/sfizz/issues/1137)) went a similar way: feature request, brief engagement on licensing concerns, no resolution.
- **Project PR throughput is slow.** Even a well-designed PR is likely to sit unreviewed for months. **We plan to ship from fork indefinitely**; treat upstream merge as upside, not roadmap.

sfizz is BSD-2-Clause, so private forking is permitted with the only obligation being preserving the copyright notice (already in our credits surface).

---

## Branches on the fork

- `mpe-shipping-on-<base-sha>` — primary working branch. The `external/sfizz` submodule in the consuming repo is pinned to a specific commit on this branch. Initially `mpe-shipping-on-f5c6e29f`, where `f5c6e29f` is the upstream commit we vendor (today: 39 commits past tag 1.2.3 on `develop`). When upstream releases a new version we want to move to, branch `mpe-shipping-on-<new-base-sha>` from the new base and rebase the engine commits onto it.
- `mpe` — upstream-PR-target branch. Constructed by cherry-picking the engine commits from `mpe-shipping-on-<base-sha>` onto a fresh branch off `upstream/develop`. Holds only the upstream-clean commits — no product-specific glue, no `SAMPLEMACHINE_*` defines. Currently the two branches contain the same commit set; they will diverge as soon as we add product-specific patches that don't belong upstream.

The submodule `origin` remote points at `rullopat/sfizz`. The original `sftools/sfizz` is preserved as `upstream` for fetching new tags / develop tip.

---

## Commit set

Six engine commits, each independently buildable. Listed oldest → newest:

1. **MidiState: introduce per-channel `ChannelState` struct** (`36a6e09`)
   Refactors the global pitch/CC/aftertouch event vectors into a private nested `ChannelState` struct, owned by `MidiState` as a 16-element array indexed by MIDI channel (0..15). All public API still resolves to `channelStates[masterChannel]` (master = 0); behavior byte-for-byte identical. Pure structural refactor.

2. **MidiState/Voice: plumb per-voice channel for modulation reads** (`a8b2874`)
   Adds channel-aware overloads to `MidiState`'s pitch/CC/aftertouch read accessors. `Voice` gains a private `triggerChannel_` field set in `startVoice` (currently always 0 / master). Per-voice modulation reads (pitch bend smoother seed, pitch envelope source, crossfade-CC values + events) route through `triggerChannel_`. Behavior byte-for-byte unchanged.

3. **Synth: add MPE-aware public API and per-channel dispatch** (`3db6b80`)
   Adds `noteOnMPE` / `hdNoteOnMPE`, `noteOffMPE`, `ccMPE` / `hdccMPE`, `pitchWheelMPE` / `hdPitchWheelMPE`, `channelAftertouchMPE` / `hdChannelAftertouchMPE`, `polyAftertouchMPE` / `hdPolyAftertouchMPE`, plus `setMPEEnabled` / `getMPEEnabled` / `setMPEPitchBendRange` / `getMPEMasterPitchBendRange` / `getMPEPerNotePitchBendRange`. `TriggerEvent` gains an `int channel = 0` field so the spawned voice's `triggerChannel_` is sourced from dispatch. Existing single-channel methods forward to the new `*MPE` variants with channel = 0.

4. **VoiceStealing: bias victim selection to a preferred MIDI channel** (`e4812d8`)
   Adds an optional `preferredChannel = -1` parameter to `VoiceStealer::checkRegionPolyphony` / `checkPolyphony`. When non-negative, the stealer prefers a same-channel victim before falling back to cross-channel. `Synth::Impl::startVoice` passes `mpeEnabled_ ? triggerEvent.channel : -1`. With MPE off, `-1` reproduces the prior single-candidate path.

5. **MidiState: lazy member-channel events + MPE regression test suite** (`5ad530a`)
   Member-channel event vectors are populated lazily; channel-aware getters return the static `nullEvent` sentinel for channels that have never been written. `flushEvents` now iterates all 16 channels (cheap empty-skip). Adds `tests/MPET.cpp` with 15 regression test cases / 68 assertions covering per-channel state isolation, channel-aware Synth API routing, voice stealing under MPE, and configuration round-trips.

6. **Sfizz: expose channel-aware MPE methods on the public C++ wrapper** (`cd7d7df`)
   Forwards the `*MPE` input methods and the `setMPEEnabled` / `setMPEPitchBendRange` configuration through `sfz::Sfizz` (the public C++ wrapper) to the underlying `Synth` implementation. Hosts that already use `Sfizz` directly can now drive MPE input without reaching into engine internals. C API in `sfizz.h` is intentionally not extended in this commit.

Total diff: ~620 insertions across `MidiState`, `Voice`, `Synth`, `VoiceStealing`, the public wrapper, and a new test file.

---

## Test results

Full sfizz test suite passes against the fork:

```
$ ./library/bin/sfizz_tests
All tests passed (52269 assertions in 492 test cases)
```

The 15 new MPE-specific cases (`[MPE] *`) cover:

- per-channel pitch bend / CC / channel aftertouch / poly aftertouch state isolation
- out-of-range channel safety (writes are no-ops, reads return 0 / nullEvent)
- single-arg overloads forwarding to master channel (backward compatibility)
- `Synth::*MPE` public API routing events to the correct channel slot
- `noteOnMPE` tagging spawned voices with the originating channel
- `setMPEEnabled` / `getMPEEnabled` and `setMPEPitchBendRange` round-trips
- voice stealing preferring same-channel candidates when MPE is enabled

---

## Rebase procedure (when moving to a newer upstream tag)

1. **Fetch upstream**: `git -C external/sfizz fetch upstream`
2. **Create the new shipping branch**: `git -C external/sfizz checkout -b mpe-shipping-on-<new-base-sha> <new-base-sha>`
3. **Cherry-pick the six engine commits in order** from `mpe-shipping-on-<old-base-sha>` (Sha1 list above). Resolve any conflicts — most likely in `MidiState`, `Voice`, or `Synth` if upstream refactored those files. Keep commit messages and content as upstream-clean as before.
4. **Run sfizz tests**: from `external/sfizz`, configure with `-DSFIZZ_TESTS=ON` in a side build dir, build the `sfizz_tests` target, run from `external/sfizz` (the binary climbs up looking for `tests/TestFiles`). All 492 cases / 52269 assertions should pass; the 15 MPE-specific ones in particular validate the per-channel routing.
5. **Push the new shipping branch** to `origin` (the fork): `git -C external/sfizz push origin mpe-shipping-on-<new-base-sha>`.
6. **Update the consuming repo's `.gitmodules`** to declare the new branch and bump the submodule pin to the tip of the new branch. Verify the consuming project still builds and tests cleanly.
7. **Reconstruct the `mpe` PR branch** by force-pushing the new commit set onto `origin/mpe` (delete the previous `mpe` branch and recreate from the new shipping branch). The upstream PR — if it's still open — will pick up the new commits automatically.

If upstream refactors `MidiState` or `Voice` significantly, expect 1–3 days per upstream version of merge work. The contribution-friendly structure (separable commits, no `SAMPLEMACHINE_*` defines on engine commits, C++11 floor, code-style match) keeps the rebase mechanical rather than archaeological.

---

## Constructing the upstream PR branch

The `mpe` branch is what an upstream PR would be opened from. To (re)construct it from the current shipping branch:

```sh
git -C external/sfizz fetch upstream
git -C external/sfizz checkout -b mpe upstream/develop
# Cherry-pick the engine commits in order. Each is independently
# buildable, so you can pause and run sfizz tests after each pick.
git -C external/sfizz cherry-pick 36a6e09a a8b28743 3db6b80a e4812d8b 5ad530a9 cd7d7df1
git -C external/sfizz push origin mpe
```

Then open a draft PR on `sftools/sfizz` against `develop` from `rullopat:mpe` and post the design summary on issue #1313, tagging `@paulfd`, `@jpcima`, and `@jamshark70` (the issue author, who explicitly offered to contribute).

---

## License

This fork is BSD-2-Clause, same as upstream sfizz. The original copyright notice is preserved in every modified file.
