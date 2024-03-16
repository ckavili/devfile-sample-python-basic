pipeline {
	agent {
		label "master"
	}
  environment {
		// GLobal Vars shared across all jenkins agents

		// Job name contains the branch eg pet-battle-feature%2Fjenkins-123
		// ensure the name is k8s compliant
		// JOB_NAME = "${JOB_NAME}".replace("%2F", "-").replace("/", "-")
		// NAME = "${JOB_NAME}".split("/")[0]
		GIT_SSL_NO_VERIFY = true

		// ArgoCD Config Repo
		// set this as an ENV_VAR on Jenkins to make this easier?
        // ARGOCD_CONFIG_REPO = "github.com/petbattle/ubiquitous-journey.git"


		// Credentials bound in OpenShift
		GIT_CREDS = credentials("${OPENSHIFT_BUILD_NAMESPACE}-github-user")
		NEXUS_CREDS = credentials("${OPENSHIFT_BUILD_NAMESPACE}-nexus-password")

		// Nexus Artifact repo 
		NEXUS_REPO_NAME="labs-static"
		NEXUS_REPO_HELM = "helm-charts"
	}

	// The options directive is for configuration that applies to the whole job.
	options {
		buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '1'))
		timeout(time: 15, unit: 'MINUTES')
		ansiColor('xterm')
	}

	stages {
		stage('🗒️ Prepare Environment') {
			failFast true
			parallel {
				stage("📝 Release Build") {
					options {
						skipDefaultCheckout(true)
					}
					agent { label "master" }
					when {
						expression { GIT_BRANCH.startsWith("master") || GIT_BRANCH.startsWith("main") }
					}
					steps {
						script {
							// ensure the name is k8s compliant
							env.NAME = "${JOB_NAME}".split("/")[0]
							env.APP_NAME = "python-example"
							env.DESTINATION_NAMESPACE = "labs-test"
                            env.VERSION = "1.0.0"
							// External image push registry info
							env.QUAY_PUSH_SECRET = "petbattle-jenkinspb-pull-secret"
							env.IMAGE_NAMESPACE = env.QUAY_ACCOUNT != null ? "${QUAY_ACCOUNT}" : "petbattle"
							env.IMAGE_REPOSITORY = "quay.io"
              env.ARGOCD_CONFIG_REPO = "${ARGOCD_CONFIG_REPO}"
						}
            sh 'printenv'
					}
				}
				stage("📝 Sandbox Build") {
					options {
						skipDefaultCheckout(true)
					}
					agent {
							node {
									label "master"
							}
					}
					when {
						expression { return !(GIT_BRANCH.startsWith("master") || GIT_BRANCH.startsWith("main") )}
					}
					steps {
						script {
							env.DESTINATION_NAMESPACE = "labs-dev"
							env.IMAGE_NAMESPACE = "${DESTINATION_NAMESPACE}"
							env.IMAGE_REPOSITORY = 'image-registry.openshift-image-registry.svc:5000'

							// ammend the name to create 'sandbox' deploys based on current branch
							env.NAME = "${JOB_NAME}".split("/")[0]
							env.APP_NAME = "python-example"
                            env.VERSION = "1.0.0"
							env.DEV_BUILD = true
						}
					}
				}
			}
		}


		stage("🧁 Bake (OpenShift Build)") {
			options {
					skipDefaultCheckout(true)
			}
			agent { label "master" }
			steps {
				sh 'printenv'
				
				echo '### Run OpenShift Build ###'
				sh '''
					BUILD_ARGS=" --build-arg git_commit=${GIT_COMMIT} --build-arg git_url=${GIT_URL}  --build-arg build_url=${RUN_DISPLAY_URL} --build-arg build_tag=${BUILD_TAG}"
					# if we're targeting master branch or main branch push image to external repo, 
					# if not just use the internal registry

						echo "🏗 Creating a sandbox build for inside the cluster 🏗"
						oc new-build --binary --name=${APP_NAME} -l app=${APP_NAME} ${BUILD_ARGS} --strategy=docker
						oc start-build ${APP_NAME} --from-dir=. ${BUILD_ARGS} --follow --wait
						# used for internal sandbox build ....
						oc tag ${OPENSHIFT_BUILD_NAMESPACE}/${APP_NAME}:latest ${DESTINATION_NAMESPACE}/${APP_NAME}:${VERSION}
				'''
			}
		}

	
	}
}