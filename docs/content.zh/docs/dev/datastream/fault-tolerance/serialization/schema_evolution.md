---
title: "状态数据结构升级"
weight: 7
type: docs
aliases:
  - /zh/dev/stream/state/schema_evolution.html
  - /zh/docs/dev/datastream/fault-tolerance/schema_evolution/
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

# 状态数据结构升级

Apache Flink 流应用通常被设计为永远或者长时间运行。
与所有长期运行的服务一样，应用程序需要随着业务的迭代而进行调整。
应用所处理的数据 schema 也会随着进行变化。

此页面概述了如何升级状态类型的数据 schema 。
目前对不同类型的状态结构（`ValueState`、`ListState` 等）有不同的限制

请注意，此页面的信息只与 Flink 自己生成的状态序列化器相关 [类型序列化框架]({{< ref "docs/dev/datastream/fault-tolerance/serialization/types_serialization" >}})。
也就是说，在声明状态时，状态描述符不可以配置为使用特定的 TypeSerializer 或 TypeInformation ，
在这种情况下，Flink 会推断状态类型的信息：

```java
ListStateDescriptor<MyPojoType> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        MyPojoType.class);

checkpointedState = getRuntimeContext().getListState(descriptor);
```

在内部，状态是否可以进行升级取决于用于读写持久化状态字节的序列化器。
简而言之，状态数据结构只有在其序列化器正确支持时才能升级。
这一过程是被 Flink 的类型序列化框架生成的序列化器透明处理的（[下面]({{< ref "docs/dev/datastream/fault-tolerance/serialization/schema_evolution" >}}#数据结构升级支持的数据类型) 列出了当前的支持范围）。

如果你想要为你的状态类型实现自定义的 `TypeSerializer` 并且想要学习如何实现支持状态数据结构升级的序列化器，
可以参考 [自定义状态序列化器]({{< ref "docs/dev/datastream/fault-tolerance/serialization/custom_serialization" >}})。
本文档也包含一些用于支持状态数据结构升级的状态序列化器与 Flink 状态后端存储相互作用的必要内部细节。

## 升级状态数据结构

为了对给定的状态类型进行升级，你需要采取以下几个步骤：

 1. 对 Flink 流作业进行 savepoint 操作。
 2. 升级程序中的状态类型（例如：修改你的 Avro 结构）。
 3. 从 savepoint 恢复作业。当第一次访问状态数据时，Flink 会判断状态数据 schema 是否已经改变，并进行必要的迁移。

用来适应状态结构的改变而进行的状态迁移过程是自动发生的，并且状态之间是互相独立的。
Flink 内部是这样来进行处理的，首先会检查新的序列化器相对比之前的序列化器是否有不同的状态结构；如果有，
那么之前的序列化器用来读取状态数据字节到对象，然后使用新的序列化器将对象回写为字节。

更多的迁移过程细节不在本文档谈论的范围；可以参考[文档]({{< ref "docs/dev/datastream/fault-tolerance/serialization/custom_serialization" >}})。

## 数据结构升级支持的数据类型

目前，仅支持 POJO 和 Avro 类型的 schema 升级
因此，如果你比较关注于状态数据结构的升级，那么目前来看强烈推荐使用 Pojo 或者 Avro 状态数据类型。

我们有计划支持更多的复合类型；更多的细节可以参考 [FLINK-10896](https://issues.apache.org/jira/browse/FLINK-10896)。

### POJO 类型

Flink 基于下面的规则来支持 [POJO 类型]({{< ref "docs/dev/datastream/fault-tolerance/serialization/types_serialization" >}}#pojo-类型的规则)结构的升级:

 1. 可以删除字段。一旦删除，被删除字段的前值将会在将来的 checkpoints 以及 savepoints 中删除。
 2. 可以添加字段。新字段会使用类型对应的默认值进行初始化，比如 [Java 类型](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html)。   
 3. 不可以修改字段的声明类型。
 4. 不可以改变 POJO 类型的类名，包括类的命名空间。

需要注意，只有从 1.8.0 及以上版本的 Flink 生产的 savepoint 进行恢复时，POJO 类型的状态才可以进行升级。
对 1.8.0 版本之前的 Flink 是没有办法进行 POJO 类型升级的。

### Avro 类型

Flink 完全支持 Avro 状态类型的升级，只要数据结构的修改是被
[Avro 的数据结构解析规则](http://avro.apache.org/docs/current/spec.html#Schema+Resolution)认为兼容的即可。

一个例外是如果新的 Avro 数据 schema 生成的类无法被重定位或者使用了不同的命名空间，在作业恢复时状态数据会被认为是不兼容的。

## 数据结构迁移限制

对 Flink 进行数据结构迁移时，需要遵循迁移的限制。如果你需要绕开这些限制并了解它们在特定用例中的安全性，请考虑使用
[自定义序列化器]({{< ref "docs/dev/datastream/fault-tolerance/serialization/custom_serialization" >}}) 或者
[状态处理器 API]({{< ref "docs/libs/state_processor_api" >}})。

### 不支持用作键的数据结构升级

用作键的数据结构无法被迁移，因为这可能会导致不确定的行为。
例如，将 POJO 作为键并且删除了其中的一个字段，可能会出现多个相同的键。Flink 不能合并相应的值。

另外，RocksDB 状态后端依赖二进制对象作为标识，而不是 `hashCode` 方法。任何对于键对象的结构更改都可能导致不确定的行为。

### 不支持使用 **Kryo** 的数据结构升级

使用 Kryo 时，Flink 框架无法验证是否进行了不兼容的更改。

{{< hint warning >}}
这意味着如果包含给定类型的数据结构通过 Kryo 序列化，则包含的数据类型**不能**进行数据结构升级。

例如，如果一个 POJO 包含 `List<SomeOtherPojo>`，并且 `List` 及其内容通过 Kryo 序列化，则 `SomeOtherPojo` **不能**进行数据结构升级。
{{< /hint >}}

{{< top >}}
