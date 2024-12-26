### 后端java pipeline脚本
```groovy
// 做下拉框联动
// 需要用到scriptler插件
// 下面是scriptler中的样例脚本
/**
List devList  = ["None"]
List testList  = ["itmg-ariot-abs-iot-manager"]
List stageList = ["Select:selected", "stage1"]
List prodList  = ["itmg-ariot-abs-iot-manager"]

List default_item = ["None"]

if (Environment == 'DEV') {
  return devList
} else if (Environment == 'TEST') {
  return testList
} else if (Environment == 'STAGE') {
  return stageList
} else if (Environment == 'PROD') {
  return prodList
} else {
  return default_item
}
*/

properties([
  // 管道速度/耐久性覆盖
  durabilityHint('PERFORMANCE_OPTIMIZED'), 
  // 保留已完成构建的储藏
  preserveStashes(), 
  // 配置联动选框
  parameters([
    activeChoice(choiceType: 'PT_SINGLE_SELECT', description: '选类型', filterLength: 1, filterable: false, name: 'Environment', randomName: 'choice-parameter-779608790271869', script: scriptlerScript(isSandboxed: true, scriptlerBuilder: [builderId: '1726884996564_17', parameters: [], propagateParams: false, scriptId: 'Environments.groovy'])), 
    reactiveChoice(choiceType: 'PT_CHECKBOX', description: '应用列表', filterLength: 1, filterable: true, name: 'Servers', randomName: 'choice-parameter-779608792418725', referencedParameters: 'Environment', script: scriptlerScript(isSandboxed: true, scriptlerBuilder: [builderId: '1726884996566_18', parameters: [], propagateParams: false, scriptId: 'BackendAriot.groovy']))
    ])
  ])

pipeline {
    agent any
    
    tools {
      maven 'local-mvn'
      jdk 'jdk-11'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '3')) //保留三个历史构建版本
    }

    parameters {
        // 代码地址
        string(defaultValue: 'http://114.255.128.176:88/syb-dev/root/h-itmg/service/bcl/ariot.git', description: '代码仓库地址', name: 'Gitlab_Registry_URL')
        // 版本分支
        gitParameter(branch: '', branchFilter: '.*', defaultValue: 'sz_dev', description: '代码分支', name: 'branch', quickFilterEnabled: false, selectedValue: 'NONE', sortMode: 'NONE', tagFilter: '*', type: 'GitParameterDefinition')
    }

    environment {
        commitLog = ''
        commit_author = ''
        SonarQube_URL='http://192.168.3.30:9100'
        SonarQube_Secret='sqa_cdfa9063791ff2b619703c6ed3d55cad84ea7407'
        ProjectName="${env.JOB_NAME}"
        scannerHome = tool 'SonarQube_Scanner_4.8'
        DOCKER_REPOSITORY='registry.cn-gdgz1.ctyun.cn/h-vpp'
        build_time = ''
        build_dir = 'itmg-bcl-ariot-abs'
        multiple_platform = "--platform=linux/arm64,linux/amd64"
    }
    

    stages {
        stage('代码拉取') {
            steps {
                echo '拉取Git代码'
                checkout scmGit(branches: [[name: '${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '53773151-da42-4355-8504-45bcb0abb1eb', url: '${Gitlab_Registry_URL}']])
                
                script{
                  // 获取最近一次代码提交信息
                  commitLog = sh(script: 'git log --oneline -n 1',returnStdout: true).trim()
                  echo "Git Commit Log:\n${commitLog}"
                  
                  commit_author = sh(script: 'git log -n 1 --pretty=format:"%an"',returnStdout: true).trim()
                  echo "Git Commit Author:\n${commit_author}"
        
                  commit_counters = sh(script: 'git log --oneline | wc -l',returnStdout: true).trim()
                  echo "Git Commit Counters:\n${commit_counters}"

                  build_time = sh (script: 'date +%Y%m%d%H%M', returnStdout: true).trim()
                  echo "build time:\n${build_time}"
                } 
            }
        }

        stage('SonarQube Analysis') {
            when {
              expression { 
                  // 检查复选框的值，如果包含'DEV'，则执行步骤
                  return params.Environment.contains('DEV') 
              }
            }
            steps{
                echo '代码质量检查'
                script{
                    def SonarQube_URL=env.SonarQube_URL
                    def SonarQube_Secret=env.SonarQube_Secret

                    echo "${ProjectName}"
                    echo "${SonarQube_URL}"
                    echo "${SonarQube_Secret}"
                    def sonarProperties = [
                            "sonar.projectKey=${ProjectName}",
                            "sonar.host.url=${SonarQube_URL}",
                            "sonar.login=${SonarQube_Secret}"
                            ]
                    withSonarQubeEnv('SonarQube') {
                        sh "mvn sonar:sonar -D${sonarProperties.join(' -D')}"
                    }
                    
                    // 代码检测失败,将不再继续执行后面的任务,直接退出,报告返回的超时时长设为5分钟
                    timeout(time: 3,unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
       // 引用其他的包
        stage('构建基础依赖') {
          when {
            anyOf {
              expression { return params.Environment.contains('TEST') }
              expression { return params.Environment.contains('PROD') } 
            }
          }  
          steps {
            echo '构建基础依赖'
            script {

              // 使用build调度前端job，并传参，将结果赋值给dtkBuild
            //   def dtkBuild=build(job: "backend-dtk", parameters: [gitParameter(name: "branch", value: params.branch)])       
            //   println dtkBuild.getProjectName()
            //   println dtkBuild.getNumber()
            //   println dtkBuild.getBuildVariables() 
              // println dtkBuild.buildVariables.image_tag
              // 使用build调度前端job，并传参，将结果赋值给dtkBuild
              def iotBuild=build(job: 'backend-iot', parameters: [gitParameter(name: 'branch', value: params.branch)])       
              println iotBuild.getProjectName()
              println iotBuild.getNumber()
              println iotBuild.getBuildVariables() 
            }

          }
        }
        
        
        stage('编译') {
          when {
            anyOf {
              expression { return params.Environment.contains('TEST') }
              expression { return params.Environment.contains('PROD') } 
            }
          }  
          steps {
            echo '编译打包'
            script {
              sh "mvn clean install -Dmaven.test.skip=true -Ddockerfile.skip=true"
            }

          }
        }

         stage('打包镜像') {
           when {
               anyOf {
                 expression { return params.Environment.contains('TEST') }
                 expression { return params.Environment.contains('PROD') } 
               }
           }  
           steps {
             echo '打包镜像'
             script {
               for (server in params.Servers.tokenize(',')) {
                 sh "docker build -t ${DOCKER_REPOSITORY}/${server}:${build_time}_${commit_counters} ${build_dir}/${server} --no-cache"
                 sh "docker push ${DOCKER_REPOSITORY}/${server}:${build_time}_${commit_counters}"
                 sh "docker rmi ${DOCKER_REPOSITORY}/${server}:${build_time}_${commit_counters}"
               }
             }

           }
         }
        
        stage('部署') {
          when {
            expression { return params.Environment.contains('TEST') }
          }  
          steps {
            echo '部署到测试环境'
            
            script {
              for (server in params.Servers.tokenize(',')) {
                def substr = server.split('-')[-1]
                def deploy = "ariot-${substr}"
                echo "${deploy}"
                def command = "kubectl set image deployment/${deploy} ${deploy}=${DOCKER_REPOSITORY}/${server}:${build_time}_${commit_counters} -n hvpp"
                echo "${command}"
                sh "ssh root@192.168.3.57 ${command}"
              }
            }
          }
        }

        stage('构建多架构镜像，自动更新演示环境') {
          when {
            expression { return params.Environment.contains('PROD') }
          }  

          steps {
            echo '构建多架构镜像，并更新演示环境'
            script {
              // 登录华为云镜像镜仓库， 天翼云镜像仓库无法打多版本镜像
              sh "docker login -u cn-north-1@IY2FYXRHUQVKV5DAA1A2 -p 6d094a81650d98fdd2e476d3a6a5132ed2d63f5e1a48221ab4ea7a1f4fc18ee4 swr.cn-north-1.myhuaweicloud.com"
              for (server in params.Servers.tokenize(',')) {

                // docker buildx build -t registry.cn-gdgz1.ctyun.cn/h-vpp/hvpp-base:multiple --build-context name=/home/docker-test/aif-auth  --no-cache   --push --provenance=false .
                sh "cd ${build_dir}/${server} && docker buildx build -t swr.cn-north-1.myhuaweicloud.com/h-vpp/${server}:${build_time}_${commit_counters} ${multiple_platform} --no-cache --push --provenance=false ."
 
                
                // 更新演示环境(192.168.3.30)里docker-compose里的image值
                def image="swr.cn-north-1.myhuaweicloud.com/h-vpp/${server}:${build_time}_${commit_counters}"
                sh """
                    ssh 192.168.3.30 "yq -i '.services.${server}.image=\\"${image}\\"' /home/hvpp-demo/backend/ariot.yaml"
                    """
                // 执行更新动作
                sh """
                    ssh 192.168.3.30 "cd /home/hvpp-demo/backend && docker-compose -f ariot.yaml up -d ${server}"
                """
                // 更新一个镜像停止1s
                sleep(1)
              }
            }

          }

        }
    }
}

```

