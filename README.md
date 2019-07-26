# ros2_performance
## Package goal
Highlight the CPU overhead introduced by using ROS 2 versus Fast RTPS directly. 
Provide data from profiling tools to point where most of the CPU overhead is coming from.
Possibly start a discussion on the ROS 2 middleware design decisions.
## Package summary
This package provides different publisher subscriber setups that generate the same traffic.
ros, rosonenode and rtps all create 20 topics where each topic has 1 publisher and 10 subscribers.
noros and nopub are added for comparison to get an initial idea of overhead generated by different aspects of the ROS 2 stack.

| Binary  | publishers | subscribers | ROS | ROS nodes | ROS timers | DDS participants |
| ------------- | ------------- |------------- |------------- |------------- |------------- |------------- |
| ros | 20  | 200 | yes | 10 | 10 | 10 |
| rosonenode | 20 | 200 | yes | 1 | 1 | 1 |
| nopub | 0  | 0 | yes | 10 | 10 | 10 |
| rtps | 20 | 200 | no | 0 | 0 | 1 |
| noros | 20&ast;  | 200&ast; | no | 0 | 0| 0 |

&ast;C++ implementation no network publishing/subscribing.

Running all examples in isolated docker containers gives the following result:
![Alt text](/images/docker_comparison.png?raw=true "Docker comparison of binaries")

## Recreating the issue using Docker
It is possible to git clone this repository, build the workspace using colcon build and inspecting the CPU usage with top or a similar program for each binary individually. It is however much easier to give each binary its own container (make sure to separate their networks or give them a unique ROS_DOMAIN_ID) and measure the usage of each container.
If you don't have docker and docker compose installed first follow online tutorials on how to install these: https://docs.docker.com/install/ https://docs.docker.com/compose/install/ .

1. Clone this repository
```
git clone https://github.com/scgroot/ros2_performance.git
```
2. cd into the folder 
```
cd ros2_performance
```
3. Build the docker image [requires you to have docker installed properly following the links above] (this will take a while)
```
docker build -t ros2_performance:dashing .
```
*you can name the image differently by replacing "ros2_performance:dashing" by something else, but this will require you to change the docker-compose.yml file accordingly.*

4. Run a compose file
```
cd compose_files
```
here you have 2 options, each folder contains a docker-compose.yml. The "all" folder will start all the binaries listed in the table above. The "pubs_subs" folder will only start "rtps", "ros" and "rosonenode".
```
cd all
```
or
```
cd pubs_subs
```
then 
```
docker-compose up
```
5. Inspect the CPU usage
In a new terminal (ctrl+alt+t) do:
```
docker stats
```
It is possible to specify the format of the output to show more or less information for example:
```
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.PIDs}}"
```
This will only display the container Name, the CPU percentage, Memory usage and PIDS.
You should now see a terminal similar to the image above. 

## Perf results
#### CPU profiling methods
To inspect which functions are using the most CPU there are multiple options. We investigated CPU usage with valgrind http://www.valgrind.org/  (callgrind) and perf https://perf.wiki.kernel.org/index.php/Tutorial. To inspect The data recorded by callgrind it is possible to use kcachegrind http://kcachegrind.sourceforge.net/html/Home.html. The data recorded by perf can be visualized using https://github.com/brendangregg/FlameGraph to generate svg files or alternatively using a GUI with the app Hotspot https://github.com/KDAB/hotspot. The Flamegraph generated by https://github.com/brendangregg/FlameGraph is integrated in the Hotspot app. How to use the Hotspot app is explained in the README.md of the app.

The results of both tools were basically the same, therefore we picked the most readable output to share our findings.
In our opinion the FlameGraphs generated from perf's data are more readable than kcachegrinds callee maps and call-graphs.

#### Measurements performed on this package using perf
We ran each binary for 30 seconds and recorded the amount of CPU cycles and time spent in each function using:
```
source /opt/ros/dashing/setup.bash
cd ros2_performance
colcon build
perf record --call-graph dwarf build/ros2_performance/<binary_name> & sleep 30; killall perf
```
This was done for all three binaries that have publishers and subscribers (rtps, ros, rosonenode). 
The total amount of cycles is of course influenced by how busy the CPU is at the time.
The CPU usage % for each container is measured at the same time so the multiplicate between the containers remains fairly stable even though the absolute number fluctuates due to background processes. Since comparing percentages of percentages gets confusing we rather compare amount of CPU cycles spent in a function to each other directly. To achieve this we measured the amount of CPU cycles for 30 seconds multiple times for each binary and averaged the values, we then calculated the multiplicates between CPU cycles. 

