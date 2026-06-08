---
layout: page
title: Real Robot Development Experience
description: My development experiences and lessons learned on real robots
img: assets/img/real_robot_cover.png
importance: 10
category: work
related_publications: false
---

This page mainly introduces my engineering experience with embodied AI robots, with a focus on engineering. If you are not interested in engineering details, just read the **TL; DR**.

# TL; DR
I have experience building and operating various robotic arms, including Franka, dual-arm Aloha-like (Songling Piper and Ark Infinite), and Agibot G1. I have **complete experience in building basic infrastructure for robotic arm platforms**, including re-packaging control interfaces based on official SDK/ROS, data collection pipeline, model testing(and remote HTTP inference on H200 server), reproduction of the RECAP reinforcement learning process on real robots (successfully reproduced before the official open-sourcing of $\pi_{0.6}^*$), and deployment of traditional robotic arm control scheme AnyGrasp (all work except camera calibration).

# SDK/ROS interface re-packaging
The control SDK/ROS provided by robotic arm hardware often includes a large number of interfaces to allow users full control over the robot's behavior. For example, the official SDK may simultaneously provide robot status (error codes, etc.), MIT mode, position control mode, and various other interfaces encapsulating CAN bus data, resulting in dozens or even hundreds of interface functions.

However, most of these interfaces are unnecessary when developing embodied AI models. Typically, software and model developers expect a Gym-like interface, providing only necessary methods like reset and step, which greatly helps quickly validate model solutions. Therefore, re-packaging interfaces (e.g., into Gym-like interfaces) is necessary.

I have conducted complete interface development on **Franka, Songling Piper, and Agibot G1 robotic arms**.

- **Franka**: Due to the Franka arm's susceptibility to acceleration faults (e.g., rapid acceleration/deceleration), I used Facebook Polymetis to encapsulate control and integrated the Realsense camera image reading interface myself to complete the package.
- **Piper**: Since we purchased the Cobot Magic dual-arm complete system, the official package was fully provided. I mainly made two improvements: 1. Added keypoint functionality (triggered by a foot pedal button to record intermediate keypoints of tasks, such as subtask switching, error recovery, etc.). 2. Re-packaged to **support Human-in-Loop inference** (bidirectional action synchronization). Details will be introduced in the **RECAP section**.
- **Agibot**: The original Agibot SDK provides extensive adaptation and encapsulation for the base, lift, joints, and end-effector control, but requires a third-party control computer via Ethernet for debugging. I mainly implemented Quest teleoperation on the third-party control computer (4090 host), data collection pipeline (adapted to Quest teleoperation) with real-time keyframe annotation, and remote model inference (LAN remote HTTP inference on H200 server).

More details are provided below.

# Data Collection
This section mainly introduces my data collection experience on Agibot. (If you need APK or technical assistance, please contact me by email, **paid or free depending on the situation**).

<iframe width="800" height="450" src="https://www.youtube.com/embed/bYjGcRI-LnQ?si=DatTVFuXKZhPBBUd" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Agibot's official VR teleoperation costs 20,000 RMB, far higher than the market price (Pico/Quest costs only 3k-5k RMB per unit). Due to budget constraints, I implemented and tested VR teleoperation for Agibot (based on Quest) by myself in **4 days**, achieving a stable **50Hz control packet send/receive frequency**. I chose VR teleoperation because building gello or exoskeleton solutions requires purchasing many motors, costing 1k+ RMB per unit, scaling linearly with the number of arms. In contrast, a single VR headset can teleoperate many arms (depending on concurrent users), making it more cost-effective for academic teams.

## Quest side
Mainly responsible for reading the Quest's button and pose information, packaging the data packet, and sending it to the control host (LAN UDP communication). For agile development, the Quest side only needs to implement network connection and raw data reading/sending logic, without complex transformation logic on the Quest side. I mainly used the Quest official SDK in Unity (requires the international version of Unity, not the domestic one). It took about 0.5-1 day to complete and test.

## Python side
On the Python side, after decoding the Quest data packets using socket + struct, relative displacement and rotation control are performed through coordinate transformations. Using sub-threads for packet reception, lazy unpacking, numpy, scipy, and ROS, a stable 50Hz control can be achieved without special optimization.

