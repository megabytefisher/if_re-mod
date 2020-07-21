# if_re-mod

a patched FreeBSD/pfsense Realtek network driver offering better throughput

## Purpose

Perfomance of the build-in FreeBSD/pfsense driver for Realtek cards is really bad. The latest 'if_re' driver distributed by Realtek ([available here](https://www.realtek.com/en/component/zoo/category/network-interface-controllers-10-100-1000m-gigabit-ethernet-pci-express-software)) gives much better performance, but still has bottlenecks. This repository contains a modified copy of the Realtek driver code with patches that aim to increase throughput.

## Disclaimer

The modifications made to this code are completely unsupported by Realtek and will probably void your warranty. By running the modified code, you alone are responsible for any potential damage to equipment caused by it. Do not run this code in production or critical environments.

I am not a professional driver developer, and you should assume I don't really know what I'm doing.

## Am I being bottlenecked by the driver?

To test whether or not you're affect by the issue this repo intends to solve, run `top -aSH` on your machine, and try to max out your network utilization. For me, the most reliable way to do this is by downloading a game through Steam. If `[intr{swi5: fast taskq}]` is maxing out near 100%, it may be limiting your speeds.

## What are the issues and modifications specifically?

When the Realtek chip sends an interrupt, the first driver function called is `re_intr`. This function just disables further interrupts, then enqueues an invocation of the `re_int_task` function on the `taskqueue_fast` global Kernel task queue.

Inside `re_int_task`, it acquires a mutex for the device, then reads/writes the chip's buffers through DMA. When done, it re-enables hardware interrupts.

### 1. Superfluous interrupt task enqueuing

In the stock driver's `re_int_task`, after processing buffers and unlocking the mutex, the interrupt register (`RE_ISR`) is checked again. If there's still an active interrupt, another incovation of `re_int_task` is queued instead of re-enabling hardware interrupts.

The modified driver instead just uses a while loop.

```
while ((status = CSR_READ_2(sc, RE_ISR)) & RE_INTRS)
```

### 2. Only process TX/RX when the corresponding interrupt register bit is set

The stock driver's `re_int_task` always syncs the device's TX/RX buffers, regardless of whether the corresponding interrupt bit is set. From what I can tell, it used to check this in old version of the 'rl' driver, so I'm not sure why it doesn't anymore.

```
if ((status & RE_ISR_RX_OK) || (status & RE_ISR_RX_ERR))
{
        re_rxeof(sc);
}
```

```
if ((status & RE_ISR_TX_OK) || (status & RE_ISR_TX_ERR))
{
        re_txeof(sc);
}
```

If you experience issues with the modified version, the first thing to try would be removing this part of the modifications. However, I haven't had any issues running it yet.

### 3. Dedicated task queues per-device

The stock driver uses the shared Kernel task queue `taskqueue_fast`. If you know more about FreeBSD than me (which wouldn't be surprising), then feel free to correct me here, but I believe only a single task on this queue runs at once. This queue shows up in `top -aSH` as `[intr{swi5: fast taskq}]`.

The modified driver sets up a dedicated queue in `re_attach` for each device. In `top -aSH`, each device will each an entry similar to `[intr{swi5: re0 intrp}]`, `[intr{swi5: re1 intrp}]`, etc..

## Building

`todo`

## Performance

Maybe one day I will run actual benchmarks, but for now..

I am running pfsense on an AWOW AK34 mini PC. It has two Realtek RTL8111H NICs, and I'm running my WAN connection in through one port with LAN out the other. With the unmodified Realtek driver, the total maximum download throughput was about 75MB/s. With the modified version, it gets over 105MB/s, which is probably near the limit of my WAN connection.

## License

```
/*
 * Copyright (c) 1997, 1998
 *	Bill Paul <wpaul@ctr.columbia.edu>.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *	This product includes software developed by Bill Paul.
 * 4. Neither the name of the author nor the names of any co-contributors
 *    may be used to endorse or promote products derived from this software
 *    without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY Bill Paul AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL Bill Paul OR THE VOICES IN HIS HEAD
 * BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF
 * THE POSSIBILITY OF SUCH DAMAGE.
 *
 * $FreeBSD: src/sys/dev/if_rl.c,v 1.38.2.7 2001/07/19 18:33:07 wpaul Exp $
 */

/*
 * RealTek 8129/8139 PCI NIC driver
 *
 * Written by Bill Paul <wpaul@ctr.columbia.edu>
 * Electrical Engineering Department
 * Columbia University, New York City
 */
```
