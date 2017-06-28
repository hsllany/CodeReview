# Android Java世界的起点

ZygoteInit是Android Java世界的起点，其为一个java进程。

首先，其ZegoteInit，分裂出SystemServer进程；

每一个Process.start，会通过LocalSocket与ZygoteInit通讯，从而从其在分裂出应用的子进程。