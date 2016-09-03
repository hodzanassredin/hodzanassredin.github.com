--- 
published: true 
layout: post 
title: Monitoring suave.io app with Datadog 
tags : [datadog, suaveio, fsharp] 
--- 
 
 
## {{page.title}} 
 
 
 
 
<p class="meta">03 September 2016 &#8211; Karelia</p> 
 

At work, we have a lot of infrastructure and we have to know what is going on with our servers and apps.
There are different options to do that and one of them is [Datadog](https://www.datadoghq.com/) service.
Datadog can handle all stats from your servers and show you nice graphs and send you alerts in case of a problem. 
To use it you have to create an account and install datadog agent service on your server.
Today I decided to write an suave.io web site and decided to connect it to our datadog account to show some info
about request rates and so on. There are different options how to connect your app to your datadog account.
For example you could write datadog custom check, but it is easier to use existing one.
We use golang expvars checks for our golang servers, so I decided to emulate them in suave. 


# What is expvars? 
Expvars is a golang module which allows you to create some variables and expose them via web interface in json format.
You could find more info [here](https://golang.org/pkg/expvar/). This is an example of expvars output. There are some predefined expvars in golang, so you could check memory stats and so on.
{% highlight json %} 
{
"cmdline": ["/xxx"],
"err_checks-rps": 20,
"memstats": {"Alloc":408478640,"a lot of other mem stats" : "tons of stats"},
"succ_checks-rps": 3471,
}
{% endhighlight %} 

By convention address of this json is "IP:8000/debug/vars".

Let’s implement one expvar in a suave app.
First of all, you have to create a datadog account and after that install datadog agent as described [here](http://docs.datadoghq.com/guides/basic_agent_usage/).
In our case this is an ubuntu server. After installation we have to create "/etc/dd-agent/conf.d/go_expvar.yaml" file.

{% highlight yaml %} 
init_config:

instances:
  - expvar_url: http://localhost:8000/debug/vars
    metrics:
       - path: hellorps
{% endhighlight %} 
Actually we also need to comment this line in  "/etc/dd-agent/checks.d/go_expvar.py": 
{% highlight python %} 
#self.get_gc_collection_histogram(data, tags, url, namespace)
{% endhighlight %} 
This line will throw in case of missing memstats data.
Now restart the datadog agent "/etc/init.d/datadog-agent restart")
More about how to setup datadog with expvars you could find [here](https://www.datadoghq.com/blog/instrument-go-apps-expvar-datadog/)
Last step is to create suave.io app with “hellorps” expvar and run it.
We will use suave script template

{% highlight fsharp %} 
open System
open System.IO

Environment.CurrentDirectory <- __SOURCE_DIRECTORY__
 
if (File.Exists "paket.exe") then
    File.Delete "paket.exe"
let url = "https://github.com/fsprojects/Paket/releases/download/3.18.2/paket.exe"
use wc = new Net.WebClient()
let tmp = Path.GetTempFileName()
wc.DownloadFile(url, tmp)
File.Move(tmp,Path.GetFileName url);;
 
// Step 1. Resolve and install the packages
 
#r "paket.exe"
 
Paket.Dependencies.Install """
source https://nuget.org/api/v2
nuget Suave
""";;
 
// Step 2. Use the packages
 
#r "packages/Suave/lib/net40/Suave.dll"
#r "System.Runtime.Serialization"
open Suave // always open suave
open Suave.Filters
open Suave.Operators
open Suave.Successful
open Suave.Json
open System.Runtime.Serialization
open System.Net

{% endhighlight %} 
Now we have to add a thread safe rate counter.
{% highlight fsharp %} 
type RateCounter() = 
    let mutable lastTick = System.Environment.TickCount
    let mutable lastFrameRate = 0
    let mutable frameRate = 0
    let lockObj = new obj()
    let synchronized f v = lock lockObj (fun () -> f v)
    let inc v = 
            if System.Environment.TickCount - lastTick >= 1000 
            then
                lastFrameRate <- frameRate
                frameRate <- v
                lastTick <- System.Environment.TickCount
            else
                frameRate <- frameRate + v
            lastFrameRate
    

    member x.Inc v = synchronized inc v |> ignore
    member x.Get () = synchronized inc 0

let helloCounter = RateCounter() 
{% endhighlight %} 
Create hello webpart and start server.
{% highlight fsharp %} 
let hello (x : HttpContext) =
      helloCounter.Inc 1 
      OK "Hello World!" x

let _, server = startWebServerAsync defaultConfig hello
{% endhighlight %} 
And do the same for expvars.
{% highlight fsharp %} 
let expvarsConfig = {defaultConfig with bindings = 
            [ HttpBinding.mkSimple  Protocol.HTTP "0.0.0.0" 8000]}

let _, expvarsServer = startWebServerAsync expvarsConfig (path "/debug/vars" 
                        >=> fun ctx -> (OK (sprintf "{\"hellorps\":%d}" 
                        <| helloCounter.Get() ) ctx))

{% endhighlight %} 
create load generator.
{% highlight fsharp %} 
let client = new WebClient ()
let rec gen_load sleepMs = async{
    let! _ = client.AsyncDownloadString(new Uri("http://127.0.0.1:8083/"))
    do! Async.Sleep(sleepMs)
    return! gen_load(sleepMs) 
}
{% endhighlight %} 
And finally run everything
{% highlight fsharp %} 
Async.Parallel [server; expvarsServer; gen_load(0)] 
    |> Async.RunSynchronously 
    |> ignore
{% endhighlight %} 

Done. Now we can go to datadog and see our metric.

![Datadog Metric]({{ site.url }}/images/datadogexp.png )

[Full code](https://gist.github.com/hodzanassredin/cdb0c1ff5b11b81fd8e5bd0fa8d975a8)


