---
title: "升级应用程序和Flink版本"
nav-parent_id: setup
nav-pos: 15
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* ToC
{:toc}


Flink DataStream程序通常设计为长时间运行，如数周，数月甚至数年。与所有长期运行的服务一样，Flink流应用程序需要维护，包括修复漏洞，实施改进，或将应用程序迁移到更高版本的Flink群集。 

本文档介绍了如何更新Flink流应用程序以及如何将正在运行的流应用程序迁移到不同的Flink群集。 

## 重新启动流应用程序

升级流应用程序或将应用程序迁移到不同群集的行动方案是基于Flink保存点[Savepoint]({{ site.baseurl }}/setup/savepoints.html)的功能。
保存点（Savepoint）是特定时间点上应用程序状态的一致快照。 

从运行的流应用程序获取保存点（Savepoint）有两种方式。 


* 生成一个保存点(savepoint)并继续处理.
```
> ./bin/flink savepoint <jobID> [pathToSavepoint]
```
建议定期生成保存点(savepoint)，以便能够从以前的时间点重新启动应用程序

* 生成保存点(savepoint)并将应用程序作为单个操作停止。
```
> ./bin/flink cancel -s [pathToSavepoint] <jobID>
```
这意味着应用程序在生成保存点(savepoint)完成后立即被取消，即在这个保存点之后再没有其他检查点（checkpoints）。 
给定从应用程序取得的保存点，相同或兼容的应用程序(查阅 [应用程序状态兼容性](#application-state-compatibility) 下面章节)
可以从该保存点开始。从保存点启动应用程序意味着其 operator 的状态被初始化，operator状态保存在保存点中。这是通过使用保存点启动应用程序来完成的。 
```
> ./bin/flink run -d -s [pathToSavepoint] ~/application.jar
```
当启动应用程序的 operator 在保存点被采用时，对原始应用程序 operator的状态（即应用程序保存点被取自）进行初始化。已启动的应用程序从保存点这一点开始继续处理。

**注意**: 即使Flink不断保存应用程序的状态，它也不能将写入还原到外部系统。如果在不停止应用程序的时候，不断生成保存点，这可能是一个问题。在这种情况下，应用程序可能在保存点生成后继续发出数据。重新启动的应用程序可能（取决于是否更改应用程序逻辑）再次发出相同的数据。对不不同的“SinkFunction”和存储系统，此行为的确切效果可能会有很大的不同。发出两次的数据可能会对像Cassandra这样的键值存储（key-value）进行幂等的写入，但是在附加到日志系统（如Kafka）的情况下会有问题。在任何情况下，应该仔细检查并测试重新启动的应用程序的行为。 

## 应用程序状态兼容性

当升级应用程序以修复漏洞或改进应用程序时，通常的目标是在保持其状态的同时更换正在运行的应用程序的应用程序逻辑。我们通过从原始应用程序获取的保存点启动升级的应用程序来实现此目的。但是，只有当两个应用程序都是*状态兼容*时才能操作，这意味着升级后的应用程序的 operator 能够以原始应用程序的 operator 的状态初始化其状态。 

在本节中，我们将讨论如何修改应用程序以保持状态兼容。

### 匹配 Operator 状态

When an application is restarted from a savepoint, Flink matches the operator state stored in the savepoint to stateful operators of the started application. The matching is done based on operator IDs, which are also stored in the savepoint. Each operator has a default ID that is derived from the operator's position in the application's operator topology. Hence, an unmodified application can always be restarted from one of its own savepoints. However, the default IDs of operators are likely to change if an application is modified. Therefore, modified applications can only be started from a savepoint if the operator IDs have been explicitly specified. Assigning IDs to operators is very simple and done using the `uid(String)` method as follows:

```
val mappedEvents: DataStream[(Int, Long)] = events
  .map(new MyStatefulMapFunc()).uid(“mapper-1”)
```

**Note:** Since the operator IDs stored in a savepoint and IDs of operators in the application to start must be equal, it is highly recommended to assign unique IDs to all operators of an application that might be upgraded in the future. This advice applies to all operators, i.e., operators with and without explicitly declared operator state, because some operators have internal state that is not visible to the user. Upgrading an application without assigned operator IDs is significantly more difficult and may only be possible via a low-level workaround using the `setUidHash()` method.

**Important:** As of 1.3.0 this also applies to operators that are part of a chain.

By default all state stored in a savepoint must be matched to the operators of a starting application. However, users can explicitly agree to skip (and thereby discard) state that cannot be matched to an operator when starting a application from a savepoint. Stateful operators for which no state is found in the savepoint are initialized with their default state.

### Stateful Operators and User Functions

When upgrading an application, user functions and operators can be freely modified with one restriction. It is not possible to change the data type of the state of an operator. This is important because, state from a savepoint can (currently) not be converted into a different data type before it is loaded into an operator. Hence, changing the data type of operator state when upgrading an application breaks application state consistency and prevents the upgraded application from being restarted from the savepoint. 

Operator state can be either user-defined or internal. 

* **User-defined operator state:** In functions with user-defined operator state the type of the state is explicitly defined by the user. Although it is not possible to change the data type of operator state, a workaround to overcome this limitation can be to define a second state with a different data type and to implement logic to migrate the state from the original state into the new state. This approach requires a good migration strategy and a solid understanding of the behavior of [key-partitioned state]({{ site.baseurl }}/dev/stream/state.html).

* **Internal operator state:** Operators such as window or join operators hold internal operator state which is not exposed to the user. For these operators the data type of the internal state depends on the input or output type of the operator. Consequently, changing the respective input or output type breaks application state consistency and prevents an upgrade. The following table lists operators with internal state and shows how the state data type relates to their input and output types. For operators which are applied on a keyed stream, the key type (KEY) is always part of the state data type as well.

| Operator                                            | Data Type of Internal Operator State |
|:----------------------------------------------------|:-------------------------------------|
| ReduceFunction[IOT]                                 | IOT (Input and output type) [, KEY]  |
| FoldFunction[IT, OT]                                | OT (Output type) [, KEY]             |
| WindowFunction[IT, OT, KEY, WINDOW]                 | IT (Input type), KEY                 |
| AllWindowFunction[IT, OT, WINDOW]                   | IT (Input type)                      |
| JoinFunction[IT1, IT2, OT]                          | IT1, IT2 (Type of 1. and 2. input), KEY |
| CoGroupFunction[IT1, IT2, OT]                       | IT1, IT2 (Type of 1. and 2. input), KEY |
| Built-in Aggregations (sum, min, max, minBy, maxBy) | Input Type [, KEY]                   |

### Application Topology

Besides changing the logic of one or more existing operators, applications can be upgraded by changing the topology of the application, i.e., by adding or removing operators, changing the parallelism of an operator, or modifying the operator chaining behavior.

When upgrading an application by changing its topology, a few things need to be considered in order to preserve application state consistency.

* **Adding or removing a stateless operator:** This is no problem unless one of the cases below applies.
* **Adding a stateful operator:** The state of the operator will be initialized with the default state unless it takes over the state of another operator.
* **Removing a stateful operator:** The state of the removed operator is lost unless another operator takes it over. When starting the upgraded application, you have to explicitly agree to discard the state.
* **Changing of input and output types of operators:** When adding a new operator before or behind an operator with internal state, you have to ensure that the input or output type of the stateful operator is not modified to preserve the data type of the internal operator state (see above for details).
* **Changing operator chaining:** Operators can be chained together for improved performance. When restoring from a savepoint taken since 1.3.0 it is possible to modify chains while preversing state consistency. It is possible a break the chain such that a stateful operator is moved out of the chain. It is also possible to append or inject a new or existing stateful operator into a chain, or to modify the operator order within a chain. However, when upgrading a savepoint to 1.3.0 it is paramount that the topology did not change in regards to chaining. All operators that are part of a chain should be assigned an ID as described in the [Matching Operator State](#Matching Operator State) section above.

## Upgrading the Flink Framework Version

This section describes the general way of upgrading Flink framework version from version 1.1.x to 1.2.x and migrating your
jobs between the two versions.

In a nutshell, this procedure consists of 2 fundamental steps:

1. Take a savepoint in Flink 1.1.x for the jobs you want to migrate.
2. Resume your jobs under Flink 1.2.x from the previously taken savepoints.

Besides those two fundamental steps, some additional steps can be required that depend on the way you want to change the
Flink version. In this guide we differentiate two approaches to upgrade from Flink 1.1.x to 1.2.x: **in-place** upgrade and 
**shadow copy** upgrade.

For **in-place** update, after taking savepoints, you need to:

  1. Stop/cancel all running jobs.
  2. Shutdown the cluster that runs Flink 1.1.x.
  3. Upgrade Flink to 1.2.x. on the cluster.
  4. Restart the cluster under the new version.

For **shadow copy**, you need to:

  1. Before resuming from the savepoint, setup a new installation of Flink 1.2.x besides your old Flink 1.1.x installation.
  2. Resume from the savepoints with the new Flink 1.2.x installation.
  3. If everything runs ok, stop and shutdown the old Flink 1.1.x cluster.

In the following, we will first present the preconditions for successful job migration and then go into more detail 
about the steps that we outlined before.

### Preconditions

Before starting the migration, please check that the jobs you are trying to migrate are following the
best practises for [savepoints]({{ site.baseurl }}/setup/savepoints.html). In particular, we advise you to check that 
explicit `uid`s were set for operators in your job. 

This is a *soft* precondition, and restore *should* still work in case you forgot about assigning `uid`s. 
If you run into a case where this is not working, you can *manually* add the generated legacy vertex ids from Flink 1.1 
to your job using the `setUidHash(String hash)` call. For each operator (in operator chains: only the head operator) you 
must assign the 32 character hex string representing the hash that you can see in the web ui or logs for the operator.

Besides operator uids, there are currently three *hard* preconditions for job migration that will make migration fail: 

1. as mentioned in earlier release notes, we do not support migration for state in RocksDB that was checkpointed using 
`semi-asynchronous` mode. In case your old job was using this mode, you can still change your job to use 
`fully-asynchronous` mode before taking the savepoint that is used as the basis for the migration.

2. The CEP operator is currently not supported for migration. If your job uses this operator you can (curently) not 
migrate it. We are planning to provide migration support for the CEP operator in a later bugfix release.

3. Another **important** precondition is that all the savepoint data must be accessible from the new installation and 
reside under the same absolute path. Please notice that the savepoint data is typically not self contained in just the created 
savepoint file. Additional files can be referenced from inside the savepoint file (e.g. the output from state backend 
snapshots)! There is currently no simple way to identify and move all data that belongs to a savepoint.


### STEP 1: Taking a savepoint in Flink 1.1.x.

First major step in job migration is taking a savepoint of your job running in Flink 1.1.x. You can do this with the
command:

```sh
$ bin/flink savepoint :jobId [:targetDirectory]
```

For more details, please read the [savepoint documentation]({{ site.baseurl }}/setup/savepoints.html).

### STEP 2: Updating your cluster to Flink 1.2.x.

In this step, we update the framework version of the cluster. What this basically means is replacing the content of
the Flink installation with the new version. This step can depend on how you are running Flink in your cluster (e.g. 
standalone, on Mesos, ...).

If you are unfamiliar with installing Flink in your cluster, please read the [deployment and cluster setup documentation]({{ site.baseurl }}/setup/index.html).

### STEP 3: Resuming the job under Flink 1.2.x from Flink 1.1.x savepoint.

As the last step of job migration, you resume from the savepoint taken above on the updated cluster. You can do
this with the command:

```sh
$ bin/flink run -s :savepointPath [:runArgs]
```

Again, for more details, please take a look at the [savepoint documentation]({{ site.baseurl }}/setup/savepoints.html).

## Compatibility Table

Savepoints are compatible across Flink versions as indicated by the table below:
                             
| Created with \ Resumed with | 1.1.x | 1.2.x |
| ---------------------------:|:-----:|:-----:|
| 1.1.x                       |   X   |   X   |
| 1.2.x                       |       |   X   |



## Limitations and Special Considerations for Upgrades from Flink 1.1.x to Flink 1.2.x
  
  - The maximum parallelism of a job that was migrated from Flink 1.1.x to 1.2.x is currently fixed as the parallelism of 
  the job. This means that the parallelism can not be increased after migration. This limitation might be removed in a 
  future bugfix release.