### Coordinate transformation
This was the biggest challenge. The Quest uses a left-handed coordinate system, while the Agibot G1's left and right arms are not translation-invariant, making development complex. The control interface provided by the Agibot SDK directly controls in the default ROS coordinate system of the base (different from Franka and Piper).

For position (xyz), only a left-multiplication by a matrix is needed. By continuously printing the received Quest data packets, the Quest's coordinate system can be easily determined, and the transformation can be constructed accordingly.

For rotation (quaternions), after converting to rotation matrices using scipy, we need to left-multiply by one matrix and right-multiply by another. Specifically, the left-multiplied matrix is exactly the same as the position transformation, while the right-multiplied matrix is related to the SDK convention. There is no good empirical formula. It is recommended to record multiple sets of robot and Quest quaternions at the same pose (e.g., the other's quaternion under the unit quaternion) to compute the right-multiplication matrix.

### Relative position
Using absolute positions and quaternions directly is very dangerous because (1) the numerical mapping is difficult to linearize (related to headset position, which varies per person per day), and (2) it is difficult to place the Quest in a perfectly forward/right orientation. Therefore, **relative displacement and relative rotation control are strongly recommended**.

Specifically, transform the Quest's rotation matrix into the robot's coordinate system and construct the relative rotation result (all matrices below are in the ROS coordinate system):

$$
R_{ctrl} = R_{curr\_ctrl} \cdot  R_{quest\_init}^{T} \cdot R_{robot\_init}
$$

Then convert $R_{ctrl}$ back to quaternion for control.

## Other tips
- When both hands are reset simultaneously, force disable teleoperation to prevent accidents.
- When controlling the waist lift or tilt, disable arm control. It is strongly recommended to trigger a refresh of the initial pose after finishing to prevent accidents.
- Left hand handtrigger (middle finger): Press to toggle teleoperation Enable/Disable. The terminal shows green/red text to indicate the current state. Each Enable resets the starting point for relative displacement. You can safely adjust your hand posture in Disable mode to maintain comfort.
- Right hand handtrigger: Triggers a breakpoint, which can be used to mark task keypoints for later unified annotation, published via a ROS topic.

# Remote HTTP testing
Due to funding and network security constraints, I did not have suitable inference cards to serve each robotic arm independently. To accelerate development iteration, I referred to the $\pi_0$ team's solution and implemented **HTTP-based remote inference**. Over LAN, it can stably guarantee **20ms data transmission latency** (model inference latency is separate).

The main technology stack includes FastAPI, Uvicorn, and Msgpack. Designed for personal use only; not adapted for multi-user random concurrency, and no quantization or automatic routing.

# Real-robot RECAP reproduction

<iframe width="800" height="450" src="https://www.youtube.com/embed/AgQntl9kPYU?si=zEQiS0_yhpX6hsnV" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

$\pi_{0.6}^*$ applied the RECAP algorithm to a VLA model, but as of May 2026 it had not been open-sourced, so I reproduced the algorithm myself.

I used the Agilex Cobot Magic with four arms (two for teleoperation, two for execution). However, the official hardware merges the CAN signals of the master and slave arms, so I made **hardware and ROS improvements** to adapt for bidirectional synchronization.

1. Added two new CAN buses to separate the merged signals.
2. Re-implemented ROS control to change from hardware signal synchronization during teleoperation to software function synchronization.
3. Added **Human-in-Loop** inference functionality: use a foot pedal to trigger specific keys to switch modes. In inference mode, the master arm follows the slave arm; in intervention mode, the slave arm follows the master arm.

# AnyGrasp deployment
Deployed the AnyGrasp model on **Franka and Piper single arms** for grasp inference in a YOLO-like detection manner.

The core difficulty of deployment is coordinate calibration and transformation, where I was **responsible for the coordinate transformation part**. Specifically, an additional right-multiplication by an $R_{adjust}$ matrix is needed to transform the rotation from the base coordinate system to the flange coordinate system (according to the SDK's requirements).