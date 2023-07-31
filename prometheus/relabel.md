# How relabeling in Prometheus works

翻译自:  [How relabeling in Prometheus works | Grafana Labs](https://grafana.com/blog/2022/03/21/how-relabeling-in-prometheus-works/#labelkeeplabeldrop)

## Prometheus labels

标签是键值对的集合，允许我们对 Prometheus 指标中实际被测量的内容进行描述和组织。

例如，在测量 HTTP 延迟时，我们可以使用标签记录返回的 HTTP 方法和状态，调用的端点是哪个，以及负责请求的服务器是哪个。

每个键值标签对的唯一组合都会作为新的时间序列存储在 Prometheus 中，因此标签对于理解数据的基数非常重要，应避免使用不受限制的值作为标签。

## 内部标签

那么对于没有标签的指标怎么办呢？Prometheus 也为我们提供了一些内部标签。它们以双下划线开头，并在所有重新标记步骤都应用后被删除；这意味着除非我们显式地配置它们，否则它们将不可用。

我们可用的一些特殊标签包括：

| 标签名                  | 描述                                       |
| ----------------------- | ------------------------------------------ |
| **\__name\__**            | 抓取的指标名称                             |
| **\__address\__**         | 被抓取目标的主机名和端口号                 |
| **\__scheme\__**          | 被抓取目标的 URI 协议                      |
| **\__metrics_path\__**    | 被抓取目标的指标端点                       |
| **\__param\_\<name**\>    | 是传递给目标的第一个 URL 参数的值          |
| **\__scrape\_interval\__** | 目标的抓取间隔（实验性）                   |
| **\__scrape_timeout\__**  | 目标的超时时间（实验性）                   |
|  \__meta\_                 | 服务发现机制设置的特殊标签               |
| \__tmp                   | 用于临时存储标签值的特殊前缀，然后将其丢弃 |

现在我们了解了各种 relabel_config 规则的输入是什么，那么我们如何创建它们呢？它们实际上可以用来做什么？

## 应用阶段

关于重标记规则的一个令人困惑的来源是它们可以在 Prometheus 配置文件的多个部分中找到。

```yaml
# A list of scrape configurations.
scrape_configs:
    - job_name: "some scrape job"
    ...
 
    # List of target relabel configurations.
    relabel_configs:
    [ - <relabel_config> ... ]

    # List of metric relabel configurations.
    metric_relabel_configs:
    [ - <relabel_config> ... ]

# Settings related to the remote write.
remote_write:
    url: https://remote-write-endpoint.com/api/v1/push 
    ...

    # List of remote write relabel configurations.
    write_relabel_configs:
    [ - <relabel_config> ... ]
```

原因在于重新标记可以应用于指标生命周期的不同阶段，从选择我们想要抓取的可用目标，到筛选我们想要存储在 Prometheus 时间序列数据库中的内容以及发送到某些远程存储。

首先，relabel_configs 键可以作为抓取作业定义的一部分找到。这些重新标记步骤在抓取之前应用，并且仅能访问 Prometheus 服务发现添加的标签。它们允许我们过滤 SD 机制返回的目标，以及操作它设置的标签。

一旦目标被定义，metric_relabel_configs 步骤将在抓取之后应用，并允许我们选择要摄取到 Prometheus 存储中的系列。

最后，write_relabel_configs 块在将数据发送到远程端点之前对数据应用重新标记规则。这可以用于过滤基数高的指标，或将指标路由到特定的 remote_write 目标。

## <relabel_config> 基础块

 `<relabel_config>` 由七个字段组成. 它们是:
- Source_labels
- Separator (default = ;)
- Target_label
- Regex (default = (.*))
- Modulus
- Replacement (default = $1)
- Action (default = replace)

Prometheus 配置文件可能包含一个重新标记步骤的数组；它们按照定义的顺序应用于标签集。省略的字段将采用它们的默认值，因此这些步骤通常会更短。

### Source_labels 和 separator

让我们从 `source_labels` 开始。它需要一个或多个标签名称的数组，用于选择相应的标签值。如果在 source_labels 数组中提供了多个名称，则结果将是它们的值的内容，使用提供的 `separator ` 连接起来。

例如，考虑以下两个指标：

```PromQL
my_custom_counter_total{server="webserver01",subsystem="kata"} 192  1644075044000
my_custom_counter_total{server="sqldatabase",subsystem="kata"} 147  1644075044000
```

以下是 relabel_config：

```prometheus
source_labels: [subsystem, server]
separator: "@"
```

可以解析出这些值：

```
kata@webserver01
kata@sqldatabase
```

### Regex

`regex` 字段需要一个有效的 RE 2 正则表达式，并用于匹配从 `source_label` 和 `separator` 字段的组合中提取的值。正则表达式支持带括号的捕获组，稍后可以引用它们。

此块将匹配我们先前提取的两个值。

```prometheus
source_labels: [subsystem, server]
separator: "@"
regex: "kata@(.*)"
```

然而，此块不会匹配先前的标签，并且会终止此特定 relabel 步骤的执行。

```prometheus
source_labels: [subsystem, server]
separator: "@"
regex: "(.*)@redis"
```

默认的 regex 值是 (.*), 因此如果没有指定它，它将匹配整个输入。

### Replacement

如果提取的值与给定的 `regex` 匹配，则使用正则表达式替换并利用先前定义的任何捕获组来填充 `replacement`。回到我们提取的值，以及像这样的块：

```prometheus
source_labels: [subsystem, server]
separator: "@"
regex: "(.*)@(.*)"
replacement: "${2}/${1}"
```

会捕获@符号前后的内容，将它们交换位置，并用斜杠分隔它们。

```
webserver01/kata
sqldatabase/kata
```

替换的默认值为 $1，因此它将匹配正则表达式中的第一个捕获组，如果未指定正则表达式，则匹配整个提取的值。

### Target_label

如果 relabel 操作导致将某个值写入某个标签，`target_label` 定义写入替换值的标签名称。例如，以下块将设置一个标签，如 `{env="production"}`：

```prometheus
replacement: "production"
target_label: "env"
action: "replace"
```

继续上一个例子，这个重新标记步骤将把 `replacement` 设置为“my_new_label”。

```prometheus
- source_labels: [subsystem, server]
  separator: "@"
  regex: "(.*)@(.*)"
  replacement: "${2}/${1}"
  target_label: "my_new_label"
  action: "replace"
```

结果是：

```
{my_new_label="webserver01/kata"}
{my_new_label="sqldatabase/kata"}
```

### Modulus

最后，`modulus` 字段需要一个正整数。Relabel_config 步骤将使用此数字使用 `MD 5（提取的值）％modulus` 表达式填充 target_label。

## 可用操作

我们走了很长的路，但我们终于有所进展了。那么我们可以用这些构建块做什么？它们如何帮助我们日常工作？

有七个可用的操作可供选择，让我们仔细看一下。

### Keep/drop

Keep 和 drop 操作允许我们根据我们的标签值是否匹配提供的正则表达式来过滤目标和指标。

让我们回到我们之前的例子：

```promql
my_custom_counter_total{server="webserver01",subsystem="kata"} 192  1644075074000
my_custom_counter_total{server="sqldatabase",subsystem="kata"} 14700  1644075074000
```

在连接 `subsystem` 和 `server` 标签的内容之后，我们可以使用以下块删除暴露 webserver-01 的目标：

```prometheus
- source_labels: [subsystem, server]
  separator: "@"
  regex: "kata@webserver"
  action: "drop"
```

或者，如果我们处于具有多个子系统的环境中，但只想监视 kata，我们可以保留有关其特定目标或指标的信息，并删除与其他服务相关的所有内容。

```prometheus
- source_labels: [subsystem, server]
  separator: "@"
  regex: "kata@(.*)"
  action: keep
```

在许多情况下，这就是内部标签发挥作用的地方。

例如，您可以仅保留特定的指标名称。

```prometheus
- source_labels: [__name__]
  regex: “my_custom_counter_total|my_custom_counter_sum|my_custom_gauge”
  action: keep
```

或者，如果您正在使用 Prometheus 的 Kubernetes 服务发现，您可能希望删除来自测试或暂存命名空间的所有目标。

```prometheus
- source_labels: [__meta_kubernetes_namespace]
  regex: "testing|staging"
  action: drop
```

### Labelkeep/labeldrop

Labelkeep 和 labeldrop 操作允许过滤标签集本身。

在前面的例子中，我们可能不再对特定子系统标签进行跟踪。

下面的重新标记将删除所有 {subsystem="\<name\>"} 标签，但保留其他标签不变。

```prometheus
- regex: "subsystem"
  action: labeldrop
```

当然，我们也可以做相反的事情，只保留特定的一组标签，并删除所有其他标签。

```prometheus
- regex: "subsystem|server|shard"
  action: labelkeep
```

我们必须确保在应用 labelkeep 和 labeldrop 规则后，所有指标仍然具有唯一的标签。

### Replace

如果我们没有指定 relabeling 规则，Replace 是重新标记规则的默认操作，它允许我们使用 `replacement` 的内容覆盖单个标签的值。

如前所述，以下块将将 env 标签设置为提供的 `replacement`，因此将添加  `{env="production"}` 到标签集中。

```
- action: replace
  replacement: production
  target_label: env
```

当您将它与其他字段结合使用时，replace 操作最有用。

以下是另一个示例：

```
- action: replace
  source_labels: [__meta_kubernetes_pod_name,__meta_kubernetes_pod_container_port_number]
  separator: ":"
  target_label: address
```

上面的代码段将连接 `__meta_kubernetes_pod_name` 和 `__meta_kubernetes_pod_container_port_number` 中存储的值。然后，提取的字符串将被设置为 target_label 并可能导致 ` {address="podname:8080"} ` 的结果。

### Hashmod

Hashmod 操作提供了一种用于水平扩展 Prometheus 的机制。

重新标记步骤计算连接标签值的 MD 5 散列值对正整数 N 取模，从而产生一个位于 \[0，N-1\] 范围内的数字。

通过一个示例来更好地理解。考虑以下指标和重新标记步骤：

```
my_custom_metric{name="node",val="42"} 100

- action: hashmod
  source_labels: [name, val]
  separator: "-"
  modulus: 8
  target_label: __tmp_hashmod
```

```shell
$ python3
>>> import hashlib
>>> m = hashlib.md5(b"node-42")
>>> int(m.hexdigest(), 16) % 8
5
```

因此，最终会将 {__tmp=5} 添加到指标的标签集中。

这通常用于将多个目标分片到一组 Prometheus 实例中。以下规则可用于在 8 个 Prometheus 实例之间分配负载，每个实例负责抓取最终在 \[0, 7\] 范围内产生某个值的一组目标，并忽略所有其他目标：

```
- action: keep
  source_labels: [__tmp_hashmod]
  regex: 5
```

### Labelmap

Labelmap 操作用于将一个或多个标签对映射到不同的标签名称。

任何标签对，其名称与提供的正则表达式匹配，都将使用替换字段中给定的新标签名称进行复制，通过利用组引用（${1}、$ {2} 等）。

`replacement` 默认为仅为第一个捕获的正则表达式  `$1`，因此有时会被省略。

以下是一个示例。如果我们使用 Prometheus 的 Kubernetes SD，则我们的目标将暂时暴露一些标签，例如：

```
    __meta_kubernetes_node_name: The name of the node object.
    __meta_kubernetes_node_provider_id: The cloud provider's name for the node object.
    __meta_kubernetes_node_address_<address_type>: The first address for each node address type, if it exists.
…
    __meta_kubernetes_namespace: The namespace of the service object.
    __meta_kubernetes_service_external_name: The DNS name of the service. (Applies to services of type ExternalName)
    __meta_kubernetes_service_name: The name of the service object.
    __meta_kubernetes_service_port_name: Name of the service port for the target.
…
    __meta_kubernetes_pod_name: The name of the pod object.
    __meta_kubernetes_pod_ip: The pod IP of the pod object.
    __meta_kubernetes_pod_container_init: true if the container is an InitContainer
    __meta_kubernetes_pod_container_name: Name of the container the target address points to.
…
```

## Prometheus 的重新标记的常见用例

以下是重新标记的常见用例列表以及适合添加重新标记步骤的位置：
1. 当您要忽略一部分应用程序时；使用 relabel_config。
2. 当将目标分配到多个 Prometheus 服务器时；使用 relabel_config + hashmod。
3. 当您想忽略一部分高基数度指标时；使用 metric_relabel_config。
4. 当将不同的指标发送到不同的端点时；使用 write_relabel_config。
