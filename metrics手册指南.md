## Metrics Core ##

Metrics的核心库是`metrics-core`，它提供了一些基本的功能：

- Metric注册表
- 5种测量指标:`Gauges`, `Counters`, `Histograms`, `Meters`和`Timers`
- 通过`JMX`,`console`,`CSV`文件和`SLF4J`来形成报告

### Metric 注册表 ###

Metric的起点是`MetricRegistry`类，它是您的应用程序（或应用程序的子集）的所有度量标准的集合。

通常，每个应用程序只需要一个`MetricRegistry`实例，但如果要在特定报告组中组织度量标准，则可以选择使用更多实例。

全局命名注册表也可以通过**静态**`SharedMetricRegistries`类共享。 这允许在代码的不同部分中使用相同的注册表，而不显式地传递`MetricRegistry`实例。

像所有的Metrics类一样，`SharedMetricRegistries`完全是线程安全的。


### Metric 名字 ###

每个`metric`都与一个`MetricRegistry`关联，并且在该注册表中具有**唯一名称**。 这是一个简单的点式名称，比如`com.example.Queue.size`。 这种灵活性使您可以将各种上下文直接编码为一个度量的名称。 如果你有**两个**`com.example.Queue`实例，你可以给它们更具体的例子：`com.example.Queue.requests.size`和`com.example.Queue.responses.size`。

`MetricRegistry`具有一组用于轻松创建名称的**静态帮助器方法**：


	MetricRegistry.name(Queue.class, "requests", "size")
	MetricRegistry.name(Queue.class, "responses", "size")

这些方法也将删除任何`null`值，允许简单的可选范围。

### Gauges ###

`gauge`是最简单的度量类型。 它只是返回一个值。 例如，如果您的应用程序具有由第三方库维护的值，则可以通过注册一个返回该值的`Gauge`实例来轻松地将其公开：


	registry.register(name(SessionStore.class, "cache-evictions"), new Gauge<Integer>() {
	    @Override
	    public Integer getValue() {
	        return cache.getEvictionsCount();
	    }
	});

这将创建一个名为`com.example.proj.auth.SessionStore.cache-evictions`的新指标，它将从缓存中返回evictions次数。

### JMX Gauges ###

鉴于许多第三方库通常只通过`JMX`公开度量标准，因此Metrics提供了`JmxAttributeGauge`类，该类使用JMX MBean的对象名称和属性名称，并生成一个标准实现，该实现返回该属性的值：

	registry.register(name(SessionStore.class, "cache-evictions"),
                 new JmxAttributeGauge("net.sf.ehcache:type=Cache,scope=sessions,name=eviction-count", "Value"));

### Ratio Gauges ###

`ratio gauge`是一种简单的方法来创建一个衡量这是两个数字之间的比例：


	public class CacheHitRatio extends RatioGauge {
	    private final Meter hits;
	    private final Timer calls;
	
	    public CacheHitRatio(Meter hits, Timer calls) {
	        this.hits = hits;
	        this.calls = calls;
	    }
	
	    @Override
	    public Ratio getRatio() {
	        return Ratio.of(hits.getOneMinuteRate(),
	                        calls.getOneMinuteRate());
	    }
	}

该`gauge`使用`meter`和`timer`返回高速缓存命中与未命中的比率。

### Cached Gauges ###

`cached gauge`允许更高效地报告计算起来昂贵的值。 该值在构造函数中**指定的时间段内被缓存**。 客户端调用的`getValue（）`方法只返回缓存的值。 受保护的`loadValue（）`方法仅在内部调用以重新加载缓存值。

	registry.register(name(Cache.class, cache.getName(), "size"),
                  new CachedGauge<Long>(10, TimeUnit.MINUTES) {
                      @Override
                      protected Long loadValue() {
                          // assume this does something which takes a long time
                          return cache.getSize();
                      }
                  });

### Derivative Gauges ###

