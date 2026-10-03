<!-- Generative AI was used in the Creation/Modification of this file -->
# Software Hub

Index of notes in `04-software/`.

Curriculum spine and coverage checklist: [[_software-curriculum]]

## ros2/

- [[ros2-client-library-stack]] — rclpy/rclcpp/rcl/rmw layering, what rclpy.init()/Node()/spin() each actually do, web-server analogy
- [[ros2-executors-and-callbacks]] — spin is event-driven not periodic, single-threaded executor serializes callbacks, periodic-tick-thread vs spin-thread concurrency
- [[ros2-callback-groups]] — mutually exclusive vs reentrant callback groups, shared-mutable-state risk
- [[ros2-qos-policies]] — reliability (RELIABLE/BEST_EFFORT), durability (TRANSIENT_LOCAL/VOLATILE), history (KEEP_LAST/KEEP_ALL), compatibility rules
- [[ros2-dds-rmw-domain-matching]] — RMW implementation/domain ID matching, when it matters (runtime discovery, not builds)
- [[ros2-tf2-buffer-fundamentals]] — static vs dynamic TF, tf_buffer as time-indexed history, cache_time pruning by message stamp, live vs bag-replay buffer-filling
- [[ros2-time-domains]] — wall clock vs ROS clock vs message stamp, use_sim_time, /clock mechanics, live/sim/bag-replay clock-domain differences
- [[ros2-colcon-build-install-layout]] — colcon build vs install layout, PYTHONPATH/AMENT_PREFIX_PATH, symlink-install (pikaros2)
- [[ros2-bag-replay-tmux-launch-nuances]] — sim time/--clock propagation, ros2 launch vs ros2 run arg syntax, tmuxp multiline YAML trap
- [[ros2-bag-corruption-recovery]] — mcap recover + reindex for a bag with empty metadata.yaml / truncated MCAP footer

## python/

- [[python-venv]] — creating/activating venvs, mechanics, gotchas (incl. unverified ROS2 interaction)
- [[python-pythonpath]] — sys.path resolution order, PYTHONPATH mechanics, shadowing gotchas

## mlflow/

- [[mlflow-tracking-architecture]] — client-server model, backend store vs artifact store, experiment/run hierarchy, MLflow 3.5+ allowed-hosts/CORS
- [[mlflow-ops-deployment]] — Docker compose stack, reboot persistence, Postgres-vs-NFS durability, backup & server migration

## Other

- [[git-submodules]] — pinned-SHA model, update/push order, "not our ref" failure and diagnosis (pikaros2 `pika_warehouse_description`)
- [[reverse-proxy-networking-fundamentals]] — reverse proxies, HTTP default ports, why 443 forces TLS, diagnosing firewall vs VPN blocks
