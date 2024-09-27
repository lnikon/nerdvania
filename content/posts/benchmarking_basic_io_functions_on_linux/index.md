+++
title = 'Benchmarking basic IO functions on Linux'
date = 2024-08-20T00:32:37+04:00
draft = false
tags = ['linux', 'io', 'benchmark', 'cpp']
+++

## Intro
Hi there!

I am working on a database, particularly building the storage engine(https://github.com/lnikon/tinykvpp) right
now, and I assigned myself the task of creating separate file abstractions(e.g.
I really like how it is done in leveldb, thus having RandomAccessFile, SequentialFile, AppendOnlyFile, etc...).

## Buffered IO versus System IO
So, while designing my approach to this, I started evaluating my current way of doing file IO - fstreams
versus plain POSIX write(), and simple Google benchmark:

```cpp
static void
BM_BenchmarkFstreamWrite(benchmark::State & state) {

  const std::string filename("test_stream.txt");
  std::fstream fs(filename, std::fstream::in |
    std::fstream::out |
    std::fstream::app |
    std::fstream::ate);

  if (!fs.is_open()) {
    std::cerr << "unable to open" << filename << '\n';
    exit(EXIT_FAILURE);
  }

  std::string payload("aaaaa");

  for (auto _: state) {
    benchmark::DoNotOptimize(fs.write(payload.c_str(), payload.size()));
  }

  std::filesystem::remove(filename);
}
BENCHMARK(BM_BenchmarkFstreamWrite);

static void
BM_BenchmarkPosixWrite(benchmark::State & state) {

  const
    std::string filename("test_stream_2.txt");

  int
  fd = open(filename.c_str(), O_WRONLY | O_APPEND | O_CREAT, 0644);

  if (fd == -1) {
    std::cerr << "Unable to open " << filename << '\n';
  }

  std::string payload("bbbbb");

  for (
    auto _: state) {
    write(fd, payload.c_str(), payload.size());
  }

  std::filesystem::remove(filename);
}

BENCHMARK(BM_BenchmarkPosixWrite);
static void BM_BenchmarkFstreamWrite(benchmark::State & state) {
  const std::string filename("test_stream.txt");
  std::fstream fs(filename, std::fstream::in | std::fstream::out | std::fstream::app | std::fstream::ate);
  if (!fs.is_open()) {
    std::cerr << "unable to open" << filename << '\n';
    exit(EXIT_FAILURE);
  }

  std::string payload("aaaaa");
  for (auto _: state) {
    benchmark::DoNotOptimize(fs.write(payload.c_str(), payload.size()));
  }

  std::filesystem::remove(filename);
}
BENCHMARK(BM_BenchmarkFstreamWrite);

static void BM_BenchmarkPosixWrite(benchmark::State & state) {
  const std::string filename("test_stream_2.txt");
  int fd = open(filename.c_str(), O_WRONLY | O_APPEND | O_CREAT, 0644);
  if (fd == -1) {
    std::cerr << "Unable to open " << filename << '\n';
  }

  std::string payload("bbbbb");
  for (auto _: state) {
    write(fd, payload.c_str(), payload.size());
  }

  std::filesystem::remove(filename);
}
BENCHMARK(BM_BenchmarkPosixWrite);
```

shows following result:

```
    Benchmark Time CPU Iterations

    BM_BenchmarkFstreamWrite 24.0 ns 24.0 ns 30455681

    BM_BenchmarkPosixWrite 2001 ns 2001 ns 356052
```

## Payload size and std::fstream buffering

So, simple POSIX write() is slower almost 100 times. This result really shocked me.
I decided to strace the program and saw that for sufficiently big
inputs std::fstream::write's got vectorized! What do I mean by 'sufficiently big input'?

Let's examine the following small programs and their straces.
To simulate the load I wrapped the write() into a while loop.
The payload size is the same for all test cases, only the iteration count changes.

Iteration count: 3

```cpp
size_t count {
  3
};
std::string payload("aaa");
while (count-- != 0) {
  fs << payload;
}
```

Strace:
```
    openat(AT_FDCWD, "test_stream_3.txt", O_RDWR|O_CREAT|O_APPEND, 0666) = 3
    lseek(3, 0, SEEK_END) = 40755
    write(3, "aaaaaaaaa", 9) = 9
    close(3) = 0
```

What we see is a simple write of 9 characters. But what would I've expected is to see three writes of 3 chars. So std::fsteam does internal buffering? Maybe that's the way it derives from std::basic_filebuf?

Okay, next program:

Iteration count 9 * 1024:

```cpp
size_t count {
  9 * 1024
};
std::string payload("aaa");
while (count-- != 0) {
  fs << payload;
}
```

Strace:
```
    openat(AT_FDCWD, "test_stream_3.txt", O_RDWR|O_CREAT|O_APPEND, 0666) = 3

    lseek(3, 0, SEEK_END) = 49980

    writev(3, [{iov_base="aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., iov_len=8190}, {iov_base="aaa", iov_len=3}], 2) = 8193

    writev(3, [{iov_base="aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., iov_len=8190}, {iov_base="aaa", iov_len=3}], 2) = 8193

    writev(3, [{iov_base="aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., iov_len=8190}, {iov_base="aaa", iov_len=3}], 2) = 8193

    write(3, "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., 3069) = 3069

    close(3) = 0
```


So what we see is that std::fstream is smart enough to auto-vectorize IO! This is really surprising to me.
After seeing this trace I decided to dig into the libstdc++ and find out the logic behind vectorization,
and I found... nothing. Grep showed zero calls to writev().
A simple search in fstream and related headers did not reference writev().
So the natural question is: where does vectorization happen?

The next experiment I conducted was to try to see if maybe the kernel somehow decides
that it can vectorize things?

So I drafted the same logic using plain write() calls.

```cpp
std::string payload("bbbbb");
size_t count {
  9 * 1024
};
while (count-- != 0) {
  write(fd, payload.c_str(), payload.size());
}
```

strace:
```
    ......

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    write(3, "bbbbb", 5) = 5

    exit_group(0) = ?
```

Strace just shows a ton of calls to write: exactly what I expected when std::fstream::write()-ing
into the stream.

## Manual IO vectorizaton via writev()
And the last experiment was to benchmark explicitly vectorized IO against fstream.

```cpp
std::vector <iovec> iov;
std::string payload("bbbbb");
for (
  auto _: state) {
  iov.emplace_back(iovec {
    .iov_base = payload.data(), .iov_len = payload.size()
  });
}
writev(fd, iov.data(), iov.size());
```

And the benchmark showed interesting result:
```
    Benchmark Time CPU Iterations

    BM_BenchmarkFstreamWrite 24.0 ns 24.0 ns 28733493

    BM_BenchmarkPosixWrite 1828 ns 1828 ns 384717

    BM_BenchmarkPosixScatterWrite 37.9 ns 37.9 ns 16676420
```

DIY writev() is almost two times slower than std::fstream::write!

What are your interpretations of these results? How to understand where the vectorization happens? Maybe I should abandon my idea of having custom file abstractions and use std::fstream.

I am using 6.10.3-1-default with gcc 13.3.1 with optimization levels set to -O2.

Thank you for reading!
