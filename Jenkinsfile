#!groovy

def releasedVersion

node('master') {
	stage('Prepare') {
        deleteDir()
        parallel Checkout: {
            checkout([$class: 'GitSCM', 
				branches: [[name: '*/master']], 
				extensions: [
					[$class: 'UserIdentity', email: "ceanma@gmail.com", name: "César A. Magalhães"],
					[$class: 'WipeWorkspace'], 
					[$class: 'LocalBranch', localBranch: 'master']], 
				userRemoteConfigs: [[credentialsId: "github-credentials", url: "https://github.com/cams7/blog-002.git"]]])
        }, 'Run Zalenium': {
            dockerCmd '''run -d --name zalenium -p 4444:4444 \
            -v /var/run/docker.sock:/var/run/docker.sock \
            --network="host" \
            --privileged 172.42.42.200:18082/dosel/zalenium:3.4.0a start --videoRecordingEnabled false --chromeContainers 1 --firefoxContainers 0'''
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
	def array = releasedVersion.split("\\.")
	return "${array[0]}.${array[1]}.$buildNumber-SNAPSHOT"
}