## 谈谈caddy v2的设计

caddy在大概19年的时候做了一次比较大版本的升级，这次升级从功能上讲没做太大的变更，主要还是在架构设计上做了较大的调整，从官方的features上来看，主要表现在如下几点：
* Production-ready after serving trillions of requests and managing millions of TLS certificates
* Scales to tens of thousands of sites ... and probably more
* Highly extensible modular architecture lets Caddy do anything without bloat


在caddy1.x时代，一直给人的感觉（官方也自己承认）是性能一般，功能性有待补充。当然和成熟的nginx相比，确实还是个弟弟，只不过对于golang技术栈的人来说是友好的。到了v2版本，主动提了性能和扩展性，这篇文章就主要从扩展性的角度来看看他是如何设计的。性能方面，没做过压测先不讲了。



### 整体结构

caddy主要由三部分组成：command（cmd）、core library和modules。

cmd（github.com/caddyserver/caddy/v2/cmd）主要负责交互，提供了一系列操作命令（start/stop/run/reload/upgrade等），这是非常明确的使用界面。用法也很简单：

```
caddy run 
```

这里有个比较好的设计就是，除了上面说的一系列面向server行为的操作命令外，caddy针对cmd做了扩展，可以支持增加module（支持subcommands）后，也直接支持外部的交互：

```
caddy file-server //这里的file-server就是一个module
```

内部的实现也比较简单，增加一个​cmd的注册接口即可：

```golang
//省略了一些影响不大的内容
//github.com/caddyserver/caddy/v2/modules/caddyhttp/fileserver
caddycmd.RegisterCommand(caddycmd.Command{
    Name:  "file-server",
    Func:  cmdFileServer,
    Usage: "[--domain <example.com>] [--root <path>] [--listen <addr>] [--browse] [--access-log]",
    Short: "Spins up a production-ready file server",
    Long: `
A simple but production-ready file server. Useful for quick deployments,
demos, and development.`,
    Flags: func() *flag.FlagSet {
      fs := flag.NewFlagSet("file-server", flag.ExitOnError)
      fs.String("domain", "", "Domain name at which to serve the files")
      fs.String("root", "", "The path to the root of the site")
      fs.String("listen", "", "The address to which to bind the listener")
      fs.Bool("browse", false, "Enable directory browsing")
      fs.Bool("templates", false, "Enable template rendering")
      fs.Bool("access-log", false, "Enable the access log")
      return fs
    }(),
  })
}
```

回过头来看这个设计，这里非常好的体现了开闭原则，可以再不修改一行原有代码的情况下，直接​扩展出新的使用界面。所以这个预埋的扩展点非常好的体现了开放性。从实现上讲，就是一个非常简单的cmd管理模块，​通过register的方式进行注册和使用。当然这边还有很关键的一点是，它和module绑定到了一起，真正的实现​模块的内聚，这个后面再说。

core部分。不知道一眼看到这个词，你会想到什么？说实话在一个web server的环境下，我看到core，我会认为这边要处理的应该是server的行为（satrt/run/stop这些）或者是核心流程的处理。但caddy在core library这部分定义的是对config的处理（primarily manages configuration），不过这里会提供一个hot-loading的方式，支持运行时的配置变更。所以说config也跟server的运行时行为产生了关系。

```golang
// Run runs the given config, replacing any existing config.
func Run(cfg *Config) error {
  cfgJSON, err := json.Marshal(cfg)
  if err != nil {
    return err
  }
  return Load(cfgJSON, true)
}
​
func Load(cfgJSON []byte, forceReload bool) error{
  //.........
  // load this new config; if it fails, we need to revert to
  // our old representation of caddy's actual config
  err = unsyncedDecodeAndRun(newCfg, true)
  //.........
}
```

