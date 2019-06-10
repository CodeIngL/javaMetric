## 入门指南 ##

入门指南将指导您完成将**Metrics**添加到现有应用程序的过程。 我们将通过**Metrics**提供的各种测量工具，如何使用它们以及什么时候派上用场。

### maven配置 ###

库依赖`metrics-core`

	<dependencies>
	    <dependency>
	        <groupId>io.dropwizard.metrics</groupId>
	        <artifactId>metrics-core</artifactId>
	        <version>${metrics.version}</version>
	    </dependency>
	</dependencies>

### Meters ###

**Meter**测量一段时间内的事件发生率（例如，“每秒请求数”）。除了平均速率之外，**Meter**还跟踪1，5和15分钟的均线。

	private final MetricRegistry metrics = new MetricRegistry();
	private final Meter requests = metrics.meter("requests");
	
	public void handleRequest(Request request, Response response) {
	    requests.mark();
	    // etc
	}
这个**Meter**将测量每秒请求的请求率。


### Console Reporter ###

一个控制台报告者正是它听起来像 - 报告给控制台。 将每秒打印一次。

	ConsoleReporter reporter = ConsoleReporter.forRegistry(metrics)
	       .convertRatesTo(TimeUnit.SECONDS)
	       .convertDurationsTo(TimeUnit.MILLISECONDS)
	       .build();
   	reporter.start(1, TimeUnit.SECONDS);

### 完整快速开始 ###

完整的入门指南是:

	  package sample;
	  import com.codahale.metrics.*;
	  import java.util.concurrent.TimeUnit;
	
	  public class GetStarted {
	    static final MetricRegistry metrics = new MetricRegistry();
	    public static void main(String args[]) {
	      startReport();
	      Meter requests = metrics.meter("requests");
	      requests.mark();
	      wait5Seconds();
	    }
	
	  static void startReport() {
	      ConsoleReporter reporter = ConsoleReporter.forRegistry(metrics)
	          .convertRatesTo(TimeUnit.SECONDS)
	          .convertDurationsTo(TimeUnit.MILLISECONDS)
	          .build();
	      reporter.start(1, TimeUnit.SECONDS);
	  }
	
	  static void wait5Seconds() {
	      try {
	          Thread.sleep(5*1000);
	      }
	      catch(InterruptedException e) {}
	  }
	}

maven 依赖

	<?xml version="1.0" encoding="UTF-8"?>
	<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	  <modelVersion>4.0.0</modelVersion>
	
	  <groupId>somegroup</groupId>
	  <artifactId>sample</artifactId>
	  <version>0.0.1-SNAPSHOT</version>
	  <name>Example project for Metrics</name>
	
	  <dependencies>
	    <dependency>
	      <groupId>io.dropwizard.metrics</groupId>
	      <artifactId>metrics-core</artifactId>
	      <version>${metrics.version}</version>
	    </dependency>
	  </dependencies>
	</project>

运行

	mvn package exec:java -Dexec.mainClass=sample.GetStarted

### Registry ###

Metric的核心是MetricRegistry类，它是所有应用程序指标的容器。 开始并新创建一个：

	final MetricRegistry metrics = new MetricRegistry();

你可能会想把它集成到你的应用程序的生命周期（也许使用你的依赖注入框架spring），但静态字段也会工作的很好

### Gauges ###

`gauge`是一个值的即时测量。 例如，我们可能想要测量队列中待处理作业的数量：

	public class QueueManager {
	    private final Queue queue;
	
	    public QueueManager(MetricRegistry metrics, String name) {
	        this.queue = new Queue();
	        metrics.register(MetricRegistry.name(QueueManager.class, name, "size"),
	                         new Gauge<Integer>() {
	                             @Override
	                             public Integer getValue() {
	                                 return queue.size();
	                             }
	                         });
	    }
	}

`gauge`进行工作时，将返回队列中的作业数量。


注册表中的每个指标都有一个唯一的名称，它只是一个像`things.count`或`com.example.Thing.latency`这样的虚线名称。 MetricRegistry有一个用于构造这些名字的静态帮助器方法：

	MetricRegistry.name(QueueManager.class, "jobs", "size")

这将返回一个类似`com.example.QueueManager.jobs.size`的字符串。


对于大多数队列和类队列结构，您不会希望简单地返回`queue.size（）`。大部分的`java.util`和`java.util.concurrent`都实现了`#size()`，它们是`O(n)`，这意味着你的`gauge`将会很慢（可能在持有锁的时候）。

### Counters ###

一个`counter`只是一个AtomicLong实例的尺度。 您可以增加或减少其值。 例如，我们可能需要一个更有效的方式来测量队列中待处理的作业：


	private final Counter pendingJobs = metrics.counter(name(QueueManager.class, "pending-jobs"));
	
	public void addJob(Job job) {
	    pendingJobs.inc();
	    queue.offer(job);
	}
	
	public Job takeJob() {
	    pendingJobs.dec();
	    return queue.take();
	}

