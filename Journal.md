## Week of Feruary 4
I built a forward kinematics (joint-space control) example, and it works in both simulation and on hardware. The franka_drake driver targets firmware v5.x, but my Panda runs firmware v4.x, which is compatible with libfranka 0.8.0—not the newer versions used by franka_drake. To address this, I build the example twice in separate directories: one for simulation and one for hardware. The goal of this week is to integrate gripper control and understand drake libraries.
## Week of January 31
The goal of this week is to build my own examples that utilize drake and run those examples on simulation and hardware. Also downgrade the simulation setup to use libfranka 0.8.0 (compatible with franka panda research robot V4), same as in hardware setup. The end goal is that same code is compatible with hardware and simulation.
## Week of January 23
The goal of this week is to run the franka panda simulations via drake and run libfranka built-in examples on simulation and hardware. I will be using the franka-drake driver from https://github.com/KhachDavid/franka_drake.git 
