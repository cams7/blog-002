#!groovy

def releasedVersion

node('master') {
  def dockerTool = tool name: 'docker', type: 'org.jenkinsci.plugins.docker.commons.tools.DockerTool'
  withEnv(["DOCKER=${dockerTool}"]) {
    stage('Prepare') {
        deleteDir()
        parallel Checkout: {
            checkout scm
        }, 'Run Zalenium': {
            dockerCmd '''run -d --name zalenium -p 4444:4444 \
            -v /var/run/docker.sock:/var/run/docker.sock \
            --network="host" \
            --privileged 172.42.42.200:18082/dosel/zalenium:3.4.0a start --videoRecordingEnabled false --chromeContainers 1 --firefoxContainers 0'''
        }
    }

    stage('Build') {
        withMaven(maven: 'apache-maven') {
            dir('app') {
                sh 'mvn clean package'
                dockerCmd 'build --tag 172.42.42.200:18083/automatingguy/sparktodo:SNAPSHOT .'
            }
        }
    }

    stage('Deploy') {
        stage('Deploy') {
            dir('app') {
                dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" 172.42.42.200:18083/automatingguy/sparktodo:SNAPSHOT'
            }
        }
    }

    stage('Tests') {
        try {
            dir('tests/rest-assured') {
                sh './gradlew clean test'
            }
        } finally {
            junit testResults: 'tests/rest-assured/build/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/rest-assured/build/**'
        }

        dockerCmd 'rm -f snapshot'
        dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" 172.42.42.200:18083/automatingguy/sparktodo:SNAPSHOT'

        try {
            withMaven(maven: 'apache-maven') {
                dir('tests/bobcat') {
                    sh 'mvn clean test -Dmaven.test.failure.ignore=true'
                }
            }
        } finally {
            junit testResults: 'tests/bobcat/target/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/bobcat/target/**'
        }

        dockerCmd 'rm -f snapshot'
        dockerCmd 'stop zalenium'
        dockerCmd 'rm zalenium'
    }

    stage('Release') {
        withMaven(maven: 'apache-maven') {
            dir('app') {
                releasedVersion = getReleasedVersion()
                withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'username', usernameVariable: 'password')]) {
                    sh "git config user.email test@automatingguy.com && git config user.name Jenkins"
                    sh "mvn release:prepare release:perform -Dusername=${username} -Dpassword=${password}"
                }
                dockerCmd "build --tag 172.42.42.200:18083/automatingguy/sparktodo:${releasedVersion} ."
            }
        }
    }

    stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' 172.42.42.200:18083/automatingguy/sparktodo:${releasedVersion}"
    }
  }
}

def dockerCmd(args) {
    sh "sudo ${DOCKER} ${args}"
}

def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}