# 使用systemctl服务管理

------

## 前言

使用systemctl命令就可以进行服务的管理，再也不需要进行项目内使用ps -ef 一大串（其实本质上是一样的，只是简化了命令）

**systemctl管理的所有service应该放在 `/usr/lib/systemd/system` 目录下**

## 一、编写脚本

注意：

①java命令如果没有进行全局变量配置的需要进行路径配置

②sh脚本需要赋予权限，否则不能使用 (赋权：chmod 777 xx.sh)

```
#!/bin/bash
_ServerName=不用管这个值,postinstall.sh脚本会替换它的值
JAVA_HOME=不用管这个值,postinstall.sh脚本会替换它的值
# 修改为自己的服务启动类名称,包含包名
_MainClass=com.example.ExampleApplication
# 修改为自己的服务标识
_SystemctlName=example-web

JAVA_BIN=${JAVA_HOME}/bin/java
JAVA_JPS=${JAVA_HOME}/bin/jps
# JAVA_HOME="/opt/opsmgr/web/components/jre18linux64.1"
# jre version global variable
_JRE_VERSION=
# path configuration
_DIR="$( cd "$(dirname "$0")" && pwd )"
# service path
_HOME_DIR=$(dirname "$_DIR")
# 去读服务lib文件夹下所有文件名称,暂时不需要了,classpath可以指定目录
# _LIB_PATH=${_HOME_DIR}/lib
# _JAR_LIST=$(ls ${_LIB_PATH})
# _JAR_LIST_STR=""
# for filename in ${_JAR_LIST}
# do
#  _JAR_LIST_STR=${_JAR_LIST_STR}${filename}:
# done
# _JAR_LIST_STR=${_JAR_LIST_STR%*:}

# java版本
_JRE_VERSION_TXT="${_DIR}/jreversionlinux.txt"
# config.priperties path
_CONF_DIR=${_HOME_DIR}/../../conf
# segmentId

# memory configuration
# avoid attribute don't set up in config.proiperties, Judge not empty and set default value, using "echo" implement return function, forbid echo other string
# $1 default value
# $2 value from config.properties
function safeGetMemory(){
    if [ $# = 1 ]
    then
        echo ${1}M
    else
        echo ${2}M
    fi
}
# memory configuration
# avoid attribute don't set up in config.proiperties, Judge not empty and set default value, using "echo" implement return function, forbid echo other string
# $1 default value
# $2 value from config.properties
function safeGetMemoryForKb(){
    if [ $# = 1 ]
    then
        echo ${1}K
    else
        echo ${2}K
    fi
}

#可以有效降低内存
_MALLOC_ARENA_MAX=$(cat /proc/cpuinfo | grep "processor" | wc -l)
export MALLOC_ARENA_MAX=${_MALLOC_ARENA_MAX}
echo "export MALLOC_ARENA_MAX=${_MALLOC_ARENA_MAX}"

_CONFIG_XMS=`cat ${_CONF_DIR}/config.properties | grep "${_SystemctlName}".[[:digit:]]*.Xms | awk '{print $1}'| awk -F "=" '{print $2}'`
_CONFIG_XMX=`cat ${_CONF_DIR}/config.properties | grep "${_SystemctlName}".[[:digit:]]*.Xmx | awk '{print $1}'| awk -F "=" '{print $2}'`
_CONFIG_XSS=`cat ${_CONF_DIR}/config.properties | grep "${_SystemctlName}".[[:digit:]]*.Xss | awk '{print $1}'| awk -F "=" '{print $2}'`
_CONFIG_METASPACESIZE=`cat ${_CONF_DIR}/config.properties | grep "${_SystemctlName}".[[:digit:]]*.MetaspaceSize | awk '{print $1}'| awk -F "=" '{print $2}'`
_CONFIG_MAXMETASPACESIZE=`cat ${_CONF_DIR}/config.properties | grep "${_SystemctlName}".[[:digit:]]*.MaxMetaspaceSize | awk '{print $1}'| awk -F "=" '{print $2}'`
_CONFIG_MAX_DIRECT_MEMORY_SIZE=`cat ${_CONF_DIR}/config.properties | grep "${_SystemctlName}".[[:digit:]]*.MaxDirectMemorySize | awk '{print $1}'| awk -F "=" '{print $2}'`
_CONFIG_RESERVED_CODE_CACHE_SIZE=`cat ${_CONF_DIR}/config.properties | grep "${_SystemctlName}".[[:digit:]]*.ReservedCodeCacheSize | awk '{print $1}'| awk -F "=" '{print $2}'`
# Easy to view configuration, print properties to file
_MemoryFile=${_DIR}/memory.txt
echo "memory config">${_MemoryFile}
_Xms=`safeGetMemory 256 ${_CONFIG_XMS}`
echo "Xms=${_Xms}">>${_MemoryFile}
_Xmx=`safeGetMemory 256 ${_CONFIG_XMX}`
echo "Xmx=${_Xmx}">>${_MemoryFile}
_Xss=`safeGetMemoryForKb 256 ${_CONFIG_XSS}`
echo "Xss=${_Xss}">>${_MemoryFile}
_MetaspaceSize=`safeGetMemory 96 ${_CONFIG_METASPACESIZE}`
echo "MetaspaceSize=${_MetaspaceSize}">>${_MemoryFile}
_MaxMetaspaceSize=`safeGetMemory 128 ${_CONFIG_MAXMETASPACESIZE}`
echo "MaxMetaspaceSize=${_MaxMetaspaceSize}">>${_MemoryFile}
_MaxDirectMemorySize=`safeGetMemory 256 ${_CONFIG_MAX_DIRECT_MEMORY_SIZE}`
echo "MaxDirectMemorySize=${_MaxDirectMemorySize}">>${_MemoryFile}
_ReservedCodeCacheSize=`safeGetMemory 128 ${_CONFIG_RESERVED_CODE_CACHE_SIZE}`
echo "ReservedCodeCacheSize=${_ReservedCodeCacheSize}">>${_MemoryFile}

# JVM配置项
JAVA_OPTS="$JAVA_OPTS -Duser.dir=${_HOME_DIR}/config"
JAVA_OPTS="$JAVA_OPTS -Xms${_Xms}"
JAVA_OPTS="$JAVA_OPTS -Xmx${_Xmx}"
JAVA_OPTS="$JAVA_OPTS -Xss${_Xss}"
JAVA_OPTS="$JAVA_OPTS -XX:MaxMetaspaceSize=${_MaxMetaspaceSize}"
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=${_MetaspaceSize}"
JAVA_OPTS="$JAVA_OPTS -XX:MaxDirectMemorySize=${_MaxDirectMemorySize}"
JAVA_OPTS="$JAVA_OPTS -XX:ReservedCodeCacheSize=${_ReservedCodeCacheSize}"
JAVA_OPTS="$JAVA_OPTS -XX:NativeMemoryTracking=summary"
JAVA_OPTS="$JAVA_OPTS -XX:SurvivorRatio=8"
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
JAVA_OPTS="$JAVA_OPTS -XX:+DisableExplicitGC"
JAVA_OPTS="$JAVA_OPTS -Djava.security.egd=file:/dev/./urandom"
JAVA_OPTS="$JAVA_OPTS -Djava.net.preferIPv4Stack=true"
# JAVA_OPTS="$JAVA_OPTS -Dcom.sun.jndi.ldap.connect.pool.protocol=\"plain ssl\""
JAVA_OPTS="$JAVA_OPTS -Djava.io.tmpdir=${_HOME_DIR}/temp"
# JAVA_OPTS="$JAVA_OPTS -Djava.rmi.server.hostname=10.19.132.54 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=1099 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
# 调试用参数,放在这里方便复制
# -Xdebug -Xrunjdwp:transport=dt_socket,suspend=n,server=y,address=8889
# -Djava.rmi.server.hostname=10.19.132.54 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=1099 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
# JDK8 特有的参数
JAVA_OPTS_JDK8="$JAVA_OPTS -XX:CICompilerCount=2 -XX:+UseG1GC -XX:ConcGCThreads=1 -XX:ParallelGCThreads=4 -XX:+ExplicitGCInvokesConcurrent"

JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -XX:+PrintGCDetails"
JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -XX:+PrintGCDateStamps"
JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -XX:+PrintGCTimeStamps"
JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -Xloggc:${_HOME_DIR}/logs/gc.log"
JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -XX:+UseGCLogFileRotation"
JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -XX:NumberOfGCLogFiles=10"
JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -XX:GCLogFileSize=8M"
# 新增jdk11所需要的启动参数，格式如下
# JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 XXXXXX"
# JDK11特有的参数
JAVA_OPTS_JDK11="$JAVA_OPTS -Xlog:gc*:file=${_HOME_DIR}/logs/gc.log:time,uptime,uptimemillis:filecount=10,filesize=10240000"
JAVA_OPTS_JDK11="$JAVA_OPTS_JDK11 --add-exports=java.base/jdk.internal.misc=ALL-UNNAMED"
# 新增jdk11所需要的启动参数，格式如下
# JAVA_OPTS_JDK11="$JAVA_OPTS_JDK11 XXXXXX"
# 配置classpath
Classpath_Extra_Jdk8=.:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
_Classpath_jdk8=${Classpath_Extra_Jdk8}:"${_HOME_DIR}/config":"${_HOME_DIR}/lib/*":"${_HOME_DIR}/../../conf"
_Classpath_Openjdk11=.:"${_HOME_DIR}/config":"${_HOME_DIR}/lib/*":"${_HOME_DIR}/../../conf":"${_HOME_DIR}/../../resource/resourcepath"

JAVA_OPTS_JDK8="$JAVA_OPTS_JDK8 -classpath ${_Classpath_jdk8}"
JAVA_OPTS_JDK11="$JAVA_OPTS_JDK11 -classpath ${_Classpath_Openjdk11}"

# 获取jre版本
function judgeJreVersion(){
    # 文件不存在,则调用java -version命令将结果写入文件中
    if [ ! -f "${_JRE_VERSION_TXT}" ]
    then
        `${JAVA_HOME}/bin/java -version>"${_JRE_VERSION_TXT}" 2>&1`
    fi
    # 判断是否为openjdk
    exists=`cat ${_JRE_VERSION_TXT} | grep 1.8 | wc -l`
    if [ ${exists} -gt 0 ]
    then
        _JRE_VERSION=8
    else
        _JRE_VERSION=11
    fi
}
judgeJreVersion
function do_exec()
{
    if [ ${_JRE_VERSION} -eq 8 ]
    then
    # JDK8
        nohup ${JAVA_BIN} $JAVA_OPTS_JDK8 ${_MainClass} >/dev/null 2>&1 &
    elif [ ${_JRE_VERSION} -eq 11 ]
    then
    # OPENJDK11
        nohup ${JAVA_BIN} $JAVA_OPTS_JDK11 ${_MainClass} >/dev/null 2>&1 &
    else
        echo "Invalid JRE version"
    fi
}
# 获取启动类名称(不含包名),用于jps命令获取pid
_MainClassName=${_MainClass##*.}
case "$1" in
    start)
        echo "service name: $_ServerName is starting..."
        chmod -R 777 ${_HOME_DIR}
        chmod -R 777 ${_HOME_DIR}/../../logs
        do_exec
            ;;
    stop)
        _Pid=`${JAVA_JPS} | grep ${_MainClassName} | awk '{print $1}'`
        echo "will kill pid ${_Pid}"
        kill -15 ${_Pid}
        echo ${_ServerName} is stopping
           ;;
    restart)
        _Pid=`${JAVA_JPS} | grep ${_MainClassName} | awk '{print $1}'`
        if [  X"$_Pid"==X ]; then
            kill -15 ${_Pid}
            do_exec
            echo "${_ServerName} restarted"
        else
            echo "service not running, will do nothing"
            exit 1
        fi
            ;;
    status)
        ps -ef | grep ${_MainClassName} | grep -v grep
        ;;
    *)
        echo "usage: service ${_ServerName} {start|stop|restart|status}" >&2
        exit 3
        ;;
esac
```
## 二、配置service

