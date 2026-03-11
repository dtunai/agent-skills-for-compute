# SimReady Isaac Sim Integration Reference

Sources:
- [Isaac Sim Assets](https://docs.isaacsim.omniverse.nvidia.com/latest/assets/usd_assets_overview.html)
- [Third-Party SimReady Assets](https://docs.isaacsim.omniverse.nvidia.com/latest/assets/usd_assets_third_party.html)

## Overview

Isaac Sim is NVIDIA's robotics simulation platform built on Omniverse. SimReady assets integrate seamlessly for robot training, synthetic data generation, and validation.

## Available SimReady Assets

### Built-in Assets

**1000+ SimReady assets including:**
- Conveyors
- Boxes (various sizes)
- Pallets (wood, plastic, metal)
- Shelving units
- Racks
- Crates
- Traffic cones
- Barrels

**Location:**
```python
# Isaac Sim asset library
/Isaac/Environments/
/Isaac/Props/
/Isaac/Robots/
```

## Loading SimReady Assets

### Using Python API

```python
from omni.isaac.core import World
from omni.isaac.core.utils.stage import add_reference_to_stage
from omni.isaac.core.utils.prims import create_prim

# Initialize world
world = World()

# Method 1: Add reference
add_reference_to_stage(
    usd_path="/Isaac/Props/Pallets/pallet.usd",
    prim_path="/World/Pallet_01"
)

# Method 2: Create prim with reference
create_prim(
    prim_path="/World/Box",
    usd_path="/Isaac/Props/Shapes/cube.usd",
    position=[1, 0, 0.5],
    scale=[0.5, 0.5, 0.5]
)

# Reset world to load assets
world.reset()
```

### Programmatic Placement

```python
import numpy as np

# Place multiple assets
for i in range(10):
    x = np.random.uniform(-5, 5)
    y = np.random.uniform(-5, 5)
    
    add_reference_to_stage(
        usd_path="/Isaac/Props/Pallets/pallet.usd",
        prim_path=f"/World/Pallet_{i}",
        translation=np.array([x, y, 0])
    )
```

## Robotics Workflows

### Manipulation Testing

```python
from omni.isaac.manipulators import SingleManipulator
from omni.isaac.core.objects import DynamicCuboid

# Add robot
robot = SingleManipulator(
    prim_path="/World/Robot",
    name="manipulator"
)
world.scene.add(robot)

# Add SimReady object to manipulate
target = DynamicCuboid(
    prim_path="/World/Target",
    usd_path="/Isaac/Props/Pallets/pallet.usd",
    position=[0.5, 0, 0.3],
    size=0.1
)
world.scene.add(target)

# Grasp planning
robot.gripper.apply_action(1.0)  # Close gripper
```

### Navigation Testing

```python
from omni.isaac.wheeled_robots.robots import WheeledRobot
from omni.isaac.core.utils.stage import get_current_stage

# Add mobile robot
robot = WheeledRobot(
    prim_path="/World/Robot",
    name="mobile_robot",
    wheel_dof_names=["left_wheel", "right_wheel"]
)

# Place obstacle (SimReady asset)
add_reference_to_stage(
    usd_path="/Isaac/Props/TrafficCone/traffic_cone.usd",
    prim_path="/World/Obstacle",
    translation=[2, 0, 0]
)

# Navigation controller
robot.apply_wheel_actions([0.5, 0.5])  # Forward
```

## Synthetic Data Generation

### Setup Replicator

```python
import omni.replicator.core as rep

# Create camera
camera = rep.create.camera(
    position=(5, 5, 5),
    look_at=(0, 0, 0)
)

# Create render product
rp = rep.create.render_product(camera, (1024, 1024))

# Attach annotators
rgb = rep.AnnotatorRegistry.get_annotator("rgb")
rgb.attach(rp)

seg = rep.AnnotatorRegistry.get_annotator("semantic_segmentation")
seg.attach(rp)

bbox = rep.AnnotatorRegistry.get_annotator("bounding_box_2d_tight")
bbox.attach(rp)

depth = rep.AnnotatorRegistry.get_annotator("distance_to_camera")
depth.attach(rp)
```

### Randomization

```python
# Define randomization
with rep.trigger.on_frame():
    # Randomize object placement
    with rep.create.group(["/World/Pallet"]):
        rep.modify.pose(
            position=rep.distribution.uniform((-2, -2, 0.5), (2, 2, 0.5)),
            rotation=rep.distribution.uniform((0, 0, 0), (0, 0, 360))
        )
    
    # Randomize lighting
    with rep.create.light():
        rep.modify.attribute(
            "intensity",
            rep.distribution.uniform(500, 2000)
        )
    
    # Randomize material colors
    with rep.get.prims(semantics=[("class", "pallet")]):
        rep.randomizer.color(
            colors=rep.distribution.uniform((0.3, 0.3, 0.3), (0.8, 0.8, 0.8))
        )

# Run data generation
rep.orchestrator.run(num_frames=1000, rt_subframes=4)
```

### Data Output

```python
# Configure writer
writer = rep.WriterRegistry.get("BasicWriter")
writer.initialize(
    output_dir="/data/synthetic",
    rgb=True,
    semantic_segmentation=True,
    bounding_box_2d_tight=True,
    distance_to_camera=True
)

# Attach writer
writer.attach(rp)

# Generated files:
# /data/synthetic/rgb/rgb_0000.png
# /data/synthetic/semantic_segmentation/semantic_segmentation_0000.png
# /data/synthetic/bounding_box_2d_tight/bounding_box_2d_tight_0000.json
# /data/synthetic/distance_to_camera/distance_to_camera_0000.npy
```

## Physics Simulation

### Dynamic Simulation

```python
from omni.isaac.core.prims import RigidPrim

# Add dynamic SimReady asset
pallet = RigidPrim(
    prim_path="/World/Pallet",
    usd_path="/Isaac/Props/Pallets/pallet.usd",
    position=[0, 0, 5]
)
world.scene.add(pallet)

# Simulation loop
for i in range(1000):
    world.step(render=True)
    
    # Get physics state
    position, orientation = pallet.get_world_pose()
    linear_velocity = pallet.get_linear_velocity()
    
    print(f"Frame {i}: Position {position}, Velocity {linear_velocity}")
```

### Contact Detection

```python
from omni.isaac.core.utils.extensions import enable_extension
enable_extension("omni.physx.tensors")

import omni.physics.tensors

# Get contact info
physics_interface = omni.physics.tensors.acquire_physics_interface()
contact_headers, contact_data = physics_interface.get_contacts()

# Process contacts
for header in contact_headers:
    if header.actor0 == "/World/Robot" or header.actor1 == "/World/Robot":
        print(f"Contact detected with {header.actor0} and {header.actor1}")
```

## Sensor Integration

### Camera Sensors

```python
from omni.isaac.sensor import Camera

# RGB camera
camera = Camera(
    prim_path="/World/Camera",
    position=[3, 0, 2],
    resolution=(1280, 720),
    frequency=30
)

# Get frame
frame = camera.get_rgba()
```

### LIDAR

```python
from omni.isaac.range_sensor import LidarRtx

# Create LIDAR
lidar = LidarRtx(
    prim_path="/World/Lidar",
    config="Example_Rotary"
)

# Get point cloud
data = lidar.get_current_frame()
points = data["point_cloud"]
```

### Depth Camera

```python
from omni.isaac.sensor import Camera

# Depth camera
depth_camera = Camera(
    prim_path="/World/DepthCamera",
    position=[3, 0, 2],
    resolution=(640, 480),
    frequency=30
)

# Get depth
depth = depth_camera.get_depth()
```

## Integration Patterns

### Warehouse Simulation

```python
# Load warehouse environment
add_reference_to_stage(
    usd_path="/Isaac/Environments/Simple_Warehouse/warehouse.usd",
    prim_path="/World/Warehouse"
)

# Populate with SimReady assets
assets = [
    ("/Isaac/Props/Pallets/pallet.usd", 50),
    ("/Isaac/Props/Shapes/box.usd", 100),
    ("/Isaac/Props/Cone/traffic_cone.usd", 20)
]

for asset_path, count in assets:
    for i in range(count):
        add_reference_to_stage(
            usd_path=asset_path,
            prim_path=f"/World/Asset_{i}",
            translation=random_position()
        )
```

### Training Loop

```python
import torch
from omni.isaac.gym.vec_env import VecEnvBase

# Create vectorized environment
env = VecEnvBase(
    headless=False,
    sim_device=0,
    enable_livestream=False,
    enable_viewport=True
)

# Add SimReady assets
env.set_task(
    task=MyTask,
    backend="torch",
    sim_params=sim_params
)

# Training loop
for epoch in range(1000):
    obs = env.reset()
    done = False
    
    while not done:
        # Agent policy
        action = policy(obs)
        
        # Step simulation
        obs, reward, done, info = env.step(action)
        
        # Update policy
        loss = update_policy(obs, action, reward)
```

## Best Practices

### Asset Selection

1. **Use built-in assets**: Tested and validated
2. **Check physics**: Verify colliders and mass
3. **Validate semantics**: Ensure labels present
4. **Test performance**: Monitor frame rate

### Scene Composition

1. **Modular design**: Use references
2. **Variants**: Create asset variations
3. **Instancing**: Reuse geometry
4. **LOD**: Use appropriate detail level

### Data Generation

1. **Domain randomization**: Vary lighting, materials, poses
2. **Semantic consistency**: Maintain label taxonomy
3. **Realistic scenarios**: Match target domain
4. **Validation**: Review generated data

### Performance

1. **Batch operations**: Load assets efficiently
2. **GPU utilization**: Monitor resource usage
3. **Streaming**: Use payloads for large scenes
4. **Culling**: Remove off-screen objects

## Common Use Cases

### Pick and Place Training

```python
# Setup scene
robot = add_manipulator()
objects = add_simready_objects(count=10)
gripper = configure_gripper()

# Training
for episode in range(1000):
    target = random.choice(objects)
    robot.reach(target.position)
    gripper.grasp(target)
    robot.move_to(goal_position)
    gripper.release()
```

### Autonomous Navigation

```python
# Setup
robot = add_mobile_robot()
obstacles = add_simready_obstacles()
goal = set_goal_position()

# Path planning
path = plan_path(robot.position, goal, obstacles)
robot.follow_path(path)
```

### Perception Training

```python
# Generate dataset
for i in range(10000):
    randomize_scene()
    capture_images()
    save_annotations()
    
# Train model
model = train_perception_model(dataset)
```
