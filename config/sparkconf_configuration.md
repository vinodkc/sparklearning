# SparkConf to Code: How Configuration Reaches the Component That Needs It

This is the story of Spark's configuration system—how properties set at submission time propagate through the driver and executors to the exact component that needs them. Spark has hundreds of configuration properties spanning memory, scheduling, SQL, streaming, serialization, shuffle, and more. These properties can come from multiple sources, merge with different precedence rules, and need to reach components running in separate JVM processes on different machines. Understanding how configuration is assembled, frozen, distributed, and accessed explains why changing a configuration property sometimes has no effect, why some properties can only be set before SparkContext starts, and how to verify what configuration a running job is actually using.

---

## Where configuration comes from: the source hierarchy

Spark reads configuration from multiple sources, applied in increasing precedence order (higher overrides lower):

1. **Hardcoded defaults in Spark's source code**: every property has a default value baked into the code. This is the baseline if nothing else is specified.

2. **`$SPARK_HOME/conf/spark-defaults.conf`**: a properties file on the machine running `spark-submit`. Contains cluster-wide defaults set by the system administrator. For example: `spark.eventLog.enabled=true`, `spark.sql.shuffle.partitions=200`.

3. **Environment variables**: certain properties map to environment variables, e.g., `SPARK_EXECUTOR_MEMORY` sets `spark.executor.memory`. Legacy support for Hadoop-style configuration.

4. **`SparkConf` set in code**: properties set in application code before SparkContext starts:
   ```scala
   val conf = new SparkConf()
     .set("spark.sql.shuffle.partitions", "400")
   val spark = SparkSession.builder().config(conf).getOrCreate()
   ```

5. **`--conf` flags on spark-submit**: properties passed on the command line:
   ```bash
   spark-submit --conf spark.sql.shuffle.partitions=600 ...
   ```
   These take precedence over everything except runtime dynamic config.

6. **`SparkSession.conf.set()` at runtime** (for SQL properties): some properties can be changed after the SparkSession starts, affecting subsequent queries:
   ```python
   spark.conf.set("spark.sql.shuffle.partitions", "800")
   ```
   Only **runtime-modifiable** properties support this; attempting to change a static property (like `spark.executor.memory`) at runtime has no effect and may raise an error.

> **The configuration hierarchy is like a legal authority chain.** Base law (source code defaults) applies everywhere. Local ordinances (spark-defaults.conf) can narrow or override base law for the locale. Individual contracts (SparkConf in code) can add specific terms. And court orders (--conf flags) take precedence over all of these. Dynamic court injunctions (runtime config) can modify certain terms, but only those the law designates as modifiable.

---

## When configuration is frozen: the SparkContext lock

When `new SparkContext(conf)` or `SparkSession.builder.getOrCreate()` is called, Spark reads all configuration from all sources, merges them, and **locks the configuration**. After this point:
- Static properties (executor memory, executor cores, master URL, deploy mode) cannot be changed. The cluster manager was already contacted with these values; changing them would be meaningless.
- Dynamic properties (SQL shuffle partitions, broadcast threshold, AQE settings) can still be changed via `spark.conf.set()`, affecting queries that run after the change.

This is the source of a common confusion: setting `spark.executor.memory` in the application code via `SparkConf` works if set before SparkContext starts, but setting it after `SparkSession.builder.getOrCreate()` returns has no effect—the executor containers have already been requested with the previous value.

---

## How configuration reaches executors

When the driver starts an executor on a worker node, it passes configuration to the executor. The mechanism:

1. The driver's `SchedulerBackend` creates an executor launch command that includes `-Dspark.property=value` JVM system properties for relevant configuration values.
2. The executor JVM starts with these system properties baked into its environment.
3. The executor creates a `SparkConf` that reads from these system properties.
4. Configuration that affects executor-side components (memory fractions, serializer, shuffle settings) is now available to those components in the executor JVM.

For example, `spark.memory.fraction=0.6` is passed to the executor, which reads it when initializing the `UnifiedMemoryManager`. `spark.serializer=org.apache.spark.serializer.KryoSerializer` is read by the executor when creating the serializer for task deserialization and shuffle.

> **Configuration reaching the executor is like a franchisor sending operations manuals to each franchise location before it opens.** The head office (driver) knows all the rules; it prints the relevant pages and sends them to each location (executor) when it opens. Each location follows those rules from the moment it opens. Headquarters can't change a rule that affects how the kitchen was built—that's static. But they can send a memo changing the sauce recipe—that's dynamic.

---

## The `SparkConf` object in code