每次测量这个`counter`时，它都会返回队列中的工作数量。


如你所见，`counters`的API稍有不同：`#counter（String）`而不是`#register（String，Metric）`。 虽然您可以使用注册并创建自己的`Counter`实例，**但是`#counter（String）`会为您完成所有工作，并允许您重用具有相同名称的度量标准**。


另外，我们在这个范围内静态导入了MetricRegistry的`name`方法来减少混乱。


### Histograms ###

`histogram`度量数据流中值的统计分布。 除了最小值，最大值，平均值等之外，它还测量中值，第75,90,95,98,99和99.9百分位数。

	private final Histogram responseSizes = metrics.histogram(name(RequestHandler.class, "response-sizes"));

	public void handleRequest(Request request, Response response) {
	    // etc
	    responseSizes.update(response.getContent().length);
	}

这个`histogram`将以字节为单位来测量响应的大小。

### Timers ###

`timer`测量一段代码被调用的速率和它的持续时间的分布。

	private final Timer responses = metrics.timer(name(RequestHandler.class, "responses"));

	public String handleRequest(Request request, Response response) {
	    final Timer.Context context = responses.time();
	    try {
	        // etc;
	        return "OK";
	    } finally {
	        context.stop();
	    }
	}

该`timer`将测量处理每个请求所需的时间（以纳秒为单位），并提供每秒请求的请求速率


### Health Checks ###

`Metrics`还可以使用`metrics-healthchecks`模块来集中您的服务的运行状况检查。

首先，创建一个新的`HealthCheckRegistry`实例：

	final HealthCheckRegistry healthChecks = new HealthCheckRegistry();


其次，实现一个`HealthCheck`子类：

	public class DatabaseHealthCheck extends HealthCheck {
	    private final Database database;
	
	    public DatabaseHealthCheck(Database database) {
	        this.database = database;
	    }
	
	    @Override
	    public HealthCheck.Result check() throws Exception {
	        if (database.isConnected()) {
	            return HealthCheck.Result.healthy();
	        } else {
	            return HealthCheck.Result.unhealthy("Cannot connect to " + database.getUrl());
	        }
	    }
	}

然后用`Metrics`注册它的一个实例：


	healthChecks.register("postgres", new DatabaseHealthCheck(database));

运行所有注册的健康检查：

	final Map<String, HealthCheck.Result> results = healthChecks.runHealthChecks();
	for (Entry<String, HealthCheck.Result> entry : results.entrySet()) {
	    if (entry.getValue().isHealthy()) {
	        System.out.println(entry.getKey() + " is healthy");
	    } else {
	        System.err.println(entry.getKey() + " is UNHEALTHY: " + entry.getValue().getMessage());
	        final Throwable e = entry.getValue().getError();
	        if (e != null) {
	            e.printStackTrace();
	        }
	    }
	}

`Metrics`带有预先构建的运行状况检查：`ThreadDeadlockHealthCheck`，它使用Java的内置线程死锁检测来确定是否有线程死锁。

### Reporting Via JMX ###

要通过JMX报告指标，请将`metrics-jmx`模块作为依赖项包含在内：

	<dependency>
	    <groupId>io.dropwizard.metrics</groupId>
	    <artifactId>metrics-jmx</artifactId>
	    <version>${metrics.version}</version>
	</dependency>

--

	final JmxReporter reporter = JmxReporter.forRegistry(registry).build();
	reporter.start();

一旦报告者启动后，注册表中的所有指标将通过`JConsole或VisualVM`（如果您安装MBeans插件）显示：
### Reporting Via HTTP ###

`Metrics`还附带了一个servlet（`AdminServlet`），它将提供所有注册指标的JSON表示。 它还将运行健康检查，打印出一个线程转储，并为负载均衡器提供一个简单的“ping”响应。 （它也有单独的Servlet，`MetricsServlet`，`HealthCheckServlet`，`ThreadDumpServlet`和`PingServlet`，它们完成这些单独的任务。）


要使用此servlet，请将`metrics-servlets`模块作为依赖项包含在内：

	<dependency>
	    <groupId>io.dropwizard.metrics</groupId>
	    <artifactId>metrics-servlets</artifactId>
	    <version>${metrics.version}</version>
	</dependency>

从那里开始，您可以将servlet映射到您认为合适的任何路径。

### Other Reporting ###

除了`JMX`和`HTTP`，`Metrics`还为以下输出提供了报告者：

- `STDOU`T, using `ConsoleReporter` from `metrics-core`
- `CSV` files, using `CsvReporter` from `metrics-core`
- `SLF4J` loggers, using Slf4jReporter from `metrics-core`
- `Graphite`, using GraphiteReporter from `metrics-graphite`

