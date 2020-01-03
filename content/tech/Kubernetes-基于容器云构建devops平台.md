+++
title = "Kubernetes 基于容器云构建devops平台"
date = "2020-01-03T07:08:54+08:00"
description = "基于k8s的持续集成平台构建"
tags = ["Kubernetes","devops","ci/cd","开发实战","docker"]
slug = "on-Kubernetes-devops-build"
categories = ["Kubernetes","devops","ci/cd","开发实战","docker"]
+++

### Kubernetes-基于容器云构建devops平台

## 1、基于kubernetes devops的整体方案
本文以Kubernetes为基础，为基于java语言研发团队提供一套完整的devops解决方案。在此方案中，开发人员基于eclipse集成开发环境进行代码；开发人员所开发的代码交由由gitlab进行托管、版本管理和分支管理；代码的依赖更新和构建工作由Maven进行处理；为了提升工作效率和代码质量，在devops中引入SonarQube进行代码检查；对于打包构建后代码，交由docker进行镜像构建，并在私有镜像仓库中对镜像进行管理；最后，devops会将自动从私有镜像仓库从拉取镜像，并在Rancher中进行部署。

!["整体方案图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180705173151.png)

### 基于此devops解决方案的整体工作过程如下所示：

1）开发人员基于eclipse集成开发环境镜像代码开发的，将代码到gitlab中进行托管；

2）jenkins从gitlab拉取代码；

3）jenkins调用Maven对代码进行打包构建；

4）jenkins调用docker构建镜像；

5）jenkins将构建好的镜像上传至基于Nexus的私有镜像仓库；

6）jenkins拉取镜像，并部署镜像至Rancher中。

 

## 2、组件安装部署
此部分描述需要为devops部署的组件，根据整体方案，devops需要使用gitlab、jenkins、nexus、maven、docker和kubernetes这些组件和系统。
其中，gitlab、jenkins、nexus都在kubernetes中安装部署，在jenkins中包含了maven；
docker直接在物理机提供，对于docker的部署不在此部分进行阐述。

### 2.1 代码托管工具-Gitlab
在本文的方案中，代码的托管基于Gitlab。下面是在Kubernetes中部署gitlab的YAML配置文件，在此文件中定义了gitlab部署和服务。gitlab部署使用的镜像为gitlab/gitlab-ce:latest，并暴露了443、80和22这三个端口，并通过NFS对配置文件、日志和数据进行持久化。在服务中，端口的类型为NodePort，即允许集群外的用户可以通过映射在主机节点上的端口对gitlab进行访问。

```
#------------------------定义代理服务-------------------------
apiVersion: v1
kind: Service
metadata:
  name: gitlab
spec:
  type: NodePort
  ports:
  # Port上的映射端口
  - port: 443
    targetPort: 443
    name: gitlab443
  - port: 80
    targetPort: 80
    name: gitlab80
  - port: 22
    targetPort: 22
    name: gitlab22
  selector:
    app: gitlab



# ------------------------定义Gitlab的部署 -----------------------


apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: gitlab
spec:
  selector:
    matchLabels:
      app: gitlab
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
      # 应用的镜像
      - image: gitlab/gitlab-ce 
        name: gitlab
        imagePullPolicy: IfNotPresent
        # 应用的内部端口
        ports:
        - containerPort: 443
          name: gitlab443
        - containerPort: 80
          name: gitlab80
        - containerPort: 22
          name: gitlab22
        volumeMounts:
        # gitlab持久化
        - name: gitlab-persistent-config
          mountPath: /etc/gitlab
        - name: gitlab-persistent-logs
          mountPath: /var/log/gitlab
        - name: gitlab-persistent-data
          mountPath: /var/opt/gitlab
      imagePullSecrets:
      - name: devops-repo
      volumes:
      # 使用nfs互联网存储
      - name: gitlab-persistent-config
        nfs:
          server: 192.168.8.150
          path: /k8s-nfs/gitlab/config
      - name: gitlab-persistent-logs
        nfs:
          server: 192.168.8.150
          path: /k8s-nfs/gitlab/logs
      - name: gitlab-persistent-data
        nfs:
          server: 192.168.8.150
          path: /k8s-nfs/gitlab/data
          
 ```
通过kubectl命令工具，执行如下的命令，在kubernetes集群中部署gitlab：
 ```
$ kubectl create -f {path}/gitlab.yaml
 ```
### 2.2 镜像仓库-Nexus
在本文的devops方案中，采用Nexus作为docker私有镜像仓库和maven的远程仓库。下面是在Kubernetes中部署Nexus的YAML配置文件，在此文件中定义了Nexus部署和服务。Nexus部署使用的镜像为sonatype/nexus3:latest，并暴露了8081、5001这两个端口，并通过NFS对配置文件、日志和数据进行持久化。在服务中，端口的类型为NodePort，即允许集群外的用户可以通过映射在主机节点上的端口对nexus进行访问。其中，5001作为docker私有镜像仓库的端口。

```
#------------------------定义Nexus代理服务-------------------------
apiVersion: v1
kind: Service
metadata:
  name: nexus3
spec:
  type: NodePort
  ports:
  # Port上的映射端口
  - port: 8081
    targetPort: 8081
    name: nexus8081
  - port: 5001
    targetPort: 5001
    name: nexus5001
  selector:
    app: nexus3

#-----------------------定义Nexus部署-------------------------
apiVersion: apps/v1beta2
kind: Deployment
metadata:
   name: nexus3
   labels:
     app: nexus3
spec:
   replicas: 1 #副本数为3
   selector: #选择器
     matchLabels:
       app: nexus3 #部署通过app：nginx标签选择相关资源
   template: #Pod的模板规范
     metadata:
       labels:
         app: nexus3 #Pod的标签
     spec:
       imagePullSecrets:
       - name: devops-repo
       containers: #容器
       - name: nexus3
         image: sonatype/nexus3
         imagePullPolicy: IfNotPresent
         ports: #端口
         - containerPort: 8081
           name: nexus8081
         - containerPort: 5001
           name: nexus5001
         volumeMounts:
         - name: nexus-persistent-storage
           mountPath: /nexus-data
       volumes:
       # 宿主机上的目录
       - name: nexus-persistent-storage
         nfs:
           server: 192.168.8.150
           path: /k8s-nfs/nexus
 ```
通过kubectl命令工具，执行如下的命令，在kubernetes集群中部署nexus：
```
$ kubectl create -f {path}/nexus.yaml
```
### 2.3 流水线工具-Jenkins
2.3.1 jenkins安装部署
在本文的devops方案中，采用jenkins作为流水线工具。下面是在Kubernetes中部署jenkins的YAML配置文件，在此文件中定义了jenkins部署和服务。jenkins部署使用的镜像为lw96/java8-jenkins-maven-git-vim:latest，并暴露了8080这个端口，并通过NFS对配置文件和数据进行持久化。在服务中，端口的类型为NodePort，即允许集群外的用户可以通过映射在主机节点上的端口对jenkins进行访问。另外，在此镜像中也提供maven和java。
 ```
#------------------------定义代理服务-------------------------
apiVersion: v1
kind: Service
metadata:
  name: jenkins-devops
spec:
  type: NodePort
  ports:
  # Port上的映射端口 
  - port: 8080
    targetPort: 8080
    name: pipeline8080
  selector:
    app: jenkins-devops


# ------------------------定义mysql的部署 -----------------------
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: jenkins-devops
spec:
  selector:
    matchLabels:
      app: jenkins-devops
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: jenkins-devops
    spec:
      containers:
      # 应用的镜像
      - image: 10.0.32.148:1008/lw96/java8-jenkins-maven-git-vim
        name: jenkins-devops
        imagePullPolicy: IfNotPresent
        # 应用的内部端口
        ports:
        - containerPort: 8080
          name: pipeline8080 
        volumeMounts:
        # jenkins-devops持久化
        - name: pipeline-persistent
          mountPath: /etc/localtime
        # jenkins-devops持久化
        - name: pipeline-persistent
          mountPath: /jenkins
        # jenkins-devops持久化
        - name: pipeline-persistent-repo
          mountPath: /root/.m2/repository
        - name: pipeline-persistent-mnt
          mountPath: /mnt
      volumes:
      # 使用nfs互联网存储
      - name: pipeline-persistent
        nfs:
          server: 192.168.8.150
          path: /k8s-nfs/jenkins-devops
      - name: pipeline-persistent-mnt
        nfs:
          server: 192.168.8.150
          path: /k8s-nfs/jenkins-devops/mnt
      - name: pipeline-persistent-repo
        nfs:
          server: 192.168.8.150
          path: /k8s-nfs/jenkins-devops/repo
 ```          
通过kubectl命令工具，执行如下的命令，在kubernetes集群中部署jenkins：
 ```
$ kubectl create -f {path}/jenkins-devops.yaml
 ```
注意，后续将执行下面的操作：

将Kubernetes集群的kubeconfig文件拷贝到192.168.8.150主机的/k8s-nfs/jenkins-devops/mnt目录下；
将maven的settings.xml文件拷贝至到192.168.8.150主机的/k8s-nfs/jenkins-devops/repo目录下；
将maven的依赖插件包拷贝至到192.168.8.150主机的/k8s-nfs/jenkins-devops/repo目录下。
## 3、 devops平台搭建
### 3.1 nexus设置
nexus在devops中承担两个功能，作为maven的远程仓库和作为docker的私有镜像仓库。

在本文中，使用nexus默认安装的maven-snapshots、maven-releases和maven-public这三个仓库。
!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706112632.png)


