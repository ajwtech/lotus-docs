---
title: "Maintenance"
description: "How to safely perform maintenance on a Lotus miner."
lead: "How to safely perform maintenance on a Lotus miner."
draft: false
menu:
    storage-providers:
        parent: "storage-providers-operate"
        identifier: "storage-providers-configure-lifecycle"
aliases:
    - /docs/storage-providers/lifecycle/
    - /storage-providers/configure/lifecycle/
weight: 385
toc: true
---

These operations are normally related to maintenances and upgrades. Given that storage providers are expected to submit proofs to the chain in a continuous fashion while running long and expensive operations, it is important that operators become familiar with how to manage some of the events in the storage provider lifecycle, so that they can be performed with the maximum guarantees.

## Safely restarting the miner daemon

The process of shutting down a miner and starting it again is complicated. Several factors need to be taken into account to be able to do it with all the guarantees:

- How long the storage provider plans to be offline.
- The existence and distribution of proving deadlines for the miner.
- The occurrence of ongoing sealing operations.

### Reducing the time offline

Given the need to continuously send proofs to the network, the storage provider should be offline as little as possible. _Offline-time_ includes the time it takes for the computer to restart the lotus-miner daemon fully. For these reasons, we recommend you follow these steps:

1. Rebuild, install any upgrades before restarting the lotus-miner process.
1. Ensure the proof parameters are on a fast storage drive like an NVMe drive or SSD. These are the proof parameters that are downloaded the first time the lotus-miner is started, and are saved to `var/tmp/filecoin-proof-parameters`, or `$FIL_PROOFS_PARAMETER_CACHE` if the environment variable is defined.

### Ensuring proofs for the current deadline have been sent

