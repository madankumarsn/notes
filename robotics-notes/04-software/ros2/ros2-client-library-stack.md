---
topic: ROS2 client library stack (rclpy/rclcpp/rcl/rmw) and the node lifecycle
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 client library stack and node lifecycle

**One-line idea:**
`rclpy` (Python) and `rclcpp` (C++) are language-specific wrappers around the same language-neutral `rcl` core, which itself talks to the network through `rmw`/DDS; `rclpy.init()` only starts this library in your process — it does not create a node, and nothing receives ROS messages until something calls `spin()`.

**Why it exists:**
A reusable mental model for "what is actually running" in any ROS2 Python program, and a way to explain ROS2's architecture to someone already familiar with web-server concepts (runtime / app / messaging / event loop).

**Math / mechanics:**

- **The layer stack:**
```
Your code (Python)
      │
      ▼
   rclpy          ← Python client library: Node, Publisher, spin(), etc.
      │
      ▼
    rcl            ← language-neutral core: node lifecycle, graph, parameters, time
      │
      ▼
    rmw             ← middleware abstraction (wraps DDS)
      │
      ▼
    DDS             ← actual network transport (Fast-DDS, Cyclone, etc.)
```
  `rclcpp` sits at the same level as `rclpy`, just for C++. Both are thin, language-specific wrappers around the *same* `rcl` core — which is why a Python node and a C++ node can freely talk to each other: they both go through `rcl` down to the same middleware.
- **`rclpy.init()` just starts the library in this process** — it does not create a node and has no identity on the ROS graph by itself. A node is a separate object created on top of it (e.g. `Node("name")`, or indirectly by a framework like `py_trees_ros.trees.BehaviourTree.setup(node_name=...)`).
- **`rclpy.spin(node)` is the event loop** — it blocks and processes that node's callbacks (subscriptions, timers, services, actions) until shutdown. Nothing on that node receives messages unless something is spinning it.
- **The web-server analogy**, useful for explaining this to someone from a web background:

| Web concept | Web example | ROS 2 equivalent |
|---|---|---|
| Runtime | Start uvicorn/nginx | `rclpy.init()` — start ROS in this process |
| App | Flask app with routes | `Node("my_node")` — your node's identity in the graph |
| Messaging | HTTP GET/POST | Topics (pub/sub), services (request/reply), actions (long tasks) |
| Event loop | Server waits for requests | `rclpy.spin(node)` — wait for messages, timers, calls |

  The analogy isn't exact (ROS is a distributed graph of many nodes, not one server with many routes), but the *roles* line up: initialize runtime → define your component → define communication → run forever handling events.

**Code:**
```python
import rclpy
from rclpy.node import Node

rclpy.init(args=None)                  # 1. start the client library (no node yet)
node = Node("my_node")                 # 2. create a node — this is your identity on the graph
pub = node.create_publisher(...)       # 3. messaging: send
sub = node.create_subscription(...)    # 3. messaging: receive
rclpy.spin(node)                       # 4. event loop: process callbacks until shutdown
```

**Gotchas:**
- Seeing `rclpy.init()` in a file does **not** mean a ROS node exists yet — always find the actual `Node(...)` construction (or the framework call that creates one, e.g. `tree.setup(node_name=...)`) before assuming anything is "on the graph."
- A node that's constructed but never spun (and has no other executor driving it) will not process any incoming callbacks — it exists on the graph (if discovery has happened) but is effectively inert for anything event-driven.

**Related:** [[ros2-executors-and-callbacks]], [[ros2-qos-policies]]

**Still unclear:**
This is general ROS2 architecture, consistent with how the client libraries are documented; not tied to project-specific code beyond the illustrative example (`bt_local_sequencer.py`'s `rclpy.init()` → `tree.setup(node_name="local_sequencer")` → `rclpy.spin(tree.node)` sequence).
