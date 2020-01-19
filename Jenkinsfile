#!/usr/bin/env groovy

pipeline {
	agent any
	
	environment {
		GIT_CREDENTIALS_ID = 'github-credentials'
		GIT_URL = 'https://github.com/cams7/blog-002.git'
		GIT_USER_EMAIL = 'ceanma@gmail.com'
		GIT_USER_NAME = 'César A. Magalhães'
		
		ROOT_PATH               = "${pwd()}"
        MAVEN_TARGET_PATH       = "${ROOT_PATH}/app/target"
		MAVEN_SETTINGS_PATH     = "${ROOT_PATH}/app/settings.xml"

        def pom                 = readMavenPom(file: "${ROOT_PATH}/pom.xml")
        ARTIFACT_ID             = pom.getArtifactId()
        RELEASE_VERSION         = pom.getVersion().replace("-SNAPSHOT", "")
				
		NEXUS_CREDENTIALS_ID = 'nexus-credentials'
		GITHUB_PACKAGES_CREDENTIALS_ID = 'github-packages-credentials'
    }
	
	tools {
        maven 'apache-maven'
		gradle 'gradle'
    }
	
	triggers {
        pollSCM "H/3 * * * *"
    }
    
    options {
        timestamps()
    }

    parameters {
		choice (
			choices: ['test', 'prod'],
            name: 'MAVEN_PROFILE', 
			description: 'Maven profile'
		)
		
        booleanParam (
			name: "RELEASE", 
			description: "Build a release from current commit.", 
			defaultValue: false
		)
    }
	
	stages {	
		stage('Prepare') {			
            steps {	
				deleteDir()
                parallel 
					Checkout: {
						checkout([
							$class: 'GitSCM', 
							branches: [[name: '*/master']], 
							extensions: [
								[$class: 'UserIdentity', email: "${GIT_USER_EMAIL}", name: "${GIT_USER_NAME}"],
								[$class: 'WipeWorkspace'], 
								[$class: 'LocalBranch', localBranch: 'master']
							], 
							userRemoteConfigs: [[credentialsId: "${GIT_CREDENTIALS_ID}", url: "${GIT_URL}"]]
						])
					}, 'Run Zalenium': {
						dockerCmd '''run -d --name zalenium -p 4444:4444 \
						-v /var/run/docker.sock:/var/run/docker.sock \
						--network="host" \
						--privileged 172.42.42.200:18082/dosel/zalenium:3.4.0a start --videoRecordingEnabled false --chromeContainers 1 --firefoxContainers 0'''
					}
            }
        }
	}
}

def dockerCmd(args) {
    sh "docker ${args}"
}

/*def releasedVersion

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
	
    stage('Build') {
        withMaven(maven: 'apache-maven') {
            dir('app') {
                sh "mvn -s settings.xml clean package"
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
                    sh "mvn clean test -Dmaven.test.failure.ignore=true"
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
				def snapshotVersion=getSnapshotVersion()                
				withCredentials([usernamePassword(credentialsId: "github-credentials", usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
					sh "mvn --batch-mode -s settings.xml -DskipTests release:clean release:prepare release:perform -DreleaseVersion=${releasedVersion} -Dtag=v${releasedVersion} -DdevelopmentVersion=${getSnapshotVersion()} -Dusername=${GIT_USERNAME} -Dpassword=${GIT_PASSWORD}"
				}
                dockerCmd "build --tag 172.42.42.200:18083/automatingguy/sparktodo:${releasedVersion} ."
            }
        }
    }
	
	stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' 172.42.42.200:18083/automatingguy/sparktodo:${releasedVersion}"
    }
}



def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}

def getSnapshotVersion() {
	def buildNumber=env.BUILD_NUMBER
	def array = releasedVersion.split("\\.")
	return "${array[0]}.${array[1]}.$buildNumber-SNAPSHOT"
}*/