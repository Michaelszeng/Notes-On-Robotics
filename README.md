# Notes-On-Robotics

### POSIX Pipes
POSIX pipes are a bit like ROS topics, but faster, more low level, and also simpler. They are a method of inter-process communication where one process running in the background can publish to pipes while client processes subsribe to a pipe and read from them.

`<unistd.h>` is the header file that imports POSIX pipe functionalities.

It has functions like `write()` and `read()` that take in a file descriptor and a char array + size and write/read the data to/from the POSIX pipe corresponding to that file descriptor.

File descriptors are an integer that is basically an ID for a POSIX pipe.

Reference: https://gitlab.com/voxl-public/voxl-sdk/core-libs/libmodal-pipe/-/blob/master/library/src/server.c

On their own, POSIX pipes are made for 1-to-1 communication, meaning one pipe can only allow one server to write to one client. However, similar to ROS topics, you many want multiple clients to be able to subscribe to data being published by a server. The way ModalAI overcomes this issue is by giving each server an array of POSIX pipes (a directory of POSIX pipe files). Whenever a new client wants to subscribe to the server's data, it requests a new pipe, and the server adds a new pipe for them to subscribe to.

ModalAI also puts the POSIX pipes in a directory that is not saved to disk, so that all pipes are wiped on reboot.

Internally, POSIX pipes are just FIFO buffers, but are also represented as files in the file system.

There is also a lot of multi-threading and mutex locking (using pthread.h) in the implementation to ensure that no pipe gets written to by 2 services at the same time and such.

Overall, this seems like one of the most low-level, no-overhead ways to perform inter-process communication. Apparently, much faster than ROS topics.

### Quaternions
I'm surprised I never explored these before. ModalAI uses quaternions to perform all their rotation between coordinate frames ([they have a lot of rotations](https://beta-docs.modalai.com/voxl-vision-px4-apriltag-relocalization-0_9/), in fact [this whole `.c` file](https://gitlab.com/voxl-public/voxl-sdk/services/voxl-vision-hub/-/blob/master/src/geometry.c) is for rotations), and my boss actually wrote the [matrix/vector math library](https://beta-docs.modalai.com/librc-math/) that powers the whole thing.

Quaternions are an alternative to the traditional pitch, roll, yaw method of rotations (called Euler rotations), where all 3D rotations are a combination of those three. Euler rotations have a problem--"Gimbal Lock". I had a lot of trouble understanding this, so I will do my best to explain. When any of pitch, roll, or yaw is exactly 90 deg, the other 2 rotation axes will be colinear. 

Quaternions, meanwhile, overcome this problem by having 4 parameters instead of 3. They're similar to axis-angle rotations, which might be easier to explain/understand. Axis-angle rotations simple define a unit vector using 3 parameters (x, y, z) to serve as the axis of rotation, and an angle θ to rotate around the axis. Quaternions can be easily derived from the angle-axis form: 

(θ, x, y, z) = (cos(θ/2), x*sin(θ/2), y*sin(θ/2), z*sin(θ/2)) = (q0, q1, q2, q3)

Observe that the magnitude of a quaaternion is always equal to 1. This is useful when applying quaternions to rotate vectors, so the vectors' magnitude will not change.

To apply a quaternion rotation to an actual vector, we first turn the (x, y, z) vector into a quaternion by adding a 4th element 0: (0, x, y, z). Let's define the rotation quaternion as `q` and the quaternion to be rotated as `p` (and the rotated quaternion as `p'`). To perform the rotation, we do 2 cross products: `p'` = `qpq^-1` (where the inverse of a rotation quaternion is the conjugate of the quaternion; the last three terms are multiplied by -1). To get the final rotated vector, we just extract the `x'`, `y'`, and `z'` from the rotated quaternion: (0, `x'`, `y'`, `z'`).

In practice, computers perform rotations using matrices, since matrix math is incredibly fast for computers to compute. Quaternions can also be easily converted to rotation matrices: 

![image](https://github.com/Michaelszeng/Notes-On-Robotics/assets/35478698/5241a0b3-83f0-4090-a9ec-3a48ab532207)

or, equivalently: 

![image](https://github.com/Michaelszeng/Notes-On-Robotics/assets/35478698/49c2d917-cf0e-48fb-9032-5632748e901c)


([source](https://danceswithcode.net/engineeringnotes/quaternions/quaternions.html))

Note that rotation matrices and quaternions are not susceptible to gimbal lock (for some reason?).

