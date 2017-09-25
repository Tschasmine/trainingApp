@Library('esdk-jenkins-lib@master') _

def version = ""
node {
	timestamps {
		ansiColor('xterm') {
			try {
				properties([parameters([
						string(defaultValue: '', description: 'Version of ESDK to use (if not same as project version, project version will be updated as well)', name: 'ESDK_VERSION'),
						string(defaultValue: 'anonymous', description: 'User who triggered the build implicitly (through a commit in another project)', name: 'BUILD_USER_PARAM'),
						string(defaultValue: '2016r4n13', description: 'abas Essentials version', name: 'ERP_VERSION')
					])
				])
				stage('Setup') {
					prepareEnv()
					String esdkInMavenLocal = '$HOME/.m2/repository/​de/abas/esdk'
					sh "if [ -e '$esdkInMavenLocal' -a -d '$esdkInMavenLocal' ]; then rm -rf $esdkInMavenLocal ; fi"
					checkout scm
					sh "git clean -f"
					sh "git reset --hard origin/$BRANCH_NAME"
					shInstallDockerCompose()
					currentBuild.description = "ERP Version: ${params.ERP_VERSION}"
					initGradleProps()
					showGradleProps()
				}
				stage('Preparation') { // for display purposes
					withCredentials([usernamePassword(credentialsId: '82305355-11d8-400f-93ce-a33beb534089',
							passwordVariable: 'MAVENPASSWORD', usernameVariable: 'MAVENUSER')]) {
						shDocker('login intra.registry.abas.sh -u $MAVENUSER -p $MAVENPASSWORD')
					}
					withEnv(["ERP_VERSION=${params.ERP_VERSION}"]) {
						shDockerComposeUp()
					}
					sleep 30
				}
				stage('Set version') {
					shGradle("--version")
					version = readVersion()
					println("version: $version")
					println("esdkVersion: $params.ESDK_VERSION")
					if (params.ESDK_VERSION.matches("[0-9]+\\.[0-9]+\\.[0-9]+(-SNAPSHOT)?") && (version != params.ESDK_VERSION)) {
						currentBuild.description = currentBuild.description + " ESDK Version: ${params.ESDK_VERSION}"
						println("Builduser: ${params.BUILD_USER_PARAM}")
						justReplace(version, params.ESDK_VERSION, "gradle.properties.template")
						if (params.ESDK_VERSION.endsWith("-SNAPSHOT")) {
							shGitCommitSnapshot("gradle.properties.template", params.ESDK_VERSION, params.BUILD_USER_PARAM)
						} else {
							shGitCommitRelease("gradle.properties.template", params.ESDK_VERSION, params.BUILD_USER_PARAM, env.BUILD_ID)
							sh 'git branch --force release HEAD'
						}
						withCredentials([usernamePassword(credentialsId: '44e7bb41-f9fc-483f-9e66-9751c0163d37', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USER')]) {
							shGitPushIntoMaster("https://$GIT_USER:$GIT_PASSWORD@github.com/Tschasmine/trainingApp.git", params.ESDK_VERSION)
						}
					}
				}
				stage('Installation') {
					shGradle("checkPreconditions -x importKeys")
					shGradle("publishHomeDirJars")
					shGradle("fullInstall")
				}
				stage('Verify') {
					shGradle("verify")
				}
				onMaster {
					stage('Publish') {
						shGradle("createAppJar")
						shGradle("publish")
					}
				}
				currentBuild.description = currentBuild.description + " => successful"
			} catch (any) {
				any.printStackTrace()
				currentBuild.result = 'FAILURE'
				currentBuild.description = currentBuild.description + " => failed"
				throw any
			} finally {
				shDockerComposeCleanUp()

				junit allowEmptyResults: true, testResults: 'build/test-results/**/*.xml'
				archiveArtifacts 'build/reports/**'

				slackNotify(currentBuild.result)
			}
		}
	}
}
