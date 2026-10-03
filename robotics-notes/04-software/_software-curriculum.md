<!-- Generative AI was used in the Creation/Modification of this file -->
# Software Fundamentals Curriculum

Spine for `04-software/`, Harvard CS207 2019 (Python, execution model, good practice, data management, team-scale software). Goal: conceptual clarity on things I already use, not new tooling.

**How to use:** each topic gets one note written as *what problem it solves → the mental model → how it works underneath → where I've hit it*. `[ ]` = no note yet, `[x]` = note exists. Existing practical notes are slotted in where they illustrate a concept.

## 1. Working environment (Unix, build, tools)
- [ ] Shell, processes, env vars, PATH, file permissions
- [ ] Compilers and the build pipeline: preprocess → compile → link, static vs shared libraries
- [ ] Makefiles / CMake: dependency graph, incremental builds
- [x] [[ros2-colcon-build-install-layout]] (applied: build vs install space)
- [ ] Debuggers (gdb, pdb), sanitizers
- [ ] Profilers: CPU, memory, flame graphs

## 2. Version control
- [ ] Git object model: blobs, trees, commits, refs
- [ ] Branching, merging vs rebasing, resolving conflicts
- [x] [[git-submodules]] (applied: pinned-SHA model)
- [ ] Collaboration workflow: PRs, review, tags/releases

## 3. Language execution model
- [ ] Python: names/objects/references, mutability, scoping, GIL, import system, how the interpreter runs code
- [x] [[python-pythonpath]], [[python-venv]] (applied: import resolution, environments)
- [ ] C++: value vs pointer vs reference, stack vs heap, lifetime/RAII, const, move semantics
- [ ] C++: templates, compile-time vs run-time polymorphism
- [ ] Memory management: ownership, leaks, dangling references

## 4. Program structure and design
- [ ] Modularity, abstraction, encapsulation
- [ ] Object-oriented programming: classes, inheritance vs composition, interfaces
- [ ] Dependencies, coupling, factoring and reuse
- [ ] Using external libraries; API design
- [x] [[ros2-client-library-stack]] (applied: layered library design)

## 5. Correctness and quality
- [ ] Testing: unit, integration, fixtures, mocking, coverage
- [ ] Debugging method: reproduce, bisect, hypothesize
- [ ] Documentation: docstrings, READMEs, design docs
- [ ] Performance debugging: measure before optimizing

## 6. Data structures and algorithms
- [ ] Asymptotic analysis, Big-O
- [ ] Arrays, lists, hash maps, trees, heaps, graphs; cache behavior
- [ ] Numerical patterns: dense/sparse linear algebra, grids, FFT (link to `01-math`)

## 7. Data management
- [ ] Serialization formats (JSON, protobuf, MCAP, HDF5)
- [ ] Databases: relational model, SQL, indexes, transactions
- [x] [[mlflow-tracking-architecture]] (applied: backend store vs artifact store)

## 8. Concurrency
- [ ] Threads vs processes, locks, race conditions, deadlock
- [ ] Async / event loops
- [x] [[ros2-executors-and-callbacks]], [[ros2-callback-groups]] (applied: executors, shared state)

## 9. Containers and deployment
- [ ] What a container actually is: namespaces, cgroups, images and layers
- [ ] Docker: Dockerfile, volumes, networking, compose
- [ ] Kubernetes: pods, deployments, services, scheduling
- [x] [[mlflow-ops-deployment]] (applied: compose stack, persistence)

## 10. Networking
- [ ] OSI/TCP-IP layers, IP, ports, DNS
- [ ] TCP vs UDP, sockets
- [ ] HTTP, TLS, proxies
- [x] [[reverse-proxy-networking-fundamentals]]
- [x] [[ros2-dds-rmw-domain-matching]] (applied: discovery over multicast)

## 11. Distributed systems
- [ ] Clocks, ordering, and time
- [x] [[ros2-time-domains]] (applied: clock domains)
- [ ] Messaging patterns: pub/sub, request/response, queues
- [x] [[ros2-qos-policies]] (applied: delivery guarantees)
- [ ] Failure, consistency, replication, CAP

## 12. Team-scale software
- [ ] Project structure, packaging, CI
- [ ] Reviewing others' code, evaluating alternatives, writing design notes
