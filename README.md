# K8S-prometheus
一、简介
1、Prometheus
Prometheus（普罗米修斯）是一套开源的监控&报警&时间序列数据库的组合.由SoundCloud公司开发。
Prometheus基本原理是通过HTTP协议周期性抓取被监控组件的状态，这样做的好处是任意组件只要提供HTTP接口就可以接入监控系统，不需要任何SDK或者其他的集成过程。这样做非常适合虚拟化环境比如VM或者Docker 。
Prometheus应该是为数不多的适合Docker、Mesos、Kubernetes环境的监控系统之一。近几年随着k8s的流行，prometheus成为了一个越来越流行的监控工具
2、Grafana
Grafana是一个通用的可视化工具。不仅仅适用于展示Prometheus下的监控数据，也同样适用于一些其他的数据可视化需求。
（1）Grafana基本概念
数据源（Data Source）
Prometheus这类为其提供数据的对象均称为数据源（Data Source）。目前，Grafana官方提供了对：Graphite, InfluxDB, OpenTSDB, Prometheus, Elasticsearch, CloudWatch的支持。只需要将这些对象以数据源的形式添加到Grafana中，Grafana便可以轻松的实现对这些数据的可视化工作。
仪表盘（Dashboard）
通过数据源定义好可视化的数据来源之后，对于用户而言最重要的事情就是实现数据的可视化。在Grafana中，我们通过Dashboard来组织和管理我们的数据可视化图表
Panel（面板）
Dashboard中一个最基本的可视化单元为一个Panel（面板），每一个Panel是一个完全独立的部分，通过Panel的Query Editor（查询编辑器）我们可以为每一个Panel自己查询的数据源以及数据查询方式，
二、部署流程

#创建监控 namespace
kubectl create ns monitor-sa

部署 node-exporter并查看情况
kubectl apply -f node-export.yaml

kubectl get pods -n monitor-sa -o wide
应该可以看到三个节点（一主两从）都部署了exporter

#创建一个 sa 账号 monitor
kubectl create serviceaccount monitor -n monitor-sa

#把 sa 账号 monitor 通过 clusterrolebing 绑定到 clusterrole 上
kubectl create clusterrolebinding monitor-clusterrolebinding -n monitor-sa --clusterrole=cluster-admin  --serviceaccount=monitor-sa:monitor

#创建一个 configmap 存储卷，用来存放 prometheus 配置信息
kubectl apply -f prometheus-cfg.yaml

#通过 deployment 部署 prometheus
kubectl apply -f prometheus-deploy.yaml

kubectl get pods -o wide -n monitor-sa

#给 prometheus pod 创建一个 service
kubectl apply -f prometheus-svc.yaml kubectl get svc -n monitor-sa

通过浏览器访问：![image](https://user-images.githubusercontent.com/60837389/225901526-6da34065-249e-4eff-ac7d-0acc67d5f518.png)

 
###为了每次修改配置文件可以热加载prometheus，也就是不停止prometheus，就可以使配置生效，想要使配置生效可用如下热加载命令：
kubectl get pods -n monitor-sa -o wide -l app=prometheus
#想要使配置生效可用如下命令热加载
curl -X POST -Ls http://上一条命令查询的ip /-/reload 

#查看 log
kubectl logs -n monitor-sa prometheus-server-75fb7f8fc6-46kd9(用你自己的)  | grep "Loading configuration file"

#Grafana 安装
kubectl apply -f grafana.yaml

kubectl get pods -n kube-system -l task=monitoring -o wide

kubectl get svc -n kube-system | grep grafana

//Grafana 配置 
（1）浏览器访问http://你的主机:上面查询的nodeport ，登陆 grafana

（2）开始配置 grafana 的 web 界面：选择 Add data source
【Name】设置成 Prometheus
【Type】选择  Prometheus
【URL】设置成 http://prometheus.monitor-sa.svc:9090		#使用service的集群内部端口配置服务端地址
点击 【Save & Test】

（4）监控 node 状态
点击左侧+号选择【Import】
点击【Upload .json File】导入 node_exporter.json 模板
【Prometheus】选择 Prometheus
点击【Import】
最后效果如图：![image](https://user-images.githubusercontent.com/60837389/225901578-dc795298-de0b-4de7-a5b0-beacf0c5db4a.png)

 
