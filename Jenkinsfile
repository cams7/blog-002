#!groovy

def releasedVersion

node('master') {
	stage('Prepare') {
        deleteDir()
        //parallel Checkout: {
            checkout([$class: 'GitSCM', 
				branches: [[name: '*/master']], 
				extensions: [
					[$class: 'UserIdentity', email: "ceanma@gmail.com", name: "César A. Magalhães"],
					[$class: 'WipeWorkspace'], 
					[$class: 'LocalBranch', localBranch: 'master']], 
				userRemoteConfigs: [[credentialsId: "github-credentials", url: "https://github.com/cams7/blog-002.git"]]])
        /*}, 'Run Zalenium': {
            dockerCmd '''run -d -p 4444:4444 \
			-v /var/run/docker.sock:/var/run/docker.sock \
			-v /home/zalenium/videos:/home/seluser/videos \
			--network="host" \
			--privileged dosel/zalenium:3.141.59x start --videoRecordingEnabled true --desiredContainers 1'''
        }*/
    }
	
    stage('Build') {
        withMaven(maven: 'apache-maven') {
            dir('app') {
                sh "mvn -s settings.xml clean package"
                dockerCmd 'build --tag 192.168.100.8:18083/automatingguy/sparktodo:SNAPSHOT .'
            }
        }
    }
	
	stage('Deploy') {
        stage('Deploy') {
            dir('app') {
                dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" 192.168.100.8:18083/automatingguy/sparktodo:SNAPSHOT'
            }
        }
    }
	
	stage('Tests') {
        try {
		    def gradleHome = tool name: 'gradle', type: 'gradle'
			dir('tests/rest-assured') {
				sh "$gradleHome/bin/gradle clean test"
			}          
        } finally {
            junit testResults: 'tests/rest-assured/build/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/rest-assured/build/**'
        }

        dockerCmd 'rm -f snapshot'
        dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" 192.168.100.8:18083/automatingguy/sparktodo:SNAPSHOT'

        try {
            withMaven(maven: 'apache-maven') {
                dir('tests/bobcat') {
                    sh "mvn clean test -Dmaven.test.failure.ignore=true"
                }
            }
        } finally {
            junit testResults: 'tests/bobcat/target/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/bobcat/target/**'
        }

        dockerCmd 'rm -f snapshot'
        //dockerCmd '''rm `docker ps | grep " dosel/zalenium" | awk '{ print $1 }'` -f'''
        //dockerCmd '''rm `docker ps | grep " elgalu/selenium" | awk '{ print $1 }'` -f'''
    }
	
	stage('Release') {
        withMaven(maven: 'apache-maven') {
            dir('app') {
                releasedVersion = getReleasedVersion()
				def snapshotVersion=getSnapshotVersion()                
				withCredentials([usernamePassword(credentialsId: "github-credentials", usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
					sh "mvn --batch-mode -s settings.xml -DskipTests release:clean release:prepare release:perform -DreleaseVersion=${releasedVersion} -Dtag=v${releasedVersion} -DdevelopmentVersion=${getSnapshotVersion()} -Dusername=${GIT_USERNAME} -Dpassword=${GIT_PASSWORD}"
				}
                dockerCmd "build --tag 192.168.100.8:18083/automatingguy/sparktodo:${releasedVersion} ."
            }
        }
    }
	
	stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' 192.168.100.8:18083/automatingguy/sparktodo:${releasedVersion}"
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