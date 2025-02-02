# Top500 Benchmark - HPL Linpack

[![CI](https://github.com/geerlingguy/top500-benchmark/workflows/CI/badge.svg?branch=master&event=push)](https://github.com/geerlingguy/top500-benchmark/actions?query=workflow%3ACI)

A common generic benchmark for clusters (or extremly powerful single node workstations) is Linpack, or HPL (High Performance Linpack), which is famous for its use in rankings in the [Top500 supercomputer list](https://top500.org) over the past few decades.

I wanted to see where my various clusters and workstations would rank, historically ([you can compare to past lists here](https://hpl-calculator.sourceforge.net/hpl-calculations.php)), so I built this Ansible playbook which installs all the necessary tooling for HPL to run, connects all the nodes together via SSH, then runs the benchmark and outputs the result.

## Why not PTS?

Phoronix Test Suite includes [HPL Linpack](https://openbenchmarking.org/test/pts/hpl) and [HPCC](https://openbenchmarking.org/test/pts/hpcc) test suites. I may see how they compare in the future.

When I initially started down this journey, the PTS versions didn't play nicely with the Pi, especially when clustered. And the PTS versions don't seem to support clustered usage at all!

## Supported OSes

Currently supported OSes:

  - Ubuntu (20.04+)
  - Raspberry Pi OS (11+)
  - Debian (11+)
  - Rocky Linux (9+)
  - AlmaLinux (9+)
  - CentOS Stream(9+)
  - RHEL (9+)
  - Fedora (38+)
  - Arch Linux
  - Manjaro

Other OSes may need a few tweaks to work correctly. You can also run the playbook inside Docker (see the note under 'Benchmarking - Single Node'), but performance will be artificially limited.

## Benchmarking - Cluster

Make sure you have Ansible installed (`pip3 install ansible`), then copy the following files:

  - `cp example.hosts.ini hosts.ini`: This is an inventory of all the hosts in your cluster (or just a single computer).
  - `cp example.config.yml config.yml`: This has some configuration options you may need to override, especially the `ssh_*` and `ram_in_gb` options (depending on your cluster layout)

Each host should be reachable via SSH using the username set in `ansible_user`. Other Ansible options can be set under `[cluster:vars]` to connect in more exotic clustering scenarios (e.g. via bastion/jump-host).

Tweak other settings inside `config.yml` as desired (the most important being `hpl_root`—this is where the compiled MPI, ATLAS/OpenBLAS/Blis, and HPL benchmarking code will live).

> **Note**: The names of the nodes inside `hosts.ini` must match the hostname of their corresponding node; otherwise, the benchmark will hang when you try to run it in a cluster. 
> 
> For example, if you have `node-01.local` in your `hosts.ini` your host's hostname should be `node-01` and not something else like `raspberry-pi`.

Then run the benchmarking playbook inside this directory:

```
ansible-playbook main.yml
```

This will run three separate plays:

  1. Setup: downloads and compiles all the code required to run HPL. (This play takes a long time—up to many hours on a slower Raspberry Pi!)
  2. SSH: configures the nodes to be able to communicate with each other.
  3. Benchmark: creates an `HPL.dat` file and runs the benchmark, outputting the results in your console.

After the entire playbook is complete, you can also log directly into any of the nodes (though I generally do things on node 1), and run the following commands to kick off a benchmarking run:

```
cd ~/tmp/hpl-2.3/bin/top500
mpirun -f cluster-hosts ./xhpl
```

> The configuration here was tested on smaller 1, 4, and 6-node clusters with 6-64 GB of RAM. Some settings in the `config.yml` file that affect the generated `HPL.dat` file may need diffent tuning for different cluster layouts!

### Benchmarking - Single Node

To run locally on a single node, clone or download this repository to the node where you want to run HPL. Make sure the `hosts.ini` is set up with the default options (with just one node, `127.0.0.1`).

All the default configuration from `example.config.yml` should be copied to a `config.yml` file, and all the variables should scale dynamically for your node.

Run the following command so the cluster networking portion of the playbook is not run:

```
ansible-playbook main.yml --tags "setup,benchmark"
```

> For testing, you can start an Ubuntu docker container:
> 
> ```
> docker run --name top500 -it -v $PWD:/code geerlingguy/docker-ubuntu2204-ansible:latest bash
> ```
>
> Then go into the code directory (`cd /code`) and run the playbook using the command above.

#### Setting `performance` CPU frequency

If you get an error like `CPU Throttling apparently enabled!`, you may need to set the CPU frequency to `performance` (and disable any throttling or performance scaling).

For different OSes and different CPU types, the way you do this could be different. So far the automated `performance` setting in the `main.yml` playbook has only been tested on Raspberry Pi OS. You may need to look up how to disable throttling on your own system. Do that, then run the `main.yml` playbook again.

### Overclocking

Since I originally built this project for a Raspberry Pi cluster, I include a playbook to set an overclock for all the Raspberry Pis in a given cluster.

You can set a clock speed by changing the `pi_arm_freq` in the `overclock-pi.yml` playbook, then run it with:

```
ansible-playbook overclock-pi.yml
```

Higher clock speeds require more power and thus more cooling, so if you are running a Pi cluster with just heatsinks, you may also require a fan blowing over them if running overclocked.

## Results

Here are a few of the results I've acquired in my testing:

| Configuration | Result | Wattage | Gflops/W |
|--- |--- |--- |--- |
| [Raspberry Pi 5 (BCM2712 @ 2.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/18) | 30.249 Gflops | 11W | 2.75 Gflops/W |
| [Raspberry Pi 4 (BCM2711 @ 1.8 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/13) | 11.889 Gflops | 7.2W | 1.65 Gflops/W |
| [Turing Pi 2 (4x CM4 @ 1.5 GHz)](https://www.jeffgeerling.com/blog/2021/turing-pi-2-4-raspberry-pi-nodes-on-mini-itx-board) | 44.942 Gflops | 24.5W | 1.83 Gflops/W |
| [Turing Pi 2 (4x CM4 @ 2.0 GHz)](https://www.jeffgeerling.com/blog/2021/turing-pi-2-4-raspberry-pi-nodes-on-mini-itx-board) | 51.327 Gflops | 33W | 1.54 Gflops/W |
| [DeskPi Super6c (6x CM4 @ 1.5 GHz)](https://www.jeffgeerling.com/blog/2022/pi-cluster-vs-ampere-altra-max-128-core-arm-cpu) | 60.293 Gflops | 40W | 1.50 Gflops/W |
| [DeskPi Super6c (6x CM4 @ 2.0 GHz)](https://www.jeffgeerling.com/blog/2022/pi-cluster-vs-ampere-altra-max-128-core-arm-cpu) | 70.338 Gflops | 51W | 1.38 Gflops/W |
| [Radxa ROCK 5B (RK3588 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/8) | 51.382 Gflops | 12W | 4.32 Gflops/W |
| [Orange Pi 5 (RK3588S 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/14) | 53.333 Gflops | 11.5W | 4.64 Gflops/W |
| [Lenovo M710q Tiny (1x i5-7400T @ 2.4 GHz)](https://www.jeffgeerling.com/blog/2023/rock-5-b-not-raspberry-pi-killer-yet) | 72.472 Gflops | 41W | 1.76 Gflops/W |
| M2 MacBook Air (1x M2 @ 3.5 GHz, in Docker) | 104.68 Gflops | N/A | N/A |
| [M1 Max Mac Studio (1x M1 Max @ 3.2 GHz, in Docker)](https://github.com/geerlingguy/top500-benchmark/issues/4) | 264.32 Gflops | 66W | 4.00 Gflops/W |
| AMD Ryzen 5 5600x @ 3.7 GHz | 229 Gflops | 196W | 1.16 Gflops/W |
| [Ampere Altra Max Q64-22 @ 2.2 GHz](https://github.com/geerlingguy/top500-benchmark/issues/19) | 655.90 Gflops | 140W | 4.69 Gflops/W |
| [Ampere Altra Max M96-28 @ 2.8 GHz](https://github.com/geerlingguy/top500-benchmark/issues/10) | 1,188.3 Gflops | 295W | 4.01 Gflops/W |
| [Ampere Altra Max M128-28 @ 2.8 GHz](https://github.com/geerlingguy/top500-benchmark/issues/17) | 1,265.5 Gflops | 296W | 4.27 Gflops/W |
| [Ampere Altra Max M128-30 @ 3.0 GHz](https://github.com/geerlingguy/top500-benchmark/issues/3) | 953.47 Gflops | 500W | 1.91 Gflops/W |

You can [enter the Gflops in this tool](https://hpl-calculator.sourceforge.net/hpl-calculations.php) to see how it compares to historical top500 lists.

### Other Listings

Over the years, as I find other people's listings of HPL results—especially those with power usage ratings—I will add them here:

  - [VMW Research Group GFLOPS/W listing](https://web.eece.maine.edu/~vweaver/group/green_machines.html)
