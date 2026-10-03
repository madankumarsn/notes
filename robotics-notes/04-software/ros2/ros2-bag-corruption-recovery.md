---
topic: Recovering a corrupt ros2 bag (empty metadata.yaml / truncated MCAP footer)
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Recovering a corrupt ros2 bag

**One-line idea:**
A bag that was copied, killed, or interrupted before its final flush can end up with a 0-byte `metadata.yaml` and an `.mcap` file missing its footer; `ros2 bag play` then fails with an opaque YAML-parsing error that doesn't mention corruption at all — recoverable via `mcap recover` + `ros2 bag reindex`, provided the `.mcap` chunk data itself wasn't destroyed.

**Why it exists:**
The failure mode and its fix are non-obvious from the error message alone, and the root cause is almost always an operational mistake (copying too early, force-killing the recorder) rather than a bug in the recording pipeline — worth recognizing quickly rather than re-debugging from scratch each time.

**Math / mechanics:**
A rosbag2 MCAP recording is two things on disk that must both be intact: `metadata.yaml` (a small YAML summary of topics/duration/compression, generated at the end of recording) and one or more `.mcap` files (the actual serialized messages, closed out with a footer/index on graceful shutdown). `mcap recover` rebuilds a valid footer/index directly from the `.mcap` file's own chunk data; `ros2 bag reindex` then regenerates `metadata.yaml` by reading that now-valid `.mcap` file — the two tools fix different halves of the same underlying problem, in that order.

**Code:**
```text
# Symptom
$ ros2 bag play ~/bags/my_bag
Exception on parsing info file: invalid node; first invalid key: "version"
```

```bash
# Diagnose
ls -la ~/bags/my_bag/
wc -c ~/bags/my_bag/metadata.yaml      # 0 bytes = never got written / got truncated
ros2 bag info ~/bags/my_bag            # fails the same way until metadata exists

# Recover: rebuild a valid MCAP footer, then regenerate metadata.yaml from it
cd ~/bags/my_bag
mcap recover my_bag_0.mcap -o my_bag_0_recovered.mcap
mv my_bag_0.mcap my_bag_0.mcap.broken
mv my_bag_0_recovered.mcap my_bag_0.mcap
rm metadata.yaml
ros2 bag reindex . -s mcap
ros2 bag info .                        # should now report topics/duration correctly

ros2 bag play ~/bags/my_bag --clock
```

**Gotchas:**
- `metadata.yaml` needs a top-level `rosbag2_bagfile_information:` block with a `version:` key to be parseable at all — a 0-byte or truncated file produces the generic "invalid node; first invalid key: 'version'" error, which gives no hint that the real problem is "there is no metadata here," not a version-mismatch issue.
- Root causes that produce this: copying the bag directory before the recorder finished its final flush (e.g. mid-`scp`/`rsync`), killing the recording process (`kill -9`, closing the terminal, a `sleep`-based script ending early), or a disk/IO issue at the moment of the final write — even if the recorder printed a clean "Recording stopped," the files on disk can still be incomplete if something external interrupted or copied them concurrently.
- Prevention: wait for the recorder's own "Recording stopped" message *and* run `ros2 bag info <dir>` successfully before copying the directory anywhere; stop recording with Ctrl+C (graceful shutdown), not `kill -9`; only copy once `metadata.yaml` is confirmed non-zero size.
- `mcap recover` rebuilds a usable footer/index from the raw chunk data inside the `.mcap` file — it can't help if the underlying chunk data itself was corrupted or truncated mid-write, only if the file is otherwise intact but missing its closing index/footer.
- `ros2 bag reindex` regenerates `metadata.yaml` *from* the `.mcap` file's own contents — run it only after the `.mcap` itself is valid (post-`mcap recover`), not as a first attempt on a still-broken file.

**Related:** [[ros2-bag-replay-tmux-launch-nuances]], [[product-mapping-tf-publishing]]

**Still unclear:**
Whether this specific recovery procedure actually succeeded on the bag that triggered this note (`mcap recover` + `reindex`) wasn't confirmed in the source material — only the recommended recovery steps were captured, not a verified "it worked" outcome. Verify the procedure end-to-end on a real corrupt bag before relying on it under time pressure.
