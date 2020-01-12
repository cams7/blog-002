#!groovy

def releasedVersion

node('master') {
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
}

def dockerCmd(args) {
    sh "docker ${args}"
}

def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}