并为docker创建一个名为registry的私有镜像仓库，其端口为5001：

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706112828.png)

### 3.2 maven设置
maven负责管理代码的依赖关系和构建。maven通过settings.xml文件设置运行环境，包括与远程仓库的连接。本文中的settings.xml如下所示，http://nexus3:8081中的nexus3是在kubernetes中的服务名称。将settings.xml拷贝至192.168.8.150机器的/k8s-nfs/jenkins-devops/repo目录下。
 ```
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <pluginGroups>
  </pluginGroups>
  <proxies>
  </proxies>
  <servers>
     <server>     
      <id>maven-release</id>     
      <username>admin</username>     
      <password>admin123</password>     
    </server>     
    <server>     
      <id>maven-snapshots</id>     
      <username>admin</username>     
      <password>admin123</password>     
    </server>     
  </servers>

  <mirrors>
    <mirror>        
     <id>nexus</id>         
     <url>http://nexus3:8081/repository/maven-public/</url>        
     <mirrorOf>*</mirrorOf>        
   </mirror>     
 </mirrors> 

  <profiles>
  <profile>
      <id>maven-release</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <repositories>
        <repository>
          <id>maven-release</id>
          <name>Repository for Release</name>
          <url>http://nexus3:8081/repository/maven-releases/</url>
     <releases>
     <enabled>false</enabled>
     </releases>
     <snapshots>
     <enabled>true</enabled>
     </snapshots>
        </repository>
      </repositories>
    </profile>

  <profile>
      <id>maven-snapshots</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <repositories>
        <repository>
          <id>maven-release</id>
          <name>Repository for Release</name>
          <url>http://nexus3:8081/repository/maven-snapshots/</url>
     <releases>
     <enabled>false</enabled>
     </releases>
     <snapshots>
     <enabled>true</enabled>
     </snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>
</settings>
 ```