`derivative gauge`可以让您从其他`gauges`中导出数值：

	public class CacheSizeGauge extends DerivativeGauge<CacheStats, Long> {
	    public CacheSizeGauge(Gauge<CacheStats> statsGauge) {
	        super(statsGauge);
	    }
	
	    @Override
	    protected Long transform(CacheStats stats) {
	        return stats.getSize();
	    }
	}

### Counters ###

`counter`是一个简单的递增和递减的64位整数：

	final Counter evictions = registry.counter(name(SessionStore.class, "cache-evictions"));
	evictions.inc();
	evictions.inc(3);
	evictions.dec();
	evictions.dec(2);

所有`Counter`度量标准从0开始。

### Histograms ###

`Histogram`测量数据流中值的分布：例如搜索返回的结果数量：

	final Histogram resultCounts = registry.histogram(name(ProductDAO.class, "result-counts");
	resultCounts.update(results.size());


`Histogram`度量可以让您衡量的不仅仅是简单的事情，如最小值，平均值，最大值和标准差值，还包括像中位数或第95百分位数的`分位数`。

传统上，计算中位数（或任何其他分位数）的方式是将整个数据集进行排序，并将中间值（或从最后的1％，第99个百分点）中取值。 这适用于小数据集或批处理系统，但不适用于高吞吐量，低延迟服务。

解决方案是在数据通过时对数据进行采样。 通过维护一个统计上代表整个数据流的小型，可管理的`reservoir`，我们可以快速简便地计算分位数，这些分位数是实际分位数的有效近似值。 这种技术被称为`reservoir sampling`。

`Metrics`提供了许多不同的`Reservoir`实现，每个实现都是有用的。

### Uniform Reservoirs ###


一个带有`uniform reservoir`的`histogram `产生的分位数是有效的整个直方图的生命周期。 它将返回一个中间值，例如，这是直方图曾经更新过的所有值的中值。 它通过使用一种叫做`Vitter's R`）的算法来完成这个任务，该算法随机选择具有线性递减概率的`reservoir`值。

如果您对长期测量感兴趣，请使用`uniform histogram`。 不要使用在你最近想要知道基础数据流的分布是否发生变化的地方。

### Exponentially Decaying Reservoirs ###

一个带有` exponentially decaying reservoir`的`histogram `产生分位数，这些分位数代表（大致）最后五分钟的数据。这是通过使用一个`forward-decaying priority reservoir`来实现的，这个优先级库对指向新数据的指数加权。与`uniform reservoir`不同， `exponentially decaying reservoir`代表最近的数据，使您可以很快知道数据的分布是否已经改变。`Timers`默认使用`exponentially decaying reservoirs`的`histograms`。

### Sliding Window Reservoirs ###

一个带有`sliding window reservoir`的`histogram`产生的分位数代表过去的`N`个测量。

### Sliding Time Window Reservoirs ###

一个带有`sliding window reservoir`的`histogram`产生分位数，该分位数严格代表过去的`N`秒（或其他时间段）。

**Warning**

虽然`SlidingTimeWindowReservoir`比`ExponentiallyDecayingReservoir`更容易理解，但其大小没有限制，因此使用它来采样高频过程可能需要大量的内存。因为它记录了每个测量值，所以它也是最慢的`reservoir`类型。

**Hint** 

尝试使用`SlidingTimeWindowReservoir`的新优化版本`SlidingTimeWindowArrayReservoir`。 它带来了更低的内存开销。 而且它的分配/自由模式也不同，所以GC的开销比`SlidingTimeWindowReservoir`低60x-80x。 现在`SlidingTimeWindowArrayReservoir`在GC开销和性能方面与`ExponentiallyDecayingReservoir`相当。 至于所需的内存，`SlidingTimeWindowArrayReservoir`每个存储的度量需要大约128位，您可以简单地计算所需的堆量。

例如：10K测量/秒，储存时间为1分钟将需要10000 * 60 * 128/8 = 9600000字节〜9兆字节

### Meters ###

`meter`测量一系列事件的发生率：

	final Meter getRequests = registry.meter(name(WebProxy.class, "get-requests", "requests"));
	getRequests.mark();
	getRequests.mark(requests.size());


`Meters`通过几种不同的方式来衡量事件的发生率。 平均速度是平均事件率。 这对trivia通常是有用的，但由于它代表了应用程序整个生命周期的总速率（例如，处理的请求总数除以进程运行的秒数），因此它不会提供最近的因子。 幸运的是，`meters`还记录了三种不同的指数加权平均移动平均速率：1，5和15分钟均线。

**Hint**

就像Unix中的`uptime`或者`top`加载平均值一样。

### Timers ###

`timer`基本上是事件类型的持续时间的`histogram`和其发生率的`meter`。


	final Timer timer = registry.timer(name(WebProxy.class, "get-requests"));
	
	final Timer.Context context = timer.time();
	try {
	    // handle request
	} finally {
	    context.stop();
	}

**Note**

使用Java的高精度`System.nanoTime()`方法，在内部以纳秒为单位测量事件的耗用时间。 其精度和准确度取决于操作系统和硬件。

### Metric Sets ###

`Metrics`也可以使用`MetricSet`接口组合到可重用的度量标准集中。 这使得库作者能够为各种功能的检测提供一个入口点。

### Reporters ###

`Reporters`是应用程序导出所有度量指标的方式。 `metrics-core`附带四种导出指标的方式：`JMX`，`console`，`SLF4J`和`CSV`。

### JMX ###

使用`JmxReporter`，您可以将度量标准公开为`JMX MBean`。 为了探索这个，你可以使用安装了`VisualVM-MBeans`插件的`VisualVM`（随大多数JDK提供的`jvisualvm`）或者`JConsol`e（随大多数JDK提供的jconsole）：

**Tip**

如果双击任何度量标准属性，VisualVM将开始绘制该属性的数据。 Sweet, eh?

**Warning**

	我们不建议您尝试从您的生产环境收集指标。 JMX的RPC API是脆弱的和疯狂的。 但是，为了开发和浏览，这可能非常有用。

通过JMX报告指标：

	final JmxReporter reporter = JmxReporter.forRegistry(registry).build();
	reporter.start();

### Console ###

对于简单的基准，`Metrics`随附了ConsoleReporter，它定期向控制台报告所有注册的metrics：

	final ConsoleReporter reporter = ConsoleReporter.forRegistry(registry)
	                                                .convertRatesTo(TimeUnit.SECONDS)
	                                                .convertDurationsTo(TimeUnit.MILLISECONDS)
	                                                .build();
	reporter.start(1, TimeUnit.MINUTES);

### CSV ###

对于更复杂的基准测试，`Metrics`随`CsvReporter`一起定期追加到给定目录下的一组`.csv`文件中：

	final CsvReporter reporter = CsvReporter.forRegistry(registry)
                                        .formatFor(Locale.US)
                                        .convertRatesTo(TimeUnit.SECONDS)
                                        .convertDurationsTo(TimeUnit.MILLISECONDS)
                                        .build(new File("~/projects/data/"));
	reporter.start(1, TimeUnit.SECONDS);

对于每个注册的指标，一个`.csv`文件将被创建，每一秒的状态将被写入一个新的行。

### SLF4J ###

还可以将度量标准记录到`SLF4J`记录器：
	
	final Slf4jReporter reporter = Slf4jReporter.forRegistry(registry)
	                                            .outputTo(LoggerFactory.getLogger("com.example.metrics"))
	                                            .convertRatesTo(TimeUnit.SECONDS)
	                                            .convertDurationsTo(TimeUnit.MILLISECONDS)
	                                            .build();
	reporter.start(1, TimeUnit.MINUTES);

### Other Reporters ###

度量也有其他的`reporter`实现：

- `MetricsServlet`是一个servlet，它不仅将您的指标公开为JSON对象，而且还运行健康检查，执行线程转储，并公开有价值的JVM级别和操作系统级别的信息。
- `GraphiteReporter`允许您不断地将指标数据传输到您的Graphite服务器。