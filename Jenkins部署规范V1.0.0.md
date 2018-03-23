## 1. 总则
1. 为每一个项目创建一个分组，同一项目下的不同功能模块及子模块归档到该分组下。
2. 使用pipeline方式创建打包流程。
3. 每一个jenkins流程必须提供不同环境的部署选项（dev与pro）。
4. 针对不同的程序语言的特点，提供打包环境或运行环境。
5. 命名规则:
    - 分组名称使用==项目中文名称==
    - jenkins流程使用==模块英文名称==,英文名称必须全==小写==
6. 所有docker需要配置dns服务器192.168.0.156

## 2. java-docker部署方式
>该方式适用于java SpringCloud方式构建的程序。
#### 部署要求
1. 使用192.168.0.156&192.168.0.157作为java-docker方式部署dev与pro环境服务器。
2. dev与pro环境相同模块使用不同的端口号进行区分。
#### 部署模板
```
# jenkins pipeline
pipeline {
    agent {
        label '192.168.0.156'
    }
    options {
        timeout(time: 30, unit: 'MINUTES') 
    }
    parameters { 
        //构建选项，选择构建环境
        choice(name: 'release_env',
                choices:'dev\npro', 
                description: 'choose release environment')
    }
    tools{
        maven 'M3'
        jdk 'jdk_1.8.0_161'
    }
    stages{
        
        stage('Prepare'){
            steps{
            //配置jenkins参数
                script{
                project_name="optplantform" // 项目名称
                module_name="$JOB_NAME"     // 模块名称
                network="optplantform"      // docker网络
                svn_addr="svn://192.168.0.151/pycf/Projects/maintain-1.0/maintain-project-1.0" //svn地址
                    if (release_env == 'dev'){
                        port=19090
                    }else{
                        port=19091
                    }
                }
                checkout([$class: 'SubversionSCM', 
                additionalCredentials: [], 
                excludedCommitMessages: '', 
                excludedRegions: '', 
                excludedRevprop: '', 
                excludedUsers: '', 
                filterChangelog: false, 
                ignoreDirPropChanges: false, 
                includedRegions: '', 
                locations: [[credentialsId: 'dffd18a8-473e-477e-a4dc-4d528fa3e55c', 
                            depthOption: 'infinity', 
                            ignoreExternalsOption: true, 
                            local: '.', 
                            remote: svn_addr ]], 
                            quietOperation: true, 
                            workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
        stage('build'){
            steps{
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Deliver'){
            steps{
                sh "JENKINS_NODE_COOKIE=dontKillMe /bin/bash /application/jenskins/script/DeliverJava-docker.sh${network} ${module_name} ${project_name} ${port} ${env.BUILD_NUMBER} ${release_env}"
                // DeliverJava.sh 部署脚本
            }
        }
    }
}

```
```
#!/bin/bash
network_name=$1  #加入的docker网络
docker_name=$2   #模块名称
docker_hub=$3    #项目名称
port=$4          #端口号
BUILD_NUMBER=$5  #构建版本
version=$6       #pro dev 

base_dir = /application/${version}/${docker_hub}/${docker_name}/ 
docker rm -f  ${docker_name}||echo "${docker_name} is not exist " 
docker  images |grep ${docker_hub}/${docker_name}|awk '{print "docker rmi "$3}'|bash

docker network create ${network_name} || echo network ${network_name} exist

[ -d ${base_dir} ]||mkdir -p ${base_dir}

mv ./target/*.jar ${base_dir}

cd ${base_dir}
touch Dockerfile
echo "
FROM java:8
VOLUME /tmp
ADD *.jar /app.jar 
EXPOSE 5000
ENTRYPOINT [\"java\",\"-jar\",\"/app.jar\",\"--server.port=5000\", \"--spring.profiles.active=${version}\"]
" >Dockerfile

docker build -t ${docker_hub}/${docker_name}:0.0.${BUILD_NUMBER} .

sleep 5
docker run -d -h ${docker_name} --name ${docker_name} --network ${network_name} -p ${port}:5000  ${docker_hub}/${docker_name}:0.0.${BUILD_NUMBER}
```


## 3.  静态资源
##### 部署要求
1. 使用192.168.0.156作为静态资源服务器，静态资源统一部署在/application/nginx/html下。
2. 在html下使用pro目录与dev目录作为不同环境的部署，该目录作为项目static的根目录,因此需要做以下映射关系:
    ```
        dev环境 /static ---> html/static/dev
        pro环境 /static ---> html/static/pro
    ```
3. 在pro与dev目录下，为各个项目创建子目录并使用项目名称作为目录名称。项目名称需要与前端进行沟通（需要沟通名称与svn路径）。
4. 需要提供两种不同版本的打包环境:
    - 4.7.3
    - 6.9.5

