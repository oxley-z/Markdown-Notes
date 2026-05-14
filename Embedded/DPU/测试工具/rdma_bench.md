# rdma_bench

编译过程



```bash
git clone https://github.com/efficient/rdma_bench.git
cmake .
```

报错

```bash
Error: src/_pylibmcmodule.h:42:10: fatal error: 'libmemcached/memcached.h' file not found
    #include <libmemcached/memcached.h>
```

安装libmemcached

```bash
sudo apt install liblibmemcached
```

报错

```bash
In file included from /home/eddy/code/test/rdma_bench/write-incomplete/main.cc:6:
/home/eddy/code/test/rdma_bench/libhrd_cpp/hrd.h:8:10: fatal error: numaif.h: 没有那个文件或目录
    8 | #include <numaif.h>
      |          ^~~~~~~~~~
compilation terminated.
```

安装libnuma-dev

```bash
sudo apt install libnuma-dev
```