| Binary  | CPU cycles total | CPU usage %| CPU cycle multiplicate | CPU usage % multiplicate
| ------------- | ------------- |------------- |------------- |------------- |
| rtps | ~3.6e9 | ~6 | base | base |
| rosonenode | ~1.3e10 | ~21| ~3.5 | ~3.5 |
| ros | ~3.8e10 | ~63| ~10.6 | ~10.5 |

Because the multiplicates between the amount of CPU cycles and the amount of CPU usage % are so close together we can assume that we can now directly compare the average amount of CPU cycles spent in a function between binaries. We are aware this is a quite crude approach and does not entirely filter out the noise of background processes, but we are certain the observations and conclusions based on this research remain the same. We are not looking to accurately measure if ROS 2 uses 10.5 or 10.6 times more CPU for our system, we are investigating WHY and WHERE ROS 2 is using more CPU.  

The following table shows the amount CPU cycles spent in the two main classes of each binary:

| Binary  |total | eProsima | SingleThreadedExecutor| other |
| ------------- | ------------- |------------- |------------- |------------- |
| rtps | ~3.6e9 | ~3.0e9 |NA | ~6.0e8 |
| rosonenode | ~1.3e10 | ~2.5e9 | ~9.1e9| ~1.4e9|
| ros | ~3.8e10 | ~1.2e10 | ~1.6e10 | ~1.0e10 |

The contribution of each class relative to their respective binary can be seen below:

| Binary  |eProsima| SingleThreadedExecutor| other %|
| ------------- | ------------- |------------- |------------- |
| rtps | ~83.7% |NA | ~16.3% |
| rosonenode| ~19.6% | ~70%| ~10.4%|
| ros | ~32% | ~ 43.9%| ~24.1% |

The distribution in the table above is also represented in the FlameGraph when opened in Hotspot.
The FlameGraph image of rosonenode below is not to scale, but does show all the function calls.

![Alt text](/images/rosonenode.png?raw=true "FlameGraph for rosonenode")

The FlameGraphs of the other binaries can be found in the images folder, to properly see the distributions in Hotspot please generate perf.data files on your local machine. How to do this is explained on the Hotspot github page, a link to this page can be found in this README under **CPU profiling methods**. The relevant command is also shown directly under **Measurements performed on this package using perf**.

## Observations
* The percentual contribution of the SingleThreadedExecutor for the rosonenode binary is very high.
* The amount of CPU cycles for the eProsima part of the rtps binary and rosonenode binary are very similar (3.0e9 vs 2.5e9).
* The amount of CPU cycles for the eprosima part of the ros binary is much higher than for the rosonenode binary (1.2e10 vs 2.5e9).
* The amount of CPU cycles for the SingleThreadedExecutor of the ros binary is much higher than for the rosonenode binary (1.6e10 vs 9.1e9).

## Conclusions
* The SingleThreadedExecutor is using a lot of CPU power. 
* Adding more ROS2 nodes increases both the work performed by the eProsima and SingleThreadedExecutor part of the application. For the SingleThreadedExecutor this makes sense, more nodes means more work for the executor. The increase in work for the eProsima part of the application stems from the 1-to-1 mapping of nodes to DDS participants. 

## Discussion
W.r.t. The SingleThreadedExecutor:

Our current analysis suggests that the SingleThreadedExecutor needs to be optimized otherwise normal ROS 2 cannot work properly on 'ARM A-class' embedded boards. We are willing to look more into this problem by performing more tests and providing feedback to improvements. If you have findings or run into similar problems, please join the ROS Discourse discussion.

ROS Discourse discussion on this topic can be found here:
https://discourse.ros.org/t/singlethreadedexecutor-creates-a-high-cpu-overhead-in-ros-2/10077

W.r.t. 1-to-1 mapping of nodes to DDS participants:

The roadmap mentions "Reconsider 1-to-1 mapping of ROS nodes to DDS participants" https://index.ros.org/doc/ros2/Roadmap/ . We would like to see this happen rather sooner than later. We already observe that this leads to problems in CPU usage and can constrain people in their freedom to design an architecture for a robotic system. The ROS2 middleware should allow for a setting where everything can be grouped into a single DDS participant for the people that want to use nodes for modularity at the top level, but don't want the code fragmented at the bottom level. Many use cases exist where one would like to create multiple nodes that all run on the same hardware. This is especially important since intra-process communication does not work efficiently at the time of writing this file. 

ROS Discourse discussion on this topic can be found here:
https://discourse.ros.org/t/reconsidering-1-to-1-mapping-of-ros-nodes-to-dds-participants/10062

