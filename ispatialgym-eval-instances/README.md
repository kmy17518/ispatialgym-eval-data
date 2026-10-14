# iSpatialGym eval instances

83 eval instances, one per subtask, laid out like BEHAVIOR-1K's `2026-challenge-task-instances` and kept next to it
in OmniGibson's data folder, with their demos in the sibling dataset `ispatialgym-eval-demos`. Written by
`scripts/eval_instances/export_counted.py` from the eval runs that `scripts/eval_instances/release_runs.tsv` lists (the
counted instances of 2026-10-14 ~10:37, frozen), each with its goal image.

## Hierarchy

- **task family**: alignment, camera_relocalization, configuration_matching, disembedding, inter_object_relation_satisfaction,
  mental_rotation, object_scaling.
- **task**: a family's task, e.g. `alignment_axial`, `alignment_vertical`, `camera_relocalization_obstructed`,
  `configuration_matching_attachment`.
- **subtask**: a task with its objects, the support left out, e.g. `alignment_axial_board_game`; each subtask has one scene
  and one template, and its instances differ only in their initial state. `isg_customs.json` is keyed by subtask.
- **variant**: the sampler's grounded setup of a subtask, its objects and support, e.g.
  `alignment-axial-board_game-breakfast_table` (the BDDL problem's `(:variant ...)`, the instance file's `task_variant`).
  A subtask may have several; here each has one, named by `variant` in the template metadata and its `eval` block.

| task family | task | subtasks |
|---|---|---|
| alignment | alignment_axial | 4 |
| alignment | alignment_vertical | 5 |
| camera_relocalization | camera_relocalization_obstructed | 9 |
| camera_relocalization | camera_relocalization_standard | 1 |
| configuration_matching | configuration_matching_articulation_close_large_scale | 2 |
| configuration_matching | configuration_matching_articulation_close_small_scale | 2 |
| configuration_matching | configuration_matching_articulation_open_large_scale | 2 |
| configuration_matching | configuration_matching_articulation_open_small_scale | 2 |
| configuration_matching | configuration_matching_attachment | 4 |
| configuration_matching | configuration_matching_block_design | 1 |
| disembedding | disembedding | 9 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_inside_large_scale | 1 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_inside_small_scale | 4 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_nextto_large_scale | 1 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_nextto_small_scale | 4 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_ontop_large_scale | 1 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_ontop_small_scale | 3 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_touching_large_scale | 1 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_touching_small_scale | 2 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_under_large_scale | 2 |
| inter_object_relation_satisfaction | inter_object_relation_satisfaction_under_small_scale | 4 |
| mental_rotation | mental_rotation | 10 |
| object_scaling | object_scaling | 9 |

## Layout

```text
metadata/instances.jsonl       one line per instance: task_family, task, subtask, instance (index within the subtask),
                               task_instance (index within the task), scene, problem, length, files (relative to this
                               folder), source (the eval run it came from)
metadata/episodes.jsonl        one line per instance's demo: episode_index, the names, length (the demo's recorded steps,
                               the actions a replay steps), action_steps + settle_steps (the oracle's program and its
                               settle window, which make up length), demo (its file in ../ispatialgym-eval-demos), demo_sha256
metadata/tasks.yaml            task family -> task -> [subtask, ...]
metadata/available_tasks.yaml  subtask -> {0: scene_model, robot_start_position, robot_start_orientation, task, task_family}
metadata/task.jsonl            one line per subtask: task_index, task_name (= subtask), task, task_family, num_instances, length
scenes/<scene>/json/<scene>_task_<subtask>_0_0_template.json          the scene with the task objects
scenes/<scene>/json/<scene>_task_<subtask>_0_0_template-meta.json     the sampler's template facts (+ "eval": the names)
scenes/<scene>/json/<scene>_task_<subtask>_instances/
    <scene>_task_<subtask>_0_<i>_template-tro_state.json   instance i
    <scene>_task_<subtask>_0_<i>_template.oracle.json      its oracle sidecar (the demo program)
    <scene>_task_<subtask>_0_<i>_template-goal_rgb.png     its goal image
../ispatialgym-eval-demos/scenes/<scene>/json/<scene>_task_<subtask>_instances/
    <scene>_task_<subtask>_0_<i>_template-demo.hdf5        instance i's demo (the actions a replay steps)
```

As in BEHAVIOR's layout the first number after the subtask is the template (always 0: one template per subtask) and the
second the instance. The BDDL problems are not here, as BEHAVIOR's are not in its task instances: an instance names its
problem (`problem` = `<activity>-<id>` in the index and the template metadata, `activity_definition_id` in the instance
file), loaded from the iSpatialGym repository's `ispatialgym/assets/bddl/activity_definitions` (numbered from 0 per
activity). The files inside keep the sampler's own fields.

## Loading

```bash
# a task's instances are its subtasks' instances in the index order; a subtask's are its own
python -m ispatialgym.oracle.run_isg --task alignment_axial --instance-indices 0 2 --mode replay          # the instances' demos
python -m ispatialgym.oracle.run_isg --task alignment_axial_board_game --instance-indices 0 --mode policy --policy ...
python -m ispatialgym.eval.instances list --level task          # names and instance counts
python -m ispatialgym.eval.instances show alignment_axial 0 1   # what an index resolves to (files, goal image, demo)
```

`ISPATIALGYM_INSTANCES_PATH` (or `--instances-root`) points at this folder, `ISPATIALGYM_DEMOS_PATH` at the demos; the
defaults are `$OMNIGIBSON_DATA_PATH/ispatialgym-eval-instances` and its sibling `ispatialgym-eval-demos`. With `--task`,
run_isg gives an instance `max(300, 2 x its demo's length)` steps (`--max-steps`) and 1 s per step of wall clock
(`--wallclock`), a `pose_settled` / settle-eval window (`--settle-steps`) of 45 steps (oracle, policy) or all of its demo's
settle rows (replay: `episodes.jsonl` `settle_steps`), and replays its demo unless `--replay` names another.
run_isg never writes into this folder: runs go to `--out` (default `./runs`) and the goal / sidecar write-backs of oracle
runs are skipped for dataset instances.

## Goal images

The goal image is the head camera's last frame (`final_rgb.png`, 1080 x 1080) of a successful replay of the instance's demo:
the eval run's replay re-run with the current code, else its original replay (`python -m ispatialgym.eval.instances
set-goal`). A replay renders the whole episode; a frame rendered straight at the demo's final state comes out about 1 %
darker (3-4 % in shadows), because the renderer's bounced light depends on what it rendered before. Each instance's
`files.goal_rgb` (here) names it, and its instance file registers it (`ispatialgym.goal_observation`, as `run_isg
--goal-image` writes it: `rgb_path`, sha256, size, the head camera key and intrinsics, and the camera's world pose: the
demo's last recorded head camera pose, which the replay holds through its settle window), which is where the policy
harness reads it. The sidecar's `tro_sha256` follows the instance file.

Another image size: replay the instance at that size (`--record video`: no demo.hdf5 / LeRobot copy of the replay), then
make that run's final frame the goal image (one goal image per instance; a re-export keeps it). The replay keeps going
through the demo's own contacts with the objects: run_isg only logs the QA metrics of a replay (`--qa`), it never stops
at them.

```bash
sbatch scripts/eval_instances/job_run_isg.sh --task alignment_axial --instance-indices 0 --mode replay \
    --image-size 720 --record video --out <dir>
python -m ispatialgym.eval.instances set-goal alignment_axial 0 --run <dir>
python -m ispatialgym.eval.instances set-goal alignment_axial 0          # back to the eval run's replay (1080)
```