`SparkConf` is a simple key-value store with methods for getting, setting, and merging properties:

```scala
val conf = new SparkConf()
conf.set("spark.app.name", "MyApp")
conf.set("spark.master", "yarn")
conf.setExecutorEnv("JAVA_HOME", "/usr/lib/jvm/java-11")
conf.registerKryoClasses(Array(classOf[MyClass]))

val spark = SparkSession.builder().config(conf).getOrCreate()
```

Alternatively, `SparkSession.builder` accepts individual config calls:
```scala
val spark = SparkSession.builder()
  .appName("MyApp")
  .master("yarn")
  .config("spark.sql.shuffle.partitions", "400")
  .getOrCreate()
```

In PySpark, `SparkConf` and `SparkSession.builder.config()` work identically.

---

## Runtime configuration: `spark.conf`

For SQL and some streaming properties, you can read and write configuration at runtime through `spark.conf`:

```python
# Read current value
current = spark.conf.get("spark.sql.shuffle.partitions")

# Change it
spark.conf.set("spark.sql.shuffle.partitions", "800")

# All queries after this line use 800 partitions

# Unset (reset to default or spark-defaults.conf value)
spark.conf.unset("spark.sql.shuffle.partitions")
```

`spark.conf.isModifiable("spark.sql.shuffle.partitions")` returns `True` for modifiable properties and `False` for static ones. This is useful for diagnostic scripts that need to know whether a config change will take effect.

---

## The Environment tab: verifying actual configuration

After a job starts, the **Spark UI's Environment tab** shows the actual configuration values the driver is using. This includes:
- All Spark properties (from all sources, after merging)
- JVM properties (Java version, JVM flags)
- System environment variables
- Classpath entries

The Environment tab answers "what is Spark actually using?" rather than "what did I intend to set?" If a property isn't taking effect, the Environment tab is the first place to check: is the value what you expected? If not, a higher-precedence source (e.g., spark-defaults.conf) is overriding your setting.

> **The Environment tab is like checking the actual contract as signed, not just your draft.** You might have intended to specify `spark.sql.shuffle.partitions=400`, but if the executed contract shows `200`, something in the chain overrode your setting. The Environment tab shows the final, executed configuration—no ambiguity.

---

## Configuration scoping: per-session vs. global

In multi-session environments (e.g., a Spark Thrift Server or a notebook environment where multiple users share one SparkContext), configuration scoping matters:

- **Global `SparkConf` properties**: apply to the entire SparkContext—all sessions in the application.
- **Session-local properties** (set via `spark.conf.set()`): in a multi-session SparkContext, `spark.conf.set()` changes the property for the current SparkSession only, not for other sessions sharing the same SparkContext. This allows per-user or per-query configuration without affecting others.

This is implemented via a `SessionState` per SparkSession that maintains a separate configuration overlay on top of the shared `SparkConf`.

---

## Common configuration pitfalls

**Setting executor memory after SparkContext starts**: `spark.executor.memory` is static—the cluster manager was already called. Setting it after `getOrCreate()` returns does nothing.

**spark-defaults.conf overriding code settings**: `spark-defaults.conf` has lower precedence than code settings. But if you're using `spark.conf.set()` rather than `SparkConf.set()` (before SparkContext), and the property is static, it won't apply anyway.

**Properties with no effect without AQE/CBO**: properties like `spark.sql.cbo.enabled` have no effect unless the query planner actually calls the CBO, which requires statistics to be collected (`ANALYZE TABLE`). Setting the property without the statistics gives the appearance that the property doesn't work.

**Different values on driver vs. executor**: some properties affect both driver and executor (e.g., serializer). If you set a property only via executor-side environment variables but not in the SparkConf, the driver might use a different value.

---

## Bringing it together

Spark's configuration flows from source to component through a layered precedence system: **defaults < spark-defaults.conf < environment variables < SparkConf in code < --conf flags < runtime spark.conf.set()**. When SparkContext starts, configuration is frozen for static properties—executor memory, cores, and other infrastructure settings can't change after this point. Dynamic properties (SQL settings, AQE thresholds) can change at runtime via `spark.conf.set()`. Configuration reaches executors through the executor launch command, baked in as JVM system properties at startup. The Environment tab in the Spark UI shows the actual merged configuration—the ground truth for debugging why a property isn't taking effect. So the story of Spark configuration is: **collect from all sources → merge by precedence → freeze static properties at SparkContext start → distribute to executors via launch command → allow dynamic properties to be tuned at runtime.** Configuration is the control panel for Spark's behavior; understanding how the panel is wired prevents the frustration of turning a knob that isn't connected.
