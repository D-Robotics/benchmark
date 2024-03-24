English| [简体中文](./README_cn.md)

# performance_test

The performance_test tool tests latency and other performance metrics of [various middleware
implementations](#middleware-plugins) that support a pub/sub pattern. It is used to simulate
non-functional performance of your application.

The performance_test tool allows you to quickly set up a pub/sub configuration,
e.g. number of publisher/subscribers, message size, QOS settings, middleware. The following metrics
are automatically recorded when the application is running:

- **latency**: corresponds to the time a message takes to travel from a publisher to subscriber. The
  latency is measured by timestamping the sample when it's published and subtracting the timestamp
  (from the sample) from the measured time when the sample arrives at the subscriber.
- **CPU usage**: percentage of the total system wide CPU usage
- **resident memory**: heap allocations, shared memory segments, stack (used for system's internal
  work).
- **sample statistics**: number of samples received, sent, and lost per experiment run.

## How to use this document

1. Start [here](#example) for a quick example of building and running the performance_test tool
   with the Cyclone DDS plugin.
2. If needed, find more detailed information about [building](#building-the-performance_test-tool)
   and [running](#running-an-experiment)
3. Or, if the quick example is good enough, skip ahead to the [list of supported middleware
   plugins](#middleware-plugins) to learn how to test a specific middleware implementation.
4. Check out the [tools for visualizing the results](#analyze-the-results)
5. If desired, read about the [design and architecture](#architecture) of the tool.

## Example

This example shows how to test the non-functional performance of the following configuration:

| Option                     | Value       |
|----------------------------|-------------|
| Plugin                     | Cyclone DDS |
| Message type               | Array1k     |
| Publishing rate            | 100Hz       |
| Topic name                 | test_topic  |
| Duration of the experiment | 30s         |

1. Install [ROS 2](https://docs.ros.org/en/foxy/index.html)

2. Install [Cyclone DDS](https://github.com/eclipse-cyclonedds/cyclonedds) to /opt/cyclonedds

3. Build performance_test with the [CMake build flag](#eclipse-cyclone-dds) for Cyclone DDS:

```bash
source /opt/ros/foxy/setup.bash
cd ~/perf_test_ws```
colcon build --cmake-args -DPERFORMANCE_TEST_CYCLONEDDS_ENABLED=ON
source ./install/setup.bash
```

4. Run with the [communication plugin option](#eclipse-cyclone-dds) for Cyclone DDS:

```bash
mkdir experiment
./install/performance_test/lib/performance_test/perf_test --communication CycloneDDS
                                                          --msg Array1k
                                                          --rate 100
                                                          --topic test_topic
                                                          --max-runtime 30
                                                          --logfile experiment/log
```

At the end of the experiment, a CSV log file will be generated in the experiment folder with a name
that starts with `log`.

## Building the performance_test tool

For a simple example, see [Dockerfile.rclcpp](dockerfiles/Dockerfile.rclcpp).

The performance_test tool is structured as a ROS 2 package, so `colcon` is used to build it.
Therefore, you must source a ROS 2 installation:

```bash
source /opt/ros/foxy/setup.bash
```

Select a middleware plugin from [this list](#middleware-plugins).
Then build the performance_test tool with the selected middleware:

```bash
mkdir -p ~/perf_test_ws/src
cd ~/perf_test_ws/src
git clone https://gitlab.com/ApexAI/performance_test.git
cd ..
# At this stage, you need to choose which middleware you want to use
# The list of available flags is described in the middleware plugins section
# Square brackets denote optional arguments, like in the Python documentation.
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release <cmake_enable_plugin_flag>
source install/setup.bash
```

## Running an experiment

The performance_test experiments are run through the `perf_test` executable.
To find the available settings, run with `--help` (note the required and default arguments):```bash
~/perf_test_ws$ ./install/performance_test/lib/performance_test/perf_test --help
```

- The `-c` argument should match the selected [middleware plugin](#middleware-plugins)
  from the build phase.
- The `--msg` argument should be one of the supported message types, which can be listed
  by running with `--msg-list`.

### Single machine or distributed system?

Based on the configuration you want to test, the usage of the performance_test tool differs. The
different possibilities are explained below.

For running tests on a single machine, you can choose between the following options:

1. Intraprocess means that the publisher and subscriber threads are in the same process.
   This is the default configuration.
1. Interprocess means that the publisher and subscriber are in different processes. To test
   interprocess communication, two instances of the performance_test must be run, e.g.

    ```bash
    perf_test <options> --num-sub-threads 1 --num-pub-threads 0 &
    sleep 1  # give the subscriber time to finish initializing
    perf_test <options> --num-sub-threads 0 --num-pub-threads 1
    ```

    1. :point_up: CPU and Resident Memory measurements are logged separately for the publisher and
       subscriber processes.
    1. Latency is only logged for the subscriber process, because it is calculated after the
       sample is received.
    1. Some plugins also support zero copy transfer. With zero copy transfer, the publisher
       requests a loan from a pre-allocated shared memory pool, where it writes the sample. The
       subscriber reads the sample from that same location. When running, use the `--zero-copy`
       argument for both the publisher and subscriber processes.
    1. :memo: The transport is dependent on the middleware
    1. It is recommended to start the subscriber process first, and delay for a short amount of time,
       so that there isn't a full queue for it to process right from the start.

On a distributed system, testing latency is difficult, because the clocks are probably not
perfectly synchronized between the two devices. To work around this, the performance_test tool
supports relay mode, which allows for a round-trip style of communication:

```bash
# On the main machine
perf_test <options> --roundtrip-mode Main

# On the relay machine:
perf_test <options> --roundtrip-mode Relay
```## Middleware plugins

### Native plugins

The performance test tool can measure the performance of a variety of communication solutions
from different vendors. In this case there is no [rclcpp or rmw
layer](http://docs.ros2.org/beta2/developer_overview.html#internal-api-architecture-overview)
overhead over the publisher and subscriber routines. The following plugins are currently
implemented:

#### Eclipse Cyclone DDS

- [Eclipse Cyclone DDS 0.8.0beta6](https://github.com/eclipse-cyclonedds/cyclonedds/tree/0.8.0beta6)
- CMake build flag: `-DPERFORMANCE_TEST_CYCLONEDDS_ENABLED=ON`
- Communication plugin: `-c CycloneDDS`
- Zero copy transport (`--zero-copy`): yes
  - Cyclone DDS zero copy requires the
    [runtime switch](https://github.com/eclipse-cyclonedds/cyclonedds/blob/iceoryx/docs/manual/shared_memory.rst)
    to be enabled.
  - When the runtime switch is enabled,
    [RouDi](https://github.com/eclipse-iceoryx/iceoryx/blob/master/doc/website/getting-started/overview.md#roudi)
    must be running.
  - If the runtime switch is enabled, but `--zero-copy` is not added, then the plugin will not use
    the loaned sample API, but iceoryx will still transport the samples.
  - See [Dockerfile.mashup](dockerfiles/Dockerfile.mashup)
- Docker file: [Dockerfile.CycloneDDS](dockerfiles/Dockerfile.CycloneDDS)
- Default transports:
  | INTRA | IPC on same machine | Distributed system |
  |-------|---------------------|--------------------|
  | INTRA | UDP                 | UDP                |


#### Eclipse Cyclone DDS C++ binding

- [Eclipse Cyclone DDS C++ bindings 0.8.1](https://github.com/eclipse-cyclonedds/cyclonedds-cxx/tree/0.8.1)
- CMake build flag: `-DPERFORMANCE_TEST_CYCLONEDDS_CXX_ENABLED=ON`
- Communication plugin: `-c CycloneDDS-CXX`
- Zero copy transport (`--zero-copy`): yes
  - Cyclone DDS zero copy requires the
    [runtime switch](https://github.com/eclipse-cyclonedds/cyclonedds/blob/iceoryx/docs/manual/shared_memory.rst)
    to be enabled.
  - When the runtime switch is enabled,
    [RouDi](https://github.com/eclipse-iceoryx/iceoryx/blob/master/doc/website/getting-started/overview.md#roudi)
    must be running.- If the runtime switch is enabled, but `--zero-copy` is not added, then the plugin will not use
    the loaned sample API, but iceoryx will still transport the samples.
  - See [Dockerfile.mashup](dockerfiles/Dockerfile.mashup)
- Docker file: [Dockerfile.CycloneDDS-CXX](dockerfiles/Dockerfile.CycloneDDS-CXX)
- Default transports:
  | INTRA | IPC on same machine | Distributed system |
  |-------|---------------------|--------------------|
  | INTRA | UDP                 | UDP                |


#### Eclipse iceoryx

- [iceoryx 1.0](https://github.com/eclipse-iceoryx/iceoryx/tree/release_1.0)
- CMake build flag: `-DPERFORMANCE_TEST_ICEORYX_ENABLED=ON`
- Communication plugin: `-c iceoryx`
- Zero copy transport (`--zero-copy`): yes
- Docker file: [Dockerfile.iceoryx](dockerfiles/Dockerfile.iceoryx)
- The iceoryx plugin is not a DDS implementation.
  - The DDS-specific options (such as domain ID, durability, and reliability) do not apply.
- To run with the iceoryx plugin,
  [RouDi](https://github.com/eclipse-iceoryx/iceoryx/blob/master/doc/website/getting-started/overview.md#roudi)
  must be running.
- Default transports:
  | INTRA     | IPC on same machine | Distributed system                |
  |-----------|---------------------|-----------------------------------|
  | zero copy | zero copy           | Not supported by performance_test |

#### eProsima Fast DDS

- [FastDDS 2.0.x](https://github.com/eProsima/Fast-RTPS/tree/2.0.x)
- CMake build flag: `-DPERFORMANCE_TEST_FASTRTPS_ENABLED=ON`
- Communication plugin: `-c FastRTPS`
- Zero copy transport (`--zero-copy`): no
- Docker file: [Dockerfile.FastDDS](dockerfiles/Dockerfile.FastDDS)
- Default transports:
  | INTRA | IPC on same machine | Distributed system |
  |-------|---------------------|--------------------|
  | UDP   | UDP                 | UDP                |

#### OCI OpenDDS

- [OpenDDS 3.13.2](https://github.com/objectcomputing/OpenDDS/tree/DDS-3.13.2)
- CMake build flag: `-DPERFORMANCE_TEST_OPENDDS_ENABLED=ON`
- Communication plugin: `-c OpenDDS`
- Zero copy transport (`--zero-copy`): no
- Docker file: [Dockerfile.OpenDDS](dockerfiles/Dockerfile.OpenDDS)
- Default transports:
  | INTRA | IPC on same machine | Distributed system |
  |-------|---------------------|--------------------|
  | TCP   | TCP                 | TCP                |#### RTI Connext DDS

- [RTI Connext DDS 5.3.1+](https://www.rti.com/products/connext-dds-professional)
- CMake build flag: `-DPERFORMANCE_TEST_CONNEXTDDS_ENABLED=ON`
- Communication plugin: `-c ConnextDDS`
- Zero copy transport (`--zero-copy`): no
- Docker file: Not available
- A license is required
- You need to source an RTI Connext DDS environment.
  - If RTI Connext DDS was installed with ROS 2 (Linux only):
    - `source /opt/rti.com/rti_connext_dds-5.3.1/setenv_ros2rti.bash`
  - If RTI Connext DDS is installed separately, you can source the following script to set the
    environment:
    - `source <connextdds_install_path>/resource/scripts/rtisetenv_<arch>.bash`
- Default transports:
  | INTRA | IPC on same machine | Distributed system |
  |-------|---------------------|--------------------|
  | INTRA | SHMEM               | UDP                |

#### RTI Connext DDS Micro

- [Connext DDS Micro 3.0.3](https://www.rti.com/products/connext-dds-micro)
- CMake build flag: `-DPERFORMANCE_TEST_CONNEXTDDSMICRO_ENABLED=ON`
- Communication plugin: `-c ConnextDDSMicro`
- Zero copy transport (`--zero-copy`): no
- Docker file: Not available
- A license is required
- Default transports:
  | INTRA | IPC on same machine | Distributed system |
  |-------|---------------------|--------------------|
  | INTRA | SHMEM               | UDP                |

### ROS 2 Middleware plugins

The performance test tool can also measure the performance of a variety of RMW implementations,
through the ROS2 `rclcpp::publisher` and `rclcpp::subscriber` API.

- [ROS 2 `rclcpp::publisher` and `rclcpp::subscriber`](https://docs.ros.org/en/foxy/Tutorials/Writing-A-Simple-Cpp-Publisher-And-Subscriber.html)
- CMake build flag: `-DPERFORMANCE_TEST_RCLCPP_ENABLED=ON` (on by default)
- Communication plugin:
  - Callback with Single Threaded Executor: `-c rclcpp-single-threaded-executor`
  - [`rclcpp::WaitSet`](https://github.com/ros2/rclcpp/pull/1047): `-c rclcpp-waitset`
- Zero copy transport (`--zero-copy`): yes
- Docker file: [Dockerfile.rclcpp](dockerfiles/Dockerfile.rclcpp)
- These plugins will use the ROS 2 RMW implementation that is configured on your system.
  - ROS 2 Foxy is pre-configured to use rmw_fastrtps_cpp.
    - Follow [these instructions](https://docs.ros.org/en/foxy/Guides/Working-with-multiple-RMW-implementations.html)
    to use a different RMW implementation with ROS 2.
    - You can find a list of several other middleware options[在这里](https://docs.ros.org/en/foxy/Concepts/About-Different-Middleware-Vendors.html).
- 默认传输方式：取决于底层DDS实现

## 分析结果

运行带有`-l`标志的实验后，将记录一个CSV文件。可以通过在运行实验之前设置`APEX_PERFORMANCE_TEST`环境变量来向CSV文件添加自定义数据，例如

```json
# JSON格式
export APEX_PERFORMANCE_TEST="
{
\"My Version\": \"1.0.4\",
\"My Image Version\": \"5.2\",
\"My OS Version\": \"Ubuntu 16.04\"
}
"
```

### 绘制结果

performance_test工具提供了几种工具来绘制生成的结果：

1. 结果渲染在PDF文件中：方便分享结果
    <img src="plotter_generated_pdf.png"  width="1000">
1. 结果渲染在Jupyter笔记本中：用于比较多个实验
    <img src="performance_test/helper_scripts/apex_performance_plotter/example_plot_two_experiments.png"  width="1000">

绘图工具需要python3和texlive。在Ubuntu系统上，您需要安装以下软件包：

`sudo apt-get install python3 python3-pip texlive texlive-pictures texlive-luatex`

启动Python虚拟环境并安装所需的Python包：

```bash
cd performance_test/helper_scripts/apex_performance_plotter
pip3 install pipenv
pipenv shell
pipenv install --ignore-pipefile
```

#### 用法

要从日志文件生成PDF，请在上一步安装的`perfplot`二进制文件中调用：

`perfplot <filename1> <filename2> ...`

另外，请务必查看`perfplot -h`以获取额外选项。:point_up: **Common Pitfalls**

所有延迟指标都由订阅者进程收集和计算。
对于进程间通信，建议为日志文件提供不同的前缀：

```bash
perf_test -c rclcpp-single-threaded-executor --msg Array1k -p 0 -s 1 -l log_sub
perf_test -c rclcpp-single-threaded-executor --msg Array1k -p 1 -s 0 -l log_pub
```

然后，要绘制延迟指标，请在订阅者的日志文件上调用perfplot。
如果在发布者的日志文件上调用perfplot，则会绘制CPU和内存指标，但延迟图表将为空。

To analyze the results in a Jupyter notebook run the following commands:

```bash
pipenv shell
jupyter notebook plot_logs.ipynb

# When you are done, deactivate the venv
deactivate
```

## Architecture

Apex.AI的《ROS 2中的性能测试》白皮书
([可在此处获取](https://drive.google.com/file/d/15nX80RK6aS8abZvQAOnMNUEgh7px9V5S/view))
描述了如何设计一个公平且无偏的性能测试，这也是这个项目的基础。
<center><img src="architecture.png"></center>

## Future extensions and limitations

- 像DDS这样的通信框架有大量的设置。该工具仅允许配置最常见的QoS设置。其他QoS设置已在应用程序中硬编码。
- 每个话题只允许一个发布者，因为数据验证逻辑不支持将数据匹配到不同的发布者。
- 一些通信插件在收到过多数据时可能会陷入内部循环。解决此类问题是该工具的目标之一。
- FastRTPS等待集不支持超时，这可能导致接收端不中止。在这种情况下，必须手动终止性能测试。
- 使用`reliable` QoS和历史种类设置为`keep_all`的Connext DDS Micro INTRA传输与Connext Micro不兼容。在使用`reliable`时，始终将`keep-last`作为QoS历史种类。
请把以下内容中出现中文的部分翻译成英文，保留原有的格式和内容：Possible additional communication which could be implemented are:

- Raw UDP communication

## Batch run experiments (for advanced users)

The [`performance_report`](performance_report) package runs multiple performance_test experiments
at once, plots the results, and generates reports based on the results.