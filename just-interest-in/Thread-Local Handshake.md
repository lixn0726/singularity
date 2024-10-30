# Thread-Local Handshake : New generation Safepoint for JVM

详见：https://openjdk.org/jeps/312

在 JDK10 之后已经发布，是一个 delivered feature。



## Summary

提供单独的 Safepoint Support for Single Java Thread。也就是尽可能低消耗的去暂停单个线程。

Introduce a way to execute a callback on threads without performing a global VM safepoint.



在引入这个新的 safepoint 机制的同时，保证了：

1. 在标准测试下，没有出现超过 1% 的性能损耗
2. 新的 safepoint 并不会增加传统的全局 safepoint 的性能



## Motivation

1. Improve the biased locking revocation performance.
2. Reduce the overall VM latency.
3. For safer stack trace (on signal).
4. Remove some memory barriers by performing Java Thread handshake.

All of these actions is to help the VM achieve a lower latency by reducing the number of global safepoint.

说白了就是减少全局 Safepoint，从而保证尽可能多的 JavaThread 不被打断，来减少应用的响应延迟。



## Description of Thread-Local Handshake

Handshake operation is a callback that is executed for each JavaThread while that thread is in a safepoint-safe state.

The callback is executed either by the thread itself or by the VMThread while the thread is in a `blocked` state.

Difference between ==Handshaking== and ==Safepoiting==:

Per thread operation will be performed on all threads ASAP and they will continue to execute ASA it's own operation is completed.