### 前端 pipeline脚本

```groovy
pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '3')) //保留三个历史构建版本
    }
    
    tools {
       jdk 'jdk-11'
       nodejs 'nodejs-14.17.6'
    }
    
    

    parameters {
        // 代码地址
        string(defaultValue: 'http://114.255.128.176:88/syb-dev/root/h-itmg/web/bcl/iot.git', description: '代码仓库地址', name: 'Gitlab_Registry_URL')
        // 版本分支
        gitParameter(branch: '', branchFilter: '.*', defaultValue: 'release-dev', description: '代码分支', name: 'branch', quickFilterEnabled: false, selectedValue: 'NONE', sortMode: 'NONE', tagFilter: '*', type: 'GitParameterDefinition')
        
        choice(name: 'Environment',choices: ['DEV','TEST','PROD'],description: "选类型")

    }

    environment {
        commitLog = ''
        commit_author = ''
        ProjectName="${env.JOB_NAME}"
        scannerHome = tool 'SonarQube_Scanner_4.8'
    }

    stages {
        stage('代码拉取') {
            steps {
                echo '拉取Git代码'
                checkout scmGit(branches: [[name: '${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '53773151-da42-4355-8504-45bcb0abb1eb', url: '${Gitlab_Registry_URL}']])
                
                script{
                  // 获取最近一次代码提交信息
                  commitLog = sh(script: 'git log --oneline -n 1',returnStdout: true).trim()
                  echo "Git Commit Log:\n${commitLog}"
                  
                  commit_author = sh(script: 'git log -n 1 --pretty=format:"%an"',returnStdout: true).trim()
                  echo "Git Commit Author:\n${commit_author}"
        
                  commit_counters = sh(script: 'git log --oneline | wc -l',returnStdout: true).trim()
                  echo "Git Commit Counters:\n${commit_counters}"
                } 
            }
        }

        stage('SonarQube Analysis') {
            when {
              expression { 
                  // 检查复选框的值，如果包含'DEV'，则执行步骤
                  return params.Environment.contains('DEV') 
              }
            }
            steps{
                echo '代码质量检查'
                script{
                    def SonarQube_URL=env.SonarQube_URL
                    def SonarQube_Secret=env.SonarQube_Secret

                    echo "${ProjectName}"
                    echo "${SonarQube_URL}"
                    echo "${SonarQube_Secret}"
                    
                    echo "${JAVA_HOME}"

                    withSonarQubeEnv('SonarQube') {
                        sh '''
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=${ProjectName} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SonarQube_URL} \
                        -Dsonar.login=${SonarQube_Secret}
                        '''
                    }

                    // 代码检测失败,将不再继续执行后面的任务,直接退出,报告返回的超时时长设为5分钟
                    timeout(time: 3,unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('编译'){
            when {
                anyOf {
                  expression { return params.Environment.contains('TEST') }
                  expression { return params.Environment.contains('PROD') } 
                }
            }
            steps{
                script{
                    sh """
                    npm install
                    npm run build
                    """
                }
            }
        }
        
        stage('测试环境更新'){
            when {
                expression { return params.Environment.contains('TEST') }
            }
            steps{
                script{
                    sh """
                    rm -rf ariot
                    mv dist ariot
                    ssh root@192.168.3.55 "mv /usr/local/nginx/html/front/ariot  /usr/local/nginx/html/front/ariot-`date +%Y%m%d%H%M`"
                    scp -r ariot root@192.168.3.55:/usr/local/nginx/html/front
                    """
                }
            }
        }

        stage('演示环境更新'){
            when {
                expression { return params.Environment.contains('PORD') }
            }
            steps{
                script{
                    sh """
                    rm -rf ariot
                    mv dist ariot
                    ssh root@192.168.3.30 "mv /home/hvpp-demo/front/ariot  /home/hvpp-demo/front/ariot-`date +%Y%m%d%H%M`"
                    scp -r ariot root@192.168.3.30:/home/hvpp-demo/front
                    """
                }
            }
        }

        
    }
}

```