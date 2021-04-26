// Fargate Template
pipeline {
	options {
		disableConcurrentBuilds()
		}
	environment {
		GIT_REPO = "https://github.com/caporg/online-def-resp-batch"
		PROJECT_NAME = "online-def-resp-batch"
		TICKET_NUM = "REQ_250453"
		}
	agent {
		label 'linux'
		}
	tools {
		jdk 'JDK 8u212'
		}
	stages {
		stage ('GIT Checkout') {
			steps {
				git branch: '${BRANCH_NAME}', url: "${GIT_REPO}",
				credentialsId: 'SDA'
			}
		}

		stage ('Initialize') {
			environment {
				ARTIFACT_NAME = readMavenPom().getArtifactId()
				ARTIFACT_VERSION = readMavenPom().getVersion()
			}
			steps {
				parallel(
					a: {
						step([$class: 'RunGlobalProcessNotifier',
						// DA server to publish to, configured in global settings.
						siteName: 'SDA',
						// Use a build parameter to determine if a global process should be triggered in DA.
						runGlobalProcessIf: 'true',
						// Wait for a completion of a global process in DA and update job status based on the process result.
						updateJobStatus: true,
						// The name of the global process you want to execute.
						globalProcessName: 'Component Creator',
						// The name of the resource you want to execute the global process on.
						resourceName: 'putllxscript10',
						// Newline separated list of quoted properties and values i.e. prop1=${BUILD_NUMBER} to pass to DA global process.
						globalProcessProperties: """
						component.type=aws_Docker_
						component.name=${ARTIFACT_NAME}
						component.template=MDW - AWS: Docker - deployment_template
						component.description=AWS Fargate Deployment
						application.container=MDW - Cloud
						resource.container=DEV - AWS
						build.type=fargate
						"""
						])
					},
					b: {
						rtServer (
						id: "ARTIFACTORY_SERVER",
						url: 'https://capartifactory.jfrog.io/capartifactory',
						credentialsId: 'artifactory.creds'
						)
						rtMavenDeployer (
						id: "MAVEN_DEPLOYER",
						serverId: "ARTIFACTORY_SERVER",
						releaseRepo: "maven.cap.lib",
						snapshotRepo: "maven.cap.lib"
						)
						rtMavenResolver (
						id: "MAVEN_RESOLVER",
						serverId: "ARTIFACTORY_SERVER",
						releaseRepo: "libs-release",
						snapshotRepo: "libs-snapshot"
						)
					}
				)
			}
		}

		stage ('Maven Build') {
			steps {
				withSonarQubeEnv('SonarQube') {
				rtMavenRun (
					tool: 'Maven 3.6.1', // Tool name from Jenkins configuration
					pom: 'pom.xml',
					goals: 'clean install dockerfile:build sonar:sonar pmd:pmd pmd:cpd findbugs:findbugs checkstyle:checkstyle com.github.spotbugs:spotbugs-maven-plugin:4.0.0:spotbugs',
					deployerId: "MAVEN_DEPLOYER",
					resolverId: "MAVEN_RESOLVER"
					)
				}
			}
		}

		stage ('Store Image and Push Properties') {
			environment {
				ARTIFACT_NAME = readMavenPom().getArtifactId()
				ARTIFACT_VERSION = readMavenPom().getVersion()
			}
			steps {
				parallel(
					a: {
						echo "$ARTIFACT_NAME"
						echo "$ARTIFACT_VERSION"
						sh '''
						cp jobdef.json target/
						docker save -o $ARTIFACT_NAME.BUILD_$BUILD_NUMBER.tar $ARTIFACT_NAME
						gzip $ARTIFACT_NAME.BUILD_$BUILD_NUMBER.tar
						mv $ARTIFACT_NAME.BUILD_$BUILD_NUMBER.tar.gz target/
						'''
					},
					b: {
						step([$class: 'RunGlobalProcessNotifier',
						// DA server to publish to, configured in global settings.
						siteName: 'SDA',
						// Use a build parameter to determine if a global process should be triggered in DA.
						runGlobalProcessIf: 'true',
						// Wait for a completion of a global process in DA and update job status based on the process result.
						updateJobStatus: true,
						// The name of the global process you want to execute.
						globalProcessName: 'Component Property Injector',
						// The name of the resource you want to execute the global process on.
						resourceName: 'putllxscript10',
						// Newline separated list of quoted properties and values i.e. prop1=${BUILD_NUMBER} to pass to DA global process.
						globalProcessProperties: """
						component.name=aws_Docker_${ARTIFACT_NAME}
						build.number=${BUILD_NUMBER}
						branch.name=${BRANCH_NAME}
						build.name=${ARTIFACT_NAME}.BUILD_${BUILD_NUMBER}
						aws.cluster.name=${ARTIFACT_NAME}
						aws.image.name=${ARTIFACT_NAME}
						docker.image.name=${ARTIFACT_NAME}
						file.name=${ARTIFACT_NAME}
						"""
						])
					}
				)
			}
		}

		stage ('Code Analysis') {
			steps {
				recordIssues(
					enabledForFailure: false, aggregatingResults: true, healthy: 10, unhealthy: 100, minimumSeverity: 'HIGH',
					tools: [checkStyle(), cpd(), spotBugs(), findBugs()]
				)
			}
		}

		stage ('SonarQube Verification') {
			steps {
				timeout(time: 2, unit: 'HOURS') {
				waitForQualityGate abortPipeline: true
				}
			}
		}

		stage ('Publish Build Info') {
			steps {
				rtPublishBuildInfo (
					serverId: "ARTIFACTORY_SERVER"
				)
			}
		}

		stage ('SDA Deployment') {
			environment {
				ARTIFACT_NAME = readMavenPom().getArtifactId()
				ARTIFACT_VERSION = readMavenPom().getVersion()
			}
			steps {
				step([$class: 'SerenaDAPublisher',
				// DA server to publish to, configured in global settings.
				siteName: 'SDA',
				// The name of the component in the DA server.
				component: 'aws_Docker_' + env.ARTIFACT_NAME,
				// Base directory where the artifacts are located.
				baseDir: env.WORKSPACE + '/target',
				// The name of the version in the DA server.
				versionName: env.BRANCH_NAME + ':' + env.BUILD_NUMBER,
				// A new line separated list of file filters to select the files to publish.
				fileIncludePatterns: '''
				*.gz
				jobdef.json
				''',
				// A new line separated list of file filters to exclude from publishing.
				fileExcludePatterns: '''
				**/.original
				**/*tmp*
				**/.git
				''',
				// Skip publishing (e.g. temporarily)
				skip: false,
				// Add status to this version in DA once it's uploaded.
				addStatus: true,
				// The full name of the status to apply to the Version.
				statusName: 'CICD',
				// Trigger a deployment of this version in DA once it's uploaded.
				deploy: true,
				// Use a build parameter to determine if a application process should be triggered in DA.
				deployIf: 'true',
				// Wait for a completion of an application process in DA and update job status based on the process result.
				deployUpdateJobStatus: true,
				// The name of the application in DA which will be used to deploy the version.
				deployApp: 'MDW - Cloud',
				// The name of the environment in DA to deploy to.
				deployEnv: 'MDW - DEV',
				// The name of the application process in DA which will be used to deploy the version.
				deployProc: 'AWS - Docker / Fargate Deployment',
				])
			}
		}
	}

	post {
		always {
			logstashSend failBuild: false, maxLines: 1000
			emailext attachLog: true,
			body: "${currentBuild.currentResult}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}",
			recipientProviders: [developers(), requestor()],
			subject: "Jenkins Build ${currentBuild.currentResult}: Job ${env.JOB_NAME}"
			}
		success {
			addBadge(icon: "success.gif", text: "Build Succeeded")
			sh '''
			cd $WORKSPACE/target
			rm -rfv *.BUILD_$BUILD_NUMBER.tar.gz
			'''
			}
		failure {
			addBadge(icon: "error.gif", text: "Build Failed")
			echo 'If ALL stages are Green then the DA Deployment step most likely caused this failure'
		}
		unstable {
			echo 'DA Deployment most Likely Caused This Failure'
		}
	}
}