这里就是很醉人的地方了，通过config的行为来控制server的行为，好像和我们主观意识上有点不太一样，可能我自己设计的时候，更愿意把server作为一个主体，来管理config的变化，其实在caddy1.x的时候也是这样，只不过这样的代码会复杂一些。这个时候再回过头去看看cmd的设计，你也能发现，其实在cmd控制server行为的具体实现里，它也是先控制了config的行为。前后的设计是按照统一理念的，所以这么设计没有问题。这里也告诉我们，不要太主观的去看待别人的设计，强加自己的意志，会对你理解别人的代码带来非常多的干扰。可以看看cmd的方法，验证下：

```golang
func cmdRun(fl Flags) (int, error) {
  caddy.TrapSignals()
​
  runCmdConfigFlag := fl.String("config")
  runCmdConfigAdapterFlag := fl.String("adapter")
  runCmdResumeFlag := fl.Bool("resume")
  runCmdLoadEnvfileFlag := fl.String("envfile")
  runCmdPrintEnvFlag := fl.Bool("environ")
  runCmdWatchFlag := fl.Bool("watch")
  runCmdPidfileFlag := fl.String("pidfile")
  runCmdPingbackFlag := fl.String("pingback")
  //.........
   var configFile string
  if !runCmdResumeFlag {
    config, configFile, err = loadConfig(runCmdConfigFlag, runCmdConfigAdapterFlag)
    if err != nil {
      return caddy.ExitCodeFailedStartup, err
    }
  }
​
  // run the initial config
  err = caddy.Load(config, true)
  if err != nil {
    return caddy.ExitCodeFailedStartup, fmt.Errorf("loading initial config: %v", err)
  }
  caddy.Log().Info("serving initial configuration")
   //.........
}
```

最后看下module部分。
```
Modules do everything else
```

其实caddy里面也有module和plugin的概念（Sometimes the terms module, plugin, and extension get used interchangably, and usually that's OK. Technically, all modules are plugins, but not all plugins are modules.），这里先不引入了，感兴趣的可以自己去了解。每个设计者对这两者的理念不一致，所使用的场景就不太一样。像我自己喜欢把这两个分开来用，不做关联。



caddy的module分为host module（or parent module）和guest mudole(or child module)。module有自己的生命周期：

1、Loaded

2、Provisioned and validated

3、Used

4、Cleaned up

相较于1.x的版本，对生命周期的管理更加的规范，增加了Provisioned的处理流程，load的过程其实就是个自我注册的过程：

```golang
func init() {
  weakrand.Seed(time.Now().UnixNano())
  caddy.RegisterModule(FileServer{})
}
```
注册哪几个模块，可以在配置文件中描述出来，caddy.json：

```golang
{
   "apps": {
      "http": {
         "servers": {
            "hello": {
               "listen": [":2015"],
               "routes": [
                  {
                     "handle": [{
                        "handler": "static_response",
                        "body": "Hello, world!"
                     }]
                  }
               ]
            }
         }
      }
   }
}
```

其中，routes类似我们常用的pipeline，caddy是在配置中route好一个request的路径。所以看到这边也比较容易理解，模块化的设计实现能力的单一性，通过组合的方式进行扩展。也充分体现了设计模式中，最基础的几个原则。关于module的扩展和规范可以看：https://caddyserver.com/docs/extending-caddy。


讲到这边，我们可以看到caddy的设计尊从了最基本的设计原则，在实现上也采用了一些基本的概念和操作。只是从代码的维度看，这里面的关系会有些绕。可能接触golang不久的人需要多看几遍。特别是新增的App、admin可以重点看一下，这部分也是比较直接的对1.x设计的变更。看代码的路径可以从cmd/config(admin)/app/server/module。


感兴趣可以再看下：https://www.youtube.com/watch?v=EhJO8giOqQs

```
In Caddy 2, we adapt several features of the Go language to design a modular program that can drastically shift its configuration on-the-fly and clean up after itself. In this talk, we discuss how Caddy performs graceful reloads, reduces global state, and improves performance ...
```