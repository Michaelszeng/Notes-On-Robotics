# Notes-On-Robotics

6/9-7/7

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

Quaternions are an alternative to the traditional pitch, roll, yaw method of rotations (called Euler rotations), where all 3D rotations are a combination of those three. Euler rotations have a problem--"Gimbal Lock". When any of pitch, roll, or yaw is exactly 90 deg, the other 2 rotation axes will be colinear, losing you a degree of freedom. This means, if you wanted to rotate infinitesimally in the third, lost axis, you physically could not. I actually do not undestand gimbal lock so hopefully I will figure this out and bother to rewrite this section.

Quaternions, meanwhile, overcome this problem by having 4 parameters instead of 3. They're similar to axis-angle rotations, which might be easier to explain/understand. Axis-angle rotations simple define a unit vector using 3 parameters (x, y, z) to serve as the axis of rotation, and an angle θ to rotate around the axis. Quaternions can be easily derived from the angle-axis form: 

(θ, x, y, z) = (cos(θ/2), x*sin(θ/2), y*sin(θ/2), z*sin(θ/2)) = (q0, q1, q2, q3)

Observe that the magnitude of a quaaternion is always equal to 1. This is useful when applying quaternions to rotate vectors, so the vectors' magnitude will not change. This is also the main advantage of quaternions over angle-axis rotations. 

To apply a quaternion rotation to an actual vector, we first turn the (x, y, z) vector into a quaternion by adding a 4th element 0: (0, x, y, z). Let's define the rotation quaternion as `q` and the quaternion to be rotated as `p` (and the rotated quaternion as `p'`). To perform the rotation, we do 2 cross products: `p'` = `qpq^-1` (where the inverse of a rotation quaternion is the conjugate of the quaternion; the last three terms are multiplied by -1). To get the final rotated vector, we just extract the `x'`, `y'`, and `z'` from the rotated quaternion: (0, `x'`, `y'`, `z'`).

In practice, computers perform rotations using matrices, since matrix math is incredibly fast for computers to compute. Quaternions can also be easily converted to rotation matrices: 

![image](https://github.com/Michaelszeng/Notes-On-Robotics/assets/35478698/5241a0b3-83f0-4090-a9ec-3a48ab532207)

or, equivalently: 

![image](https://github.com/Michaelszeng/Notes-On-Robotics/assets/35478698/49c2d917-cf0e-48fb-9032-5632748e901c)


([source](https://danceswithcode.net/engineeringnotes/quaternions/quaternions.html))

Note that rotation matrices and quaternions are not susceptible to gimbal lock (for some reason?).

### Threading and Multi-processing
These past couple of weeks I've been working on a project called `voxl-apps`. The vision for this system we (mostly James) designed involves having an App Manager application constantly running in the background, searching for and initializing app .so files, and waiting for commands from a command pipe to begin running an app. When a proper command is received, an app is then run in a child process, with any inter-process communication done through pipes. This way, whenever a developer makes an app, no matter how bad they mess up the app (seg faults, or whatever), they can never crash the App Manager itself. The App Manager will be intelligent enough to ensure the drone is always doing something safe even when an app isn't actively running.

Firstly, this utilization of multi-processing is pretty interesting and not something I had thought much about before (typically multi-processing is thought of as a way to increase speed). Multiple threads are also employed to manage the app, watching their status to ensure they are only run-able when they are done initializing and ensuring they can be terminated on timeouts and on error. My former experience in threading and multi-processing involves some messing around in Python. So this complexity involved a learning curve. 

In C/POSIX land, there are complexities especially when using multi-threading and multi-processing at the same time. Creating threads in child processes is the easy case; since the child process basically becomes independent (no shared memory with its parent) there are no issues with doing this at all. However, some complexity arises when trying to fork new child processes in a multi-threaded application.

If the child process wants to modify any variables from the parent process, and those variables have a chance of being mutex locked at the time `fork()` is called, then there is a race condition in the child process; it's inconsistent whether that variable that the child wants to use will be mutex locked or not. This is definitely a problem, but doesn't mean that forking in a multi-threaded process should never be done.

If your child process doesn't need to modify any variables from the parent process, then forking the child from a multi-threaded parent is perfectly fine. `voxl-apps` is exactly like this; the child process that is forked off needs to know the name of the app that its parent created it to run, but after it knows the app name, the child process independently finds the app's .so file, reads the contents, and runs the app without ever having to use its parent's variables again. In this case, the child doesn't care about the state of mutex locks on all its parent's variables.

However, if your child process does need to modify variables it inherited from its parent, you need to be more careful. `pthread_atfork()` was made for this scenario, and provides opportunities to define handler functions that are called right before and right after the fork occurs, allowing you the opportunity to put all mutex locks into a known state when creating the child process. At least I think this is how it works--I have never actually tried.

Overall, internet wisdom seems to advise against forking in multi-threaded applications, but in certain cases it doesn't seem too bad at all.

Some sources I used in my research (in order of usefulness):
- https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_atfork.html
- https://thorstenball.com/blog/2014/10/13/why-threads-cant-fork/
- https://man7.org/linux/man-pages/man7/signal-safety.7.html
- https://stackoverflow.com/questions/39890363/what-happens-when-a-thread-forks
- http://www.qnx.com/developers/docs/7.1/#com.qnx.doc.neutrino.getting_started/topic/s1_procs_Multithreaded_fork.html

I've also taken more detailed notes about the technicalities of performing multi-threading and multi-processing in my [C Notes document](https://www.notion.so/C-Notes-dd071483f6274bf6bdd92fbbbb95dd66?pvs=4).
