# JMX Exporter

JMX to Prometheus exporter: a collector that can configurably scrape and expose mBeans of a JMX target.  JMX 到 Prometheus 导出器：一个可以可配置地抓取和公开 JMX 目标的 mBean 的收集器。

This exporter is intended to be run as a Java Agent, exposing a HTTP server and serving metrics of the local JVM. It can be also run as an independent HTTP server and scrape remote JMX targets, but this has various disadvantages, such as being harder to configure and being unable to expose process metrics (e.g., memory and CPU usage). Running the exporter as a Java Agent is thus strongly encouraged.  此导出器旨在作为 Java 代理运行，公开 HTTP 服务器并提供本地 JVM 的指标。 它也可以作为独立的 HTTP 服务器运行并抓取远程 JMX 目标，但这有各种缺点，例如更难配置和无法公开进程指标（例如，内存和 CPU 使用率）。 因此，强烈建议将导出器作为 Java 代理运行。


## Running

The Java agent is available in two versions with identical functionality:  Java 代理有两个具有相同功能的版本：

* [jmx_prometheus_javaagent-0.16.0.jar](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.16.0/jmx_prometheus_javaagent-0.16.0.jar) requires Java >= 7.

* [jmx_prometheus_javaagent-0.16.0_java6.jar](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent_java6/0.16.0/jmx_prometheus_javaagent_java6-0.16.0.jar) is compatible with Java 6.

