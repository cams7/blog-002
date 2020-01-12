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
	stage('Tests') {
        /*try {
		    def gradleHome = tool name: 'gradle', type: 'gradle'
			dir('tests/rest-assured') {
				sh "$gradleHome/bin/gradle clean test"
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
        }*/

        dockerCmd 'rm -f snapshot'
        dockerCmd 'stop zalenium'
        dockerCmd 'rm zalenium'
    }
	stage('Release') {
        withMaven(maven: 'apache-maven') {
            dir('app') {
                releasedVersion = getReleasedVersion()
				def snapshotVersion=getSnapshotVersion()
				sh "echo ${releasedVersion}"
				sh "echo ${snapshotVersion}"
                /*withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'username', usernameVariable: 'password')]) {
                    sh 'git config user.email ceanma@gmail.com && git config user.name "César A. Magalhães"'
                    sh "mvn release:clean release:prepare release:perform -DreleaseVersion=${releasedVersion} -Dtag=v${releasedVersion} -DdevelopmentVersion=${releasedVersion}.${buildNumber}-SNAPSHOT -Dusername=${username} -Dpassword=${password}"
                }
                dockerCmd "build --tag 172.42.42.200:18083/automatingguy/sparktodo:${releasedVersion} ."*/
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

def getSnapshotVersion() {
	def buildNumber=env.BUILD_NUMBER
	def array = releasedVersion.split('\.')
	return "${array[0]}.${array[1]}.$buildNumber-SNAPSHOT"
}