# Processing Line Challenge

This challenge consists in implement a inter-process communication (IPC)
between a serie of processes that together form a processing line
similar to the assembly lines common in car industry.

The goal is to implement this processing line using any IPC offered by
the Linux OS:
[sockets](https://man7.org/linux/man-pages/man2/socket.2.html),
[pipes](https://man7.org/linux/man-pages/man2/pipe.2.html),
files,
[share memory](https://man7.org/linux/man-pages/man7/shm_overview.7.html),
[queues](https://man7.org/linux/man-pages/man7/mq_overview.7.html)
or any other or combination of.

Non-POSIX Linux-only are accepted too.

Even the use of 3rd party libraries are allowed too like
[zeromq](https://zeromq.org/) o
[dbus](https://en.wikipedia.org/wiki/D-Bus).

In any case the idea is to provide the fastest implementation while also
learn about IPC and have a good dose of coding.

## The binary blobs

The processing will be on binary blobs: a *generator* program `G`
generates several thousands of binary blobs that will *flow* through the
processing line.

These binary blobs range between 128 bytes and 64 MB of length and
follow this structure:

```cpp
struct binary_blob_t {
    uint64_t id;
    uint32_t slots[SLOTS];
    uint32_t len;
    uint8_t data[];
};
```

To make it clear `sizeof(struct binary_blob_t)` is in the range of 128
bytes and 64 MB.

The program `G` is in charge to create and initialize this structure:

 - set the unique incremental number `id`
 - zero'ed the `slots` array
 - set `len` to the length in bytes of the `data` field
 - and of course, setting the `data` itself.

Because the challenge focus on the communication of the processes and
not in the processing itself, `G` basically will generate random blobs.

The `slots` is a fixed length array of `SLOTS` elements defined at
compile time. During the processing, each process along the processing
line will compute a value based on the blob and it will store it in the
blob itself, in one of the slots.

Perhaps is a good time to introduce to you these processes.

## The (W) process

This program `W` is called as follows:

```shell
./worker <slot num> <src addr> [<dst addr>]
```

The worker will receive a binary blob from the *source address* and it
will compute a `CRC32` over `data` and store the result in `slots[num]`
where `num` is the *slot number* received from command line.

Once done the processing, the worker will forward the blob to the
*destination address* (acts as a *forwarder*).

If the *destination address* is not given, the worker must return the
blob to its source (acts as a *replier*).

Exactly what are theses addresses and how the worker will receive and
send the binary blobs is up to you.

## The broker (B) process

The broker program `B` is called as follows:

```shell
./broker <fan in> <fan out> <src addr> [<src addr>...] <dst addr> [<dst addr>...]
```

The broker will receive from one or more *source addresses* (the *fan
in*) the binary blobs and it will send each blob to *one* of its
*destination addresses* (the *fan out*).

Which destination address the broker should pick is up to you. You could
do a simple round robin or a more complex schema to maximize the
throughput.

Remember, the broker sends the blob to only one of its destination
addresses.

## The chief (C) process

The chief program `C` is called as follows:

```shell
./chief <src addr> <dst addr> <worker addr> [<worker addr>...]
```

It receives from the *source address* a binary blob but instead of doing
the processing itself, it sends the binary blob to one of its *workers*.

This part is similar to the broker `B` but there is a catch: once completed
the processing each worker will send the blob back to the chief.

Once that happen the load balancer will forward the blob to its
*destination address*.

Like in the broker, to which of its workers the chief should send the
blob for processing is up to you.

## The end-of-line (X) process

Everything must come to an end at some point. The end-of-line program
`X` is the *sink* of the pipeline where all the blobs will end.

The program is called as:

```
./eol <max slot num> <src addr>
```

On each received blob the `X` program will verify the values in the
`slots` and it will print an error on `stdout` if there are unexpected
values. To know how many slots needs to check the *maximum slot number*
assigned to a worker is passed via command line.



## The processing line architectures

The challenge has a few different architectures to test how will your
IPC system works on different scenarios.

The first architecture is the simplest:

```
G -> X
```

This scenario will put under test how well you can move blobs from `G`
to `X`.

The next is slightly more challenging:

```
G -> W -> X
```

This setup not only makes you move blobs but also `W` needs to compute
something on them.

The natural progression is:

```
G -> C -> X
    /|\
   W W W
```

Now with the help of the chief `C` your pipeline will process the
blobs concurrently on the workers `W`.

And this is another way to achieve concurrency:

```
      /-> W --\
G -> B -> W -> B -> X
      \-> W --/
```

This ultimate architecture will test how well you can fan out and fan in
binary blobs.


