# Faster IO through io_uring

来自于Youtube：https://www.youtube.com/watch?v=-5T4Cjw46ys

So the fundamental issue is that it needs to be so good that I don't go "why isn't this exactly the same as all the other failed clever things we've done"  - Linus



System call interface: Splice. By 





## What is it

- Fundamentally, ring based communication channel
- Submission Queue: SQ
  - struct io_uring_sqe
- Completion Queue: CQ
  - struct io_)uring_cqe
- All data shared between kernel and application
- Adds critically missing features
- Aim for easy to use, while powerful
  - Hard to misuse
- Flexible and extendable



## Ring Setup

- int io_uring_stup(u32 nentries, struct io_uring_params *p)

  - returns ring file descriptor (ring fd)

    struct io_uring_params {

      _u32 sq_entries;

      _u32 sq_entries;

      _u32 flags;

      _u32 sq_thread_cpu;

      _u32 sq_thread_idle;

      _u32 features;

      _u32 resv[4];

      struct io_sqring_offsets sq_off;

      struct io_cqring_offsets cq_off;

    }

    struct io_sqring_offsets {

      _32 head;

      _32 tail;

      _32 ring_mask;

      _32 ring_entries;

      _32 flag;

      _32 dropped;

      _32 array;

      _32 resv1;

      _64 resv2;

    }



## Ring access

#define IORING_OFF_SQ_RING

#define IORING_OFF_CQ_RING

#define IORING_OFF_SQES



// 用来获取ring的fd

sq->ring_ptr = mmap(0, sq->ring_sz, PROT_READ | PROT_WRITE,

​		MAP_SHARED | MAP_POPULATE, ring_fd, IORING_OFF_SQ_RING);

sq->khead = sq->ring_ptr + p-<sq_off.head;

sq->ktail = sq->ring_ptr + p-<sq_off.tail;



## Reading and writing rings

- head and tail indices free running
  - Integer wraps
  - Entry always head/tail masked with ring mask
- App produce SQ ring entries
  - Updates tail, kernel consumes at head
  - ->array[] holds index into ->sqes[]
  - Why not directly indexed ?
- Kernel produce CQ ring entries
  - Update tails, app consumes at head
  - ->cqes[] indexed directly 

也就是head/tail是不会被masked的