##### 部署模板
``` java
pipeline {
    agent {
        label '192.168.0.156' //部署节点
    }
    environment { 
        //环境变量，需要根据项目进行修改
        dir = "$JOB_NAME"
        send_mail_to = 'huruizhi@pystandard.com'
        svn_addr = 'svn://192.168.0.151/pycf/ProjectsFE/py_platform/trunk'
    }
    options {
        //五分钟未响应结束
        timeout(time: 30, unit: 'MINUTES') 
    }
    parameters { 
        //构建选项，选择构建环境
        choice(name: 'release_env',
                choices:'development\nstage', 
                description: 'choose release environment')
        //构建选项，选择构建nvm版本
        choice(name: 'version',
                choices:'4.7.3\n6.9.5', 
                description: 'choose nvm version')
        
    }
    stages{
        stage('prepare'){
            //svn版本
            steps{
                checkout([$class: 'SubversionSCM', 
                additionalCredentials: [], 
                excludedCommitMessages: '', 
                excludedRegions: '', 
                excludedRevprop: '', 
                excludedUsers: '', 
                filterChangelog: false, 
                ignoreDirPropChanges: false, 
                includedRegions: '', 
                locations: [[credentialsId: 'dffd18a8-473e-477e-a4dc-4d528fa3e55c', 
                            depthOption: 'infinity', 
                            ignoreExternalsOption: true, 
                            local: '.', 
                            remote: svn_addr ]], 
                            quietOperation: true, 
                            workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
        stage('build'){
            //进行构建
            steps{
                sh "source /root/.bashrc && nvm use ${params.version} && cnpm install && npm run build"
            }
        }
        stage('Deliver'){
            //构建后发布
            steps{
                sh "[ -d /application/nginx/html/${release_env}/static/${dir} ]||mkdir -p /application/nginx/html/static/${dir}"
                sh 'rm -rf  /application/nginx/html/${release_env}/static/${dir}/*'
                sh 'mv ./*  /application/nginx/html/${release_env}/static/${dir}/'
            }
        }
    }
}
```
## 4. java-jar 方式
>该方式适用于未使用Spring Cloud的方式。
#### 部署要求
1. 采用这种方式构建的程序一般是老版本升级，不一定有dev环境。
2. 部署环境与需要与项目经理沟通。
#### 部署模板
```
pipeline {
    agent {
        label '192.168.0.153' //部署服务器
    }
    options {
        timeout(time: 30, unit: 'MINUTES') 
    }
    parameters { 
        //构建选项，选择构建环境
        choice(name: 'release_env',
                choices:'development', 
                description: 'choose release environment')
    }
    tools{
        maven 'M3'
        jdk 'jdk_1.8.0_161'
    }
    stages{
        
        stage('Prepare'){
            steps{
                script {
                    project_name="sys_2"    // 项目名称
                    module_name="$JOB_NAME" // 模块名称
                    port = 16011            // 端口
                    svn_addr="svn://192.168.0.151/pycf/Projects/pySecondSystemNew"   //svn地址
                }
                
                checkout([$class: 'SubversionSCM', 
                additionalCredentials: [], 
                excludedCommitMessages: '', 
                excludedRegions: '', 
                excludedRevprop: '', 
                excludedUsers: '', 
                filterChangelog: false, 
                ignoreDirPropChanges: false, 
                includedRegions: '', 
                locations: [[credentialsId: 'dffd18a8-473e-477e-a4dc-4d528fa3e55c', 
                            depthOption: 'infinity', 
                            ignoreExternalsOption: true, 
                            local: '.', 
                            remote: svn_addr ]], 
                            quietOperation: true, 
                            workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
        stage('build'){
            steps{
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Deliver'){
            steps{
                sh "[ -d /script/java/ ]||mkdir /script/java/"
                sh "mv ./target/*jar /script/java/${env.JOB_NAME}.jar "
                sh "ps -ef|grep java|grep ${env.JOB_NAME}|awk '{print \"kill -15 \" \$1}'|bash"
                sh "ENKINS_NODE_COOKIE=dontKillMe nohup java -jar /script/java/${env.JOB_NAME}.jar --server.port=${port} >/dev/null 2>&1 &"
            }
        }
    }
}
```


## 5.  python-docker部署方式
>该方式适用于python-django框架构建的程序
#### 部署要求
1. 使用192.168.0.156&192.168.0.157作为python-docker方式部署dev与pro环境服务器。
2. dev与pro环境相同模块使用不同的端口号进行区分。
#### 部署模板pipeline
```
pipeline {
    
    agent {
        label '192.168.0.157' // 指定构建节点
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')  // 构建任务超时时间
    }
    
    parameters { 
        //构建选项，选择构建环境，dev开发环境，pro正式环境
        choice(name: 'release_env',
                choices:'dev\npro', 
                description: 'choose release environment')
    }
    
    stages {
        
        stage('准备源码') {
            steps {
                script {
                project_name="brokers_py"   // 项目名称
                module_name="$JOB_NAME"     // 模块名称,调用jenkins系统变量
                svn_addr="svn://192.168.0.151/pycf/Projects/PythonProjects/brokers_py"  // svn路径
                send_mail_to = 'huruizhi@pystandard.com'
                    if (release_env == 'dev'){
                        network="brokers_py_dev" 
                        port=3601
                    } else {
                        network="brokers_py_pro"
                        port=3601
                    }
                }
                checkout([$class: 'SubversionSCM', 
                additionalCredentials: [], 
                excludedCommitMessages: '', 
                excludedRegions: '', 
                excludedRevprop: '', 
                excludedUsers: '', 
                filterChangelog: false, 
                ignoreDirPropChanges: false, 
                includedRegions: '', 
                locations: [[credentialsId: 'dffd18a8-473e-477e-a4dc-4d528fa3e55c', 
                            depthOption: 'infinity', 
                            ignoreExternalsOption: true, 
                            local: '.', 
                            remote: svn_addr ]], 
                            quietOperation: true, 
                            workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
        
        stage('构建运行') {
            steps {
                // 调用构建脚本
                sh "JENKINS_NODE_COOKIE=dontKillMe /bin/bash /application/jenskins/script/DeliverPython3_django-docker_brokers_py.sh ${network} ${module_name} ${project_name} ${port} ${env.BUILD_NUMBER} ${release_env}"
            }
        }
        
    }
}
```
#### 构建脚本：DeliverPython3_django-docker_brokers_py.sh
```
待补充
```