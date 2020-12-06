# Tuning with Parameters



我们通过在读操作的时候包含了一个`quorm`参数，允许我们上一个测试通过，但是为了看到原始的脏读的bug，我们不得不再次编辑源码，设置标志为`false`。如果我们能从命令行调整该参数，那将会比较好。jepsen提供了一些默认的命令行选项[jepsen.cli](https://github.com/jepsen-io/jepsen/blob/0.1.7/jepsen/src/jepsen/cli.clj#L52-L87),但是我们可以通过`:opt-spec`给`cli/single-test-cmd`添加我们自己的选项。

```clj
(def cli-opts
  "Additional command line options."
    [["-q" "--quorum" "Use quorum reads, instead of reading from any primary."]])
```

CLI选项是一个vector集合，给定一个简短的名称，一个全名，一个文档描述，并且选项影响着如何解析它们的默认值等。这些信息将传递给[tools.cli](https://github.com/clojure/tools.cli)，标准的处理option的clojure库

现在，让我们那个选项规范给传递CLI。

```clj
(defn -main
  "Handles command line arguments. Can either run a test, or a web server for
  browsing results."
  [& args]
  (cli/run! (merge (cli/single-test-cmd {:test-fn  etcd-test
                                         :opt-spec cli-opts})
                   (cli/serve-cmd))
            args)
```


如果我们再次通过`lein run test -q ...`运行我们的测试,我们将在我们的测试map中看到一个新的`:quorum`选项

```clj
10:02:42.532 [main] INFO  jepsen.cli - Test options:
 {:concurrency 10,
 :test-count 1,
 :time-limit 30,
 :quorum true,
 ...
```

Jepsen解析我们的`-q`选项，发现该选项是我们提供的，并且添加`:quorum true`键值对到选项map中，该选项map会传给`etcd-test`，`etcd-test`将会`merge(合并)`选项map到测试 map中。Viola! 我们有了一个`:quorum` 的key在我们的测试中。

现在，让我们使用quorum选项来控制是否客户端达到法定人数读，在客户端的`invoke`函数如下：

```clj
        (case (:f op)
          :read (let [value (-> conn
                                (v/get k {:quorum? (:quorum test)})
                                parse-long)]
                  (assoc op :type :ok, :value (independent/tuple k value)))
```


让我们尝试用携带 `-q` 和不携带 `-q` 参数执行`lein run`，我们将会观察到是否它能让我们再次看到脏读的bug。

```bash
$ lein run test -q ...
...

$ lein run test ...
...
clojure.lang.ExceptionInfo: throw+: {:errorCode 209, :message "Invalid field", :cause "invalid value for \"quorum\"", :index 0, :status 400}
...
```

Huh。让我们双重检查在测试map中`:quorum`的值时什么，它在每个jepsen开始运行时，会打印在日志中

```clj
2018-02-04 09:53:24,867{GMT}	INFO	[jepsen test runner] jepsen.core: Running test:
 {:concurrency 10,
 :db
 #object[jepsen.etcdemo$db$reify__4946 0x15a8bbe5 "jepsen.etcdemo$db$reify__4946@15a8bbe5"],
 :name "etcd",
 :start-time
 #object[org.joda.time.DateTime 0x54a5799f "2018-02-04T09:53:24.000-06:00"],
 :net
 #object[jepsen.net$reify__3493 0x2a2b3aff "jepsen.net$reify__3493@2a2b3aff"],
 :client {:conn nil},
 :barrier
 #object[java.util.concurrent.CyclicBarrier 0x6987b74e "java.util.concurrent.CyclicBarrier@6987b74e"],
 :ssh
 {:username "root",
  :password "root",
  :strict-host-key-checking false,
  :private-key-path nil},
 :checker
 #object[jepsen.checker$compose$reify__3220 0x71098fb3 "jepsen.checker$compose$reify__3220@71098fb3"],
 :nemesis
 #object[jepsen.nemesis$partitioner$reify__3601 0x47c15468 "jepsen.nemesis$partitioner$reify__3601@47c15468"],
 :active-histories #<Atom@18bf1bad: #{}>,
 :nodes ["n1" "n2" "n3" "n4" "n5"],
 :test-count 1,
 :generator
 #object[jepsen.generator$time_limit$reify__1996 0x483fe83a "jepsen.generator$time_limit$reify__1996@483fe83a"],
 :os
 #object[jepsen.os.debian$reify__2908 0x8aa1562 "jepsen.os.debian$reify__2908@8aa1562"],
 :time-limit 30,
 :model {:value nil}}
```

真奇怪，上面没有打印出 `:quorum`这个key，如果选项标志出现在命令行中，则他们只会出现在选项map中；如果他们排除在命令行之外，则他们也排除在选项map外。当我们访问 `(:quorum test)`时， `test` 没有 `:quorum`选项，我们将会得到`nil`。有一些简单的方式来修复这个问题。,在客户端或者在 `etcd-test`中，我们可以强迫`nil` 为 `false`通过使用`(boolean (:quorum test))`。或者我们可以强迫在该选项省略时，为该选项通过添加`:default false`指定一个默认值。我们将使用`boolean`在`etcd-test`。以防有人直接调用它，而不是通过CLI。

```clj
(defn etcd-test
  "Given an options map from the command line runner (e.g. :nodes, :ssh,
  :concurrency ...), constructs a test map. Special options:

      :quorum     Whether to use quorum reads"
  [opts]
  (let [quorum (boolean (:quorum opts))]
    (merge tests/noop-test
           opts
           {:pure-generators true
            :name            (str "etcd q=" quorum)
            :quorum          quorum

            ...
```

为了在两个地方我们可以使用quorum的布尔值，我们绑定`quorum`到一个变量上。我们添加它到测试的`名称`上，这将会让人很容易一看就知道哪个测试使用了quorum读。我们也添加它到`:quorum`选项上。因为我们合并`opts`之前，我们的`:quorum`的布尔版本将优先于`opts`中的变量。现在，不需要`-q`，我们的test将会发现如下错误。

```bash
$ lein run test --time-limit 60 --concurrency 100 -q
...
Everything looks good! ヽ(‘ー`)ノ

$ lein run test --time-limit 60 --concurrency 100
...
Analysis invalid! (ﾉಥ益ಥ）ﾉ ┻━┻
```

## Tunable difficulty

你将会注意到一些测试卡在痛苦缓慢的分析上，这依赖于你的计算机性能。很难预先控制这个测试难度，就像`~n!`，这儿的n表示并发数。大量的crash的进程会使得秒和天的检查差异。

为了帮助解决这个问题，让我们在我们的测试中添加一些可调度的选项，这些选项可以控制你在单个key上执行的操作数，以及生成操作的快慢。

在generator中，让我们将我们的硬编码1/10 延时 变成一个参数，通过每秒的速率来给定。同时将每个key的generator上硬编码的limit也变成一个可配置的参数。

```clj
(defn etcd-test
  "Given an options map from the command line runner (e.g. :nodes, :ssh,
  :concurrency ...), constructs a test map."
  [opts]
  (let [quorum (boolean (:quorum opts))]
    (merge tests/noop-test
           opts
           {:pure-generators true
            :name            (str "etcd q=" quorum)
            :quorum          quorum
            :os              debian/os
            :db              (db "v3.1.5")
            :client          (Client. nil)
            :nemesis         (nemesis/partition-random-halves)
            :checker         (checker/compose
                               {:perf   (checker/perf)
                                :indep (independent/checker
                                         (checker/compose
                                           {:linear   (checker/linearizable
                                                        {:model (model/cas-register)
                                                         :algorithm :linear})
                                            :timeline (timeline/html)}))})
            :generator       (->> (independent/concurrent-generator
                                    10
                                    (range)
                                    (fn [k]
                                      (->> (gen/mix [r w cas])
                                           (gen/stagger (/ (:rate opts)))
                                           (gen/limit (:ops-per-key opts)))))
                                  (gen/nemesis
                                    (->> [(gen/sleep 5)
                                          {:type :info, :f :start}
                                          (gen/sleep 5)
                                          {:type :info, :f :stop}]
                                         cycle))
                                  (gen/time-limit (:time-limit opts)))})))
```

同时添加响应的命令行选项

```clj
(def cli-opts
  "Additional command line options."
  [["-q" "--quorum" "Use quorum reads, instead of reading from any primary."]
   ["-r" "--rate HZ" "Approximate number of requests per second, per thread."
    :default  10
    :parse-fn read-string
    :validate [#(and (number? %) (pos? %)) "Must be a positive number"]]
   [nil "--ops-per-key NUM" "Maximum number of operations on any given key."
    :default  100
    :parse-fn parse-long
    :validate [pos? "Must be a positive integer."]]])
```

我们没必要为每个选项参数都提供一个简短的名称：我们使用`nil`来表明`--ops-per-key`没有缩写。每个标志后面的大写首字母(例如：“HZ” & "NUM")是你要传递的值的任意占位符。他们将会作为使用文档的一部分被打印。我们为这两选项都提供了`:default`，如果没有通过命令行指定，它的默认值将会被使用。对rates而言，我们希望允许一个整数，浮点数和小数，因此...，我们将使用Clojure内置的`read-string`函数来解析上述三类。然后我们将校验它是一个正整数。以阻止人们传递字符串，负数，0等。

现在，如果我们想运行一个小的测试，我们可以执行如下命令。

```bash
$ lein run test --time-limit 10 --concurrency 10 --ops-per-key 10 -r 1
...
Everything looks good! ヽ(‘ー`)ノ
```

浏览每个key的历史记录，我们可以看到操作处理的很慢，同时每个键只有10个，这个测试更容易检查。然而，它也不能发现bug！这是jepsen中的原有的tension：
我们必须积极的发现错误，但是积极地验证这些历史记录更加困难，---甚至的不可能的事情。

线性一直读检查时NP-hard；现在还没有办法能够解决。我们会设计一些更有效的检查器，但是最终，指数级的困难将使我们寸步难行。或许，我们可以验证一个 *weaker(稍弱)*的属性，线性或者对数时间。让我们[add a commutative test](08-set.md)