# Notes-On-Robotics

### POSIX Pipes
POSIX pipes are a bit like ROS topics, but faster, more low level, and also simpler. They are a method of inter-process communication where one process running in the background can publish to pipes while client processes subsribe to a pipe and read from them.

`<unistd.h>` is the header file that imports POSIX pipe functionalities.

Has functions like `write()` and `read()` that take in a file descriptor and a char array + size and write/read the data to/from the POSIX pipe corresponding to that file descriptor.

File descriptors are an integer that is basically an ID for a POSIX pipe.

Reference: https://gitlab.com/voxl-public/voxl-sdk/core-libs/libmodal-pipe/-/blob/master/library/src/server.c

On their own, POSIX pipes are made for 1-to-1 communication, meaning one pipe can only allow one server to write to one client. However, similar to ROS topics, you many want multiple clients to be able to subscribe to data being published by a server. The way ModalAI overcomes this issue is by giving each server an array of POSIX pipes (a directory of POSIX pipe files). Whenever a new client wants to subscribe to the server's data, it requests a new pipe, and the server adds a new pipe for them to subscribe to.

Internally, the way POSIX pipes work is similar to just writing and reading to a file.

### Quaternions