Shutting down your lotus-miner while there are still pending proving operations could get your [miner slashed](https://docs.filecoin.io/mine/slashing/). Check that there are no pending operations by running `lotus-miner proving info`. If any deadline shows a _block height_ in the past, do not restart yet.

In the following example, `Deadline Open` is 454, which is earlier than `Current Epoch` of 500. This miner should not be shut down or restarted:

```shell
$ lotus-miner proving info

Miner: t01001
Current Epoch:           500
Proving Period Boundary: 154
Proving Period Start:    154 (2h53m0s ago)
Next Period Start:       3034 (in 21h7m0s)
Faults:      768 (100.00%)
Recovering:  768
Deadline Index:       5
Deadline Sectors:     0
Deadline Open:        454 (23m0s ago)
Deadline Close:       514 (in 7m0s)
Deadline Challenge:   434 (33m0s ago)
Deadline FaultCutoff: 384 (58m0s ago)
```

In this next example, the miner can be safely restarted because no `Deadlines` are earlier than `Current Epoch`. You have about 45 minutes before the miner must be back online to declare faults. This is known as the `Deadline FaultCutoff`. If the miner has no faults, you have about an hour.

```shell
$ lotus-miner proving info

Miner: t01000
Current Epoch:           497
Proving Period Boundary: 658
Proving Period Start:    658 (in 1h20m30s)
Next Period Start:       3538 (in 25h20m30s)
Faults:      0 (0.00%)
Recovering:  0
Deadline Index:       0
Deadline Sectors:     768
Deadline Open:        658 (in 1h20m30s)
Deadline Close:       718 (in 1h50m30s)
Deadline Challenge:   638 (in 1h10m30s)
Deadline FaultCutoff: 588 (in 45m30s)
```

The `proving info` examples above show information for the current proving window and deadlines. If you wish to see any upcoming deadlines you can use:

```shell
$ lotus-miner proving deadlines
```

Every row corresponds to a deadline (a period of 30 minutes covering 24 hours). The current one is marked. This is sometimes useful to find a time of day in which the miner does not have to submit any proofs to the chain.

### Checking ongoing seal operations

To get an overview of your current sealing jobs with:

```shell
lotus-miner sealing jobs
```

Before shutting down the lotus-miner you should gracefully shutdown any lotus-workers, or at least stop the current ongoing sealing operations. You can make sure that your lotus-workers do not get any new sealing tasks by disabling sealing tasks on a given lotus-worker.

```shell
lotus-worker tasks disable --all
```

After the worker has finished their sealing tasks, you can stop the worker with:

```shell
lotus-worker stop
```

### Restarting the miner

With all the considerations above you can decide on the best moment to shutdown your miner:

```shell
lotus-miner stop
# When using systemd
# systemctl stop lotus-miner
```

You can restart the miner as soon as you wish. Workers do not need to be restarted as they will reconnect to the miner automatically when it comes back up.

## Graceful shutdown of lotus workers

Lotus [seal workers]({{< relref "seal-workers" >}}) can be restarted any time, but if they are in the middle of one of the sealing steps, then the operation will start again (from the last checkpoint). Its therefore recommended to gracefully let the lotus-worker finish its sealing tasks before stopping it. You can disable any new sealing tasks on the worker with:

```shell
lotus-worker tasks disable --all
```

After the worker has finished its sealing tasks, you can stop the lotus-worker gracefully with:

```shell
lotus-worker stop
```

## Changing storage locations

If you wish to change the location of your miner-related storage to a different path, for the miner or the seal workers, make sure that the Lotus miner and any seal workers are aware of the new location.

```sh
lotus-miner storage list
```

The above command will give you an overview of [storage locations known to the miner]({{< relref "custom-storage-layout" >}}). This information is stored in `~/.lotusminer/storage.json` (or `$LOTUS_MINER_PATH/storage.json` if defined). Lotus seal workers store all the data in the `~/.lotusworker` folder (or `$LOTUS_WORKER_PATH` if defined).

If you wish to change any of the storage locations of the Lotus miner, follow these steps:

1. Set your miner to reject any new storage and retrieval deals so the storage is not modified during the copy.
2. Copy the data **as is**, keeping it in the original location, while the miner is running. Since this usually involves moving large amounts of storage, it will take time. In the meatime our miner will keep tending to its tasks.
3. Once the data is copied, [stop the miner](#safely-restarting-the-miner-daemon), following the recommendations above.
4. Edit `storage.json` with the new location for the data and make the old data unavailable for the miner (by renaming, or unmounting) to make sure it is not going to be used anymore when you start it.
5. Start the miner.
6. Verify that things are working correctly. If so, you can discard the old copy.

If you wish to expand your storage, while keeping the current, you can always [add additional storage locations to a Lotus miner]({{< relref "custom-storage-layout" >}}) with `lotus storage attach` (see `--help`).

If you wish to change the storage location for any of the lotus workers:

1. Stop the Lotus Worker.
2. Move the data to the new location.
3. Set `$LOTUS_WORKER_PATH` accordingly.
4. Start the worker again.

Any operations that the worker was performing before stopping will be restarted from the last checkpoint (a point in which they can be restarted, which may correspond to the start of the current sealing phase).

{{< alert icon="warning" >}}
Moving data between different workers is not currently supported. Moving the worker storage folder to a different worker machine will not work as the miner expects the ongoing sealing operations to be completed by the worker they were assigned to in the first place.
{{< /alert >}}

## Using a different Lotus Node

If you are planning to run maintenance on the Lotus node used by the miner, or if you need to failover to a different Lotus node because the current one does not work, follow these steps:

1. [Stop the miner](#safely-restarting-the-miner-daemon)
2. Set the `FULLNODE_API_INFO` environment variable to the location of the new node:

```sh
export FULLNODE_API_INFO=<api_token>:/ip4/<lotus_daemon_ip>/tcp/<lotus_daemon_port>/http
```

Follow these steps to learn [how to obtain a token]({{< relref "reference/basics/api-access#api-tokens" >}}).

3. If you have not exported your wallets yet, export them now from the old node and re-import them to the new Lotus node.
4. Start the miner. It should now communicate with the new Lotus Node and, since it has the same wallets as the older one, it should be able to perform the necessary operations on behalf of the miner.

{{< alert icon="callout" >}}
Make sure your new Lotus Node is fully synced.
{{< /alert >}}