### 3.3 docker设置
为了能够支持将远程提交的代码构建成镜像，以及将构建好的镜像上传至镜像仓库需要在/etc/docker/daemon.json的文件中添加下面的内容。其中，10.0.32.163:32476为镜像仓库的地址和端口，tcp://0.0.0.0:4243为对外暴露的地址和端口。
 ```
{
"hosts":["tcp://0.0.0.0:4243","unix:///var/run/docker.sock"],
"insecure-registries":["10.0.32.163:32476"]
}
 ```
并通过执行下面的命令重启docker服务：
 ```
$ systemctl daemon-reload
$ systemctl restart docker
 ```
### 3.4 jenkins设置
3.4.1 安装插件
jenkins作为devops平台的流程线工具，需要从gitlab中获取代码，并提交给maven进行构建；在代码构建成功后，调用docker构建镜像，并将上传至基于Nexus的私有镜像仓库；最终，在Kubernetes中部署和运行镜像。为了实现上述能力，需要在jenkins中安装如下插件：

git plugin：与gitlab集成的插件，用于获取代码；
maven plugin：与maven集成的插件，用于构建代码；
CloudBees Docker Build and Publish plugin：与docker集成的插件，用于构建docker镜像，并上传至镜像仓库；
Kubernetes Continuous Deploy Plugin：与kubernetes集成的插件，用于将镜像部署到kubernetes环境。
3.4.2 maven设置
在jenkins中的“全局工具配置”页面，设置maven的安装信息，name可以按照自己的喜好填写，MAVEN_HOME为maven的安装地址，此处为/opt/maven。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706132104.png)