The only difference between these versions is the version of the bundled snakeyaml dependency. See [release notes](https://github.com/prometheus/jmx_exporter/releases/tag/parent-0.16.0) for more info.  这些版本之间的唯一区别是捆绑的snakeyaml 依赖项的版本。 有关更多信息，请参阅发行说明。

To run as a Java agent, download one of the JARs and run:

```bash
java -javaagent:./jmx_prometheus_javaagent-0.16.0.jar=8080:config.yaml -jar yourJar.jar
```

Metrics will now be accessible at http://localhost:8080/metrics

To bind the java agent to a specific IP change the port number to `host:port`.

See `./run_sample_httpserver.sh` for a sample script that runs the httpserver against itself.  请参阅 ./run_sample_httpserver.sh 以获取针对自身运行 httpserver 的示例脚本。

Please note that due to the nature of JMX the `/metrics` endpoint might exceed Prometheus default scrape timeout of 10 seconds.  请注意，由于 JMX 的性质，/metrics 端点可能会超过 Prometheus 默认的 10 秒抓取超时。

## Building

`./mvnw package` to build.

## Configuration

The configuration is in YAML. An example with all possible options:

```yaml
---
startDelaySeconds: 0
hostPort: 127.0.0.1:1234
username: 
password: 
jmxUrl: service:jmx:rmi:///jndi/rmi://127.0.0.1:1234/jmxrmi
ssl: false
lowercaseOutputName: false
lowercaseOutputLabelNames: false
whitelistObjectNames: ["org.apache.cassandra.metrics:*"]
blacklistObjectNames: ["org.apache.cassandra.metrics:type=ColumnFamily,*"]
rules:
  - pattern: 'org.apache.cassandra.metrics<type=(\w+), name=(\w+)><>Value: (\d+)'
    name: cassandra_$1_$2
    value: $3
    valueFactor: 0.001
    labels: {}
    help: "Cassandra metric $1 $2"
    cache: false
    type: GAUGE
    attrNameSnakeCase: false
```

* startDelaySeconds

start delay before serving requests. Any requests within the delay period will result in an empty metrics set.  在服务请求之前开始延迟。 延迟期内的任何请求都将导致指标集为空。

* hostPort

The host and port to connect to via remote JMX. If neither this nor jmxUrl is specified, will talk to the local JVM.  要通过远程 JMX 连接的主机和端口。 如果既没有指定 this 也没有指定 jmxUrl ，将与本地 JVM 对话。

* username

The username to be used in remote JMX password authentication.

* password

The password to be used in remote JMX password authentication.

* jmxUrl

A full JMX URL to connect to. Should not be specified if hostPort is.  要连接到的完整 JMX URL。 如果是 hostPort，则不应指定。

* ssl

Whether JMX connection should be done over SSL. To configure certificates you have to set following system properties:  JMX 连接是否应该通过 SSL 完成。 要配置证书，您必须设置以下系统属性：
`-Djavax.net.ssl.keyStore=/home/user/.keystore`
`-Djavax.net.ssl.keyStorePassword=changeit`
`-Djavax.net.ssl.trustStore=/home/user/.truststore`
`-Djavax.net.ssl.trustStorePassword=changeit`

* lowercaseOutputName

Lowercase the output metric name. Applies to default format and `name`. Defaults to false.  小写输出指标名称。 适用于默认格式和名称。 默认为假。

* lowercaseOutputLabelNames

Lowercase the output metric label names. Applies to default format and `labels`. Defaults to false.

* whitelistObjectNames

A list of [ObjectNames](http://docs.oracle.com/javase/6/docs/api/javax/management/ObjectName.html) to query. Defaults to all mBeans.  要查询的 ObjectName 列表。 默认为所有 mBean。

* blacklistObjectNames

A list of [ObjectNames](http://docs.oracle.com/javase/6/docs/api/javax/management/ObjectName.html) to not query. Takes precedence over `whitelistObjectNames`. Defaults to none.  不查询的 ObjectName 列表。 优先于 whitelistObjectNames。 默认为无。

* rules

A list of rules to apply in order, processing stops at the first matching rule. Attributes that aren't matched aren't collected. If not specified, defaults to collecting everything in the default format.  要按顺序应用的规则列表，处理在第一个匹配规则处停止。 不收集不匹配的属性。 如果未指定，则默认以默认格式收集所有内容。

* pattern

Regex pattern to match against each bean attribute. The pattern is not anchored. Capture groups can be used in other options. Defaults to matching everything.  匹配每个 bean 属性的正则表达式模式。 该模式未锚定。 捕获组可用于其他选项。 默认匹配所有内容。

* attrNameSnakeCase

Converts the attribute name to snake case. This is seen in the names matched by the pattern and the default format. For example, anAttrName to an\_attr\_name. Defaults to false.  将属性名称转换为蛇形大小写。 这可以从与模式和默认格式匹配的名称中看出。 例如，anAttrName 到 an_attr_name。 默认为假。

* name

The metric name to set. Capture groups from the `pattern` can be used. If not specified, the default format will be used. If it evaluates to empty, processing of this attribute stops with no output.  要设置的指标名称。 可以使用模式中的捕获组。 如果未指定，将使用默认格式。 如果计算结果为空，则此属性的处理将停止且没有输出。

* value

Value for the metric. Static values and capture groups from the `pattern` can be used. If not specified the scraped mBean value will be used.  指标的值。 可以使用模式中的静态值和捕获组。 如果未指定，将使用刮取的 mBean 值。

* valueFactor

Optional number that `value` (or the scraped mBean value if `value` is not specified) is multiplied by, mainly used to convert mBean values from milliseconds to seconds.  该值（如果未指定值，则为刮取的 mBean 值）乘以的可选数字，主要用于将 mBean 值从毫秒转换为秒。

* labels

A map of label name to label value pairs. Capture groups from `pattern` can be used in each. `name` must be set to use this. Empty names and values are ignored. If not specified and the default format is not being used, no labels are set.  标签名称到标签值对的映射。 可以在每个中使用模式中的捕获组。 name 必须设置为使用它。 空名称和值将被忽略。 如果未指定且未使用默认格式，则不会设置标签。

* help

Help text for the metric. Capture groups from `pattern` can be used. `name` must be set to use this. Defaults to the mBean attribute description and the full name of the attribute.  指标的帮助文本。 可以使用模式中的捕获组。 name 必须设置为使用它。 默认为 mBean 属性描述和属性的全名。

* cache

Whether to cache bean name expressions to rule computation (match and mismatch). Not recommended for rules matching on bean value, as only the value from the first scrape will be cached and re-used. This can increase performance when collecting a lot of mbeans. Defaults to `false`.  是否将 bean 名称表达式缓存到规则计算（匹配和不匹配）。 不推荐用于匹配 bean 值的规则，因为只有第一次刮取的值才会被缓存和重用。 这可以在收集大量 mbean 时提高性能。 默认为假。

* type

The type of the metric, can be `GAUGE`, `COUNTER` or `UNTYPED`. `name` must be set to use this. Defaults to `UNTYPED`.


Metric names and label names are sanitized. All characters other than `[a-zA-Z0-9:_]` are replaced with underscores, and adjacent underscores are collapsed. There's no limitations on label values or the help text.  指标名称和标签名称已清理。 除了 `[a-zA-Z0-9:_]` 之外的所有字符都替换为下划线，并且相邻的下划线折叠。 标签值或帮助文本没有限制。

A minimal config is `{}`, which will connect to the local JVM and collect everything in the default format. Note that the scraper always processes all mBeans, even if they're not exported.  最小配置是 {}，它将连接到本地 JVM 并以默认格式收集所有内容。 请注意，scraper 始终处理所有 mBean，即使它们没有被导出。

Example configurations for javaagents can be found at  可以在以下位置找到 javaagents 的示例配置
https://github.com/prometheus/jmx_exporter/tree/master/example_configs


### Pattern input


The format of the input matches against the pattern is  输入的格式与模式匹配是

```
domain<beanpropertyName1=beanPropertyValue1, beanpropertyName2=beanPropertyValue2, ...><key1, key2, ...>attrName: value
```

* domain

Bean name. This is the part before the colon in the JMX object name.  这是 JMX 对象名称中冒号之前的部分。

* beanPropertyName/Value

Bean properties. These are the key/values after the colon in the JMX object name.  这些是 JMX 对象名称中冒号后的键/值。

* keyN

If composite or tabular data is encountered, the name of the attribute is added to this list.  如果遇到复合或表格数据，则将属性名称添加到此列表中。

* attrName

The name of the attribute. For tabular data, this will be the name of the column. If `attrNameSnakeCase` is set, this will be converted to snake case.  属性的名称。 对于表格数据，这将是列的名称。 如果设置了 attrNameSnakeCase，这将被转换为蛇形大小写。

* value

The value of the attribute.

No escaping or other changes are made to these values, with the exception of if `attrNameSnakeCase` is set. The default help includes this string, except for the value.  不会对这些值进行转义或其他更改，除非设置了 attrNameSnakeCase。 默认帮助包括此字符串，但值除外。


### Default format

The default format will transform beans in a way that should produce sane metrics in most cases. It is  默认格式将以在大多数情况下应该产生合理指标的方式转换 bean。 这是

```
domain_beanPropertyValue1_key1_key2_...keyN_attrName{beanpropertyName2="beanPropertyValue2", ...}: value
```

If a given part isn't set, it'll be excluded.

## Testing

The JMX exporter uses [Testcontainers](https://www.testcontainers.org/) to run tests with different Java versions. You need to have Docker installed to run these tests.

You can run the tests with:

```
./mvnw verify
```

## Debugging

You can start the jmx's scraper in standalone mode in order to debug what is called  您可以在独立模式下启动 jmx

```bash
$ git clone https://github.com/prometheus/jmx_exporter.git

$ cd jmx_exporter

$ ./mvnw package

$ java -cp collector/target/collector*.jar io.prometheus.jmx.JmxScraper service:jmx:rmi:your_url

```

To get finer logs (including the duration of each jmx call), create a file called logging.properties with this content:  要获得更精细的日志（包括每个 jmx 调用的持续时间），请使用以下内容创建一个名为 logging.properties 的文件：

```
handlers=java.util.logging.ConsoleHandler
java.util.logging.ConsoleHandler.level=ALL
io.prometheus.jmx.level=ALL
io.prometheus.jmx.shaded.io.prometheus.jmx.level=ALL
```

Add the following flag to your Java invocation:

`-Djava.util.logging.config.file=/path/to/logging.properties`


## Installing

A Debian binary package is created as part of the build process and it can be used to install an executable into `/usr/bin/jmx_exporter` with configuration in `/etc/jmx_exporter/jmx_exporter.yaml`.  Debian 二进制包是作为构建过程的一部分创建的，它可用于将可执行文件安装到 /usr/bin/jmx_exporter 中，并在 /etc/jmx_exporter/jmx_exporter.yaml 中进行配置。

