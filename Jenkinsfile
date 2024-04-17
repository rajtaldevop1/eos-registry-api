def label = "eosagent"
//def mvn_version = 'M2'
def mvn_version = 'maven'
podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: build
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  containers:
  - name: build
    image: rajeshtalla0209/eos-jenkins-agent-base:latest
    command:
    - cat
    tty: true
    resources:
      requests:
        memory: "256Mi"  # Request 256 megabytes of memory
        cpu: "100m"      # Request 100 milliCPU (0.1 CPU core)
      limits:
        memory: "1024Mi"  # Limit memory usage to 512 megabytes
        cpu: "500m"      # Limit CPU usage to 200 milliCPU (0.2 CPU core)
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
) {
    node (label) {
        stage ('Checkout SCM'){
          git credentialsId: 'git', url: 'https://github.com/rajtaldevop1/eos-registry-api.git', branch: 'main'
          container('build') {
                stage('Build a Maven project') {
                  withEnv( ["PATH+MAVEN=${tool mvn_version}/bin"] ) {
                   sh "mvn clean package"
                    }
                  //sh './mvnw clean package' 
                   //sh 'mvn clean package'
                }
            }
        }
          stage ('Sonar Scan'){
          container('build') {
                stage('Sonar Scan') {
                  withSonarQubeEnv('sonar') {
                  sh 'mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=eosproj1_eos'
                }
                }
            }
        }


        stage ('Artifactory configuration'){
          container('build') {
                stage('Artifactory configuration') {
                    rtServer (
                    id: "jfrog",
                    url: "https://rajdev.jfrog.io/artifactory",
                    credentialsId: "jfrog"
                )

                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog",
                    releaseRepo: "eos-libs-release-local",
                    snapshotRepo: "eos-libs-release-local"
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog",
                    releaseRepo: "eos-libs-release",
                    snapshotRepo: "eos-libs-release"
                )            
                }
            }
        }
        stage ('Deploy Artifacts'){
          container('build') {
                stage('Deploy Artifacts') {
                  // sh 'chmod 777 /home/jenkins/agent/workspace/eos-registry-api_main/mvnw'
                    sh 'chmod 777 /home/jenkins/agent/workspace/eos-registry-api/mvnw'
                    rtMavenRun (
                    tool: "java", // Tool name from Jenkins configuration
                    useWrapper: true,
                    pom: 'pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                  )
                }
            }
        }
        stage ('Publish build info') {
            container('build') {
                stage('Publish build info') {
                rtPublishBuildInfo (
                    serverId: "jfrog"
                  )
               }
           }
       }
       stage ('Docker Build'){
          container('build') {
                stage('Build Image') {
                    docker.withRegistry( 'https://registry.hub.docker.com', 'docker' ) {
                    def customImage = docker.build("rajeshtalla0209/eos-registry-api:latest")
                    customImage.push()             
                    }
                }
            }
        }

        stage ('Helm Chart') {
          container('build') {
            dir('charts') {
              withCredentials([usernamePassword(credentialsId: 'jfrog', usernameVariable: 'username', passwordVariable: 'password')]) {
              sh '/usr/local/bin/helm package registry-api'
              sh '/usr/local/bin/helm push-artifactory registry-api-1.0.tgz https://rajdev.jfrog.io/artifactory/eos-helm-local --username $username --password $password'
              }
            }
        }
        }
    }
}