## 4、devops持续集成示例
### 1）安装git客户端和创建密钥

在工作计算上安装git客户端，并通过下面的命令创建ssh密钥：
 ```
ssh-keygen -t rsa -C "your.email@example.com" -b 4096
 ```
执行上述命令后，会在本地的～/.ssh目录创建id_rsa.pub(公钥)和id_rsa(私钥)。

### 2）在gitlab创建oms项目

进入gitlab，并创建一个名称为oms的项目。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706135052.png)


在gitlab中添加所创建的公钥：

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706135307.png)


### 3）在eclipse中进行代码开发

在eclipse中创建类型为maven的oms项目，并进行代码开发。在完成代码开发后，将代码上传至gitlab中进行代码托管。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706135457.png)

### 4）在jenkins中创建oms项目

进入jenkins，创建一个名称为oms的maven项目。


!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706135816.png)
 

### 5）在jenkins中设置获取代码信息

在jenkins中，进入oms的配置页面，在源代码管理处设置获取源代码的相关信息。

Repository URL：项目在gitlab中的地址；
Credentials：访问gitlab的认证方式；
Branch Specifier：代码的分支。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706140335.png)

此处访问gitlab的认证方式为“SSH Username with private key”，Private Key的值来自于～/.ssh目录下id_rsa(私钥)的内容。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706145635.png)

### 6）在jenkins中设置构建和上传镜像信息

在oms项目的配置的“Docker Build and Publish”部分，填写如下的信息；

Repository Name：镜像的名称；
Tag：镜像的版本；
Docker Host URI：docker服务的地址和端口；
Docker registry URL：docker镜像仓库的地址；
Registry credentials：镜像仓库的认证方式。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706140801.png)

### 7）在jenkins中设置部署容器信息

在oms项目的配置的“Deploy to Kubernetes”部分，填写如下的信息；

Kubernetes Cluster Credentials：Kubernetes集群的认证方式；
Path：kubeconfig文件的所在地址；
Config Files：构建YAML配置文件。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706140848.png)


### 8）在jenkins中执行oms构建

在oms项目创建和设置完成后，可以对项目进行构建操作。通过一键操作，jenkins将会完成从构建、打包成镜像和部署的所有工作内容：

从gitlab中获取oms的代码；
提交给maven进行构建；
调用docker构建镜像；
上传镜像至Nexus的私有镜像仓库；
部署镜像到kubernetes集群。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706141150.png)


### 9）在kubernetes中查看部署情况

进入kubernetes(本文使用的为Rancher系统)界面，在default命名空间下，可以看到已部署的oms。

!["配图"](/images/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180706140929.png)

## 参考材料

[^1]: 《User documentation》地址：http://10.0.32.163:32269/help/user/index.md

[^2]: 《CloudBees Docker Build and Publish plugin》地址：https://wiki.jenkins.io/display/JENKINS/CloudBees+Docker+Build+and+Publish+plugin

[^3]: 《Kubernetes Continuous Deploy Plugin》地址：https://wiki.jenkins.io/display/JENKINS/Kubernetes+Continuous+Deploy+Plugin