1. 在 `/usr/lib/systemd/system `目录下创建**hik.kafka.zookeeper.1.service**的文件
```
[Unit]
# 服务名
Description=registered zookeeper as system service
After=network.target

[Service]
LimitNOFILE=65536
LimitNPROC=65536
LimitMEMLOCK=infinity
LimitCORE=infinity
User=root
Group=root
Type=forking
Restart=on-failure
KillMode=control-group
# 启动脚本
ExecStart=/opt/opsmgr/web/components/kafka.1/bin/zookeeper/bin/service-mgr start
# 停止脚本
ExecStop=/opt/opsmgr/web/components/kafka.1/bin/zookeeper/bin/service-mgr stop
ExecReload=/opt/opsmgr/web/components/kafka.1/bin/zookeeper/bin/service-mgr restart
Environment=JAVA_HOME=/opt/opsmgr/web/components/jre18linux64.1

[Install]
WantedBy=multi-user.target
```
2. 刷新systemctl配置
```
# 文件创建好之后，刷新配置
systemctl daemon-reload
```

## 三、使用命令

以`hik.kafka.zookeeper.1.service`服务为例
```
#启动服务
systemctl start hik.kafka.zookeeper.1.service

#查看服务状态
systemctl status hik.kafka.zookeeper.1.service

#停止服务
systemctl stop hik.kafka.zookeeper.1.service

#允许服务开机自启动
systemctl enable hik.kafka.zookeeper.1.service
```


## 四、主要命令介绍

```
start：立刻启动后面接的 unit。

stop：立刻关闭后面接的 unit。

restart：立刻关闭后启动后面接的 unit，亦即执行 stop 再 start 的意思。

reload：不关闭 unit 的情况下，重新载入配置文件，让设置生效。

enable：设置下次开机时，后面接的 unit 会被启动。

disable：设置下次开机时，后面接的 unit 不会被启动。

status：目前后面接的这个 unit 的状态，会列出有没有正在执行、开机时是否启动等信息。

is-active：目前有没有正在运行中。

is-enabled：开机时有没有默认要启用这个 unit。

kill ：不要被 kill 这个名字吓着了，它其实是向运行 unit 的进程发送信号。

show：列出 unit 的配置。

mask：注销 unit，注销后你就无法启动这个 unit 了。

unmask：取消对 unit 的注销。
```
