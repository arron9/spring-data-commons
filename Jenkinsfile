import java.util.Properties
import java.io.File

def props = new Properties()
def artifactoryCredentialsRef = ""

pipeline {

	agent none

	triggers {
		pollSCM 'H/10 * * * *'
		upstream(upstreamProjects: "spring-data-build/3.0.x", threshold: hudson.model.Result.SUCCESS)
	}

	options {
		disableConcurrentBuilds()
		buildDiscarder(logRotator(numToKeepStr: '14'))
	}

	stages {

		stage("Initialize") {
			agent {
				label 'data'
			}
			options { timeout(time: 30, unit: 'MINUTES') }
			steps {
				script {
					sh('mkdir -p target')
					sh('curl https://raw.githubusercontent.com/spring-projects/spring-data-build/3.0.x/ci/configuration.properties > target/configuration.properties')
					File propertiesFile = new File('target/configuration.properties')
					propertiesFile.withInputStream {
						props.load(it)
					}
					artifactoryCredentialsRef = props['artifactory.credentials.ref']
				}
			}
		}

		stage("test: baseline (Java 17)") {
			when {
				beforeAgent(true)
				anyOf {
					branch(pattern: "main|(\\d\\.\\d\\.x)", comparator: "REGEXP")
					not { triggeredBy 'UpstreamCause' }
				}
			}
			agent {
				label 'data'
			}
			options { timeout(time: 30, unit: 'MINUTES') }
			environment {
				ARTIFACTORY = credentials("${artifactoryCredentialsRef}")
			}
			steps {
				script {
					docker.withRegistry('', props['docker.credentials.ref']) {
						docker.image(props['docker.java.main.image']).inside(props['docker.java.inside']) {
							withEnv([props['docker.java.env']]) {
								sh './mvnw -s settings.xml clean dependency:list verify -Dsort -U -B'
							}
						}
					}
				}
			}
		}

		stage('Release to artifactory') {
			when {
				beforeAgent(true)
				anyOf {
					branch(pattern: "main|(\\d\\.\\d\\.x)", comparator: "REGEXP")
					not { triggeredBy 'UpstreamCause' }
				}
			}
			agent {
				label 'data'
			}
			options { timeout(time: 20, unit: 'MINUTES') }

			environment {
				ARTIFACTORY = credentials("${artifactoryCredentialsRef}")
			}

			steps {
				script {
					docker.withRegistry('', props['docker.credentials.ref']) {
						docker.image(props['java.main.docker']).inside(props['docker.java.inside']) {
							withEnv([props['docker.java.env']]) {
								sh './mvnw -s settings.xml -Pci,artifactory ' +
										'-Dartifactory.server=https://repo.spring.io ' +
										"-Dartifactory.username=${ARTIFACTORY_USR} " +
										"-Dartifactory.password=${ARTIFACTORY_PSW} " +
										"-Dartifactory.staging-repository=libs-snapshot-local " +
										"-Dartifactory.build-name=spring-data-commons " +
										"-Dartifactory.build-number=${BUILD_NUMBER} " +
										'-Dmaven.test.skip=true clean deploy -U -B'
							}
						}
					}
				}
			}
		}
	}

	post {
		changed {
			script {
				slackSend(
						color: (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger',
						channel: '#spring-data-dev',
						message: "${currentBuild.fullDisplayName} - `${currentBuild.currentResult}`\n${env.BUILD_URL}")
				emailext(
						subject: "[${currentBuild.fullDisplayName}] ${currentBuild.currentResult}",
						mimeType: 'text/html',
						recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider']],
						body: "<a href=\"${env.BUILD_URL}\">${currentBuild.fullDisplayName} is reported as ${currentBuild.currentResult}</a>")
			}
		}
	}
}
