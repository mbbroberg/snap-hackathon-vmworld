# Steps to get up and running for this hackathon

VMworld 2016 has a hosted hackathon that I offered four options for:

1. Beginner Ops: Creating a Telemetry Playground VM where we gather Linux metrics with Snap and visualize it in Grafana
2. Beginner Dev: Configure your dev environment to write Go code, the language Snap is written in, and build from source
3. Advance Ops: Deploy multiple VMs and run multiple Snap instances to push centrally to a Grafana instance
4. Advance Dev: Write your Go code to collect metrics for ANY system you want using the plugin model of Snap

The **1st option** was mosted voted for before the event! Here's how to do it.


The TL;DR we all need to get to here:
* Grafana UI is accessible at localhost:3000
* Snap API is available via localhost:8181


## Step by step

1. Download Snap for your OS: http://snap-telemetry.io/download

1. Download the plugin packages we want to play with: http://snap-telemetry.io/plugins.html
I will focus on (NOTE: you don't need to download their source code to use them, but it's helpful to review the docs here):
  - psutil: https://github.com/intelsdi-x/snap-plugin-collector-psutil
  - exec: https://github.com/intelsdi-x/snap-plugin-collector-exec
You can download exec, and other plugin binaries, from our CI: 
```
$ curl -sSO http://snap.ci.snap-telemetry.io/plugin/build/latest/snap-plugin-collector-exec
```
1. Download Docker. I used Docker for my influxdb and grafana instances. If you have an alternative way you would like to store your data or utilize grafana, go for it, Snap will work regardless.
1. Going off of the Docker route, set up two containers. One for influxdb and one for grafana. [Here](https://docs.docker.com/engine/reference/commandline/cli/) is the documentation for the docker cli if you would like to learn more.

```
# Run InfluxDB with persistence
export WHEREYOUWANT=/tmp/
docker run -d --volume=$WHEREYOUWANT:/data -p 8083:8083 -p 8086:8086 --name influxdb tutum/influxdb

# create /var/lib/grafana as persistent volume storage
docker run -d -v /var/lib/grafana --name grafana-storage busybox:latest

# start grafana
docker run \
  -d \
  -p 3000:3000 \
  -p 8181:8181 \
  --link influxdb \
  --name grafana \
  -e "GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource" \
  --volumes-from grafana-storage \
  grafana/grafana
```
1. Next, navigate to your InfluxDB database (localhost:8083) and make a new database with a name you like (mine is called playground).
1. Start Snap with `snapd -t 0`
1. Open a new terminal and load the plugins you want to use. Below are the plugins I used to make my dashboard:
```
snapctl plugin load snap-plugin-collector-cpu  
snapctl plugin load snap-plugin-collector-cgroups  
snapctl plugin load snap-plugin-collector-psutil  
snapctl plugin load snap-plugin-publisher-influxdb  
```
8. Now is time to write your Task Manifest file. I've included mine below:
``` 
{
    "version": 1,
    "schedule": {
        "type": "simple",
        "interval": "1s"
    },
    "workflow": {
        "collect": {
            "metrics": {
                "/intel/procfs/cpu/all/user_jiffies" : {},
                "/intel/procfs/cpu/all/nice_jiffies" : {},
                "/intel/procfs/cpu/all/system_jiffies" : {},
                "/intel/procfs/cpu/all/idle_jiffies" : {},
                "/intel/procfs/cpu/all/iowait_jiffies" : {},
                "/intel/procfs/cpu/all/irq_jiffies" : {},
                "/intel/procfs/cpu/all/softirq_jiffies" : {},
                "/intel/procfs/cpu/all/steal_jiffies" : {},
                "/intel/procfs/cpu/all/guest_jiffies" : {},
                "/intel/procfs/cpu/all/guest_nice_jiffies" : {},

               "/intel/linux/cgroups/cpuacct_stats/*/cpu_usage/total_usage" : {},
               "/intel/linux/cgroups/cpuacct_stats/*/cpu_usage/usage_in_kernelmode" : {},
               "/intel/linux/cgroups/cpuacct_stats/*/cpu_usage/usage_in_usermode" : {},
               "/intel/linux/cgroups/cpu_stats/*/cpu_usage/total_usage" : {},
               "/intel/linux/cgroups/cpu_stats/*/throttling_data/throttled_time": {},

                "/intel/psutil/load/load1": {},
                "/intel/psutil/load/load5": {},
                "/intel/psutil/load/load15": {},
                "/intel/psutil/vm/available": {},
                "/intel/psutil/vm/free": {},
                "/intel/psutil/vm/used": {},
                "/intel/psutil/vm/used_percent": {},  
                "/intel/psutil/vm/inactive": {},  
                "/intel/psutil/vm/cached": {},  
                "/intel/psutil/vm/buffers": {},  
                "/intel/psutil/vm/active": {},  
                "/intel/psutil/net/all/packets_recv": {},  
                "/intel/psutil/net/all/packets_sent": {} 
            },      
            "process": null,
            "publish": [
                {
                  "plugin_name": "influx",
                  "config": {
                  "host": "localhost",
                  "port": 8086,
                  "database": "playground",
                  "user": "admin",
                  "password" : "admin"
                        }
                    }
                ]
          }
     }
}
```
And there are other examples in [task-manifest/](task-manifest/) For the full syntax, see [the documentation](https://github.com/intelsdi-x/snap/blob/master/docs/TASKS.md)
9. Start the task using `snapctl task create -t <taskname>.json`. You can find your task id with `snapctl task list`, and watch it with, `snapctl task watch <task id>`.
10. From here you can navigate to your Grafana container at localhost:3000. The first thing you need to do is to add your datasource. After you have your data source added, you can start creating a dashboard. 



