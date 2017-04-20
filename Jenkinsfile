#!/usr/bin/env groovy

node('docker') {

    def image = docker.image('bayesian/cucos-worker')
    def commitId

    stage('Checkout') {
        checkout scm
        commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    }

    stage('Build') {
        dockerCleanup()
        docker.build(image.id, '--pull --no-cache .')
        sh "docker tag ${image.id} docker-registry.usersys.redhat.com/${image.id}"
        docker.build('cucos-lib-tests', '-f Dockerfile.tests .')
    }

    stage('Tests') {
        timeout(30) {
            sh './runtest.sh'
        }
    }

    stage('Integration Tests') {
        ws {
            docker.withRegistry('https://docker-registry.usersys.redhat.com/') {
                docker.image('bayesian/bayesian-api').pull()
                docker.image('bayesian/coreapi-jobs').pull()
                docker.image('bayesian/coreapi-pgbouncer').pull()
                docker.image('bayesian/coreapi-downstream-data-import').pull()
            }

            git url: 'https://github.com/baytemp/common.git', branch: 'master', credentialsId: 'baytemp-ci-gh'
            dir('integration-tests') {
                timeout(30) {
                    sh './runtest.sh'
                }
            }
        }
    }

    if (env.BRANCH_NAME == 'master') {
        stage('Push Images') {
            docker.withRegistry('https://docker-registry.usersys.redhat.com/') {
                image.push('latest')
                image.push(commitId)
            }
            docker.withRegistry('https://registry.devshift.net/') {
                image.push('latest')
                image.push(commitId)
            }
        }

        stage('Prepare Template') {
            dir('openshift') {
                sh "sed -i \"/image-tag\$/ s|latest|${commitId}|\" template.yaml"
                stash name: 'template', includes: 'template.yaml'
                archiveArtifacts artifacts: 'template.yaml'
            }
        }
    }
}

if (env.BRANCH_NAME == 'master') {
    node('oc') {
        stage('Deploy - dev') {
            unstash 'template'
            sh 'oc --context=dev process -v DEPLOYMENT_PREFIX=DEV -v WORKER_ADMINISTRATION_REGION=ingestion -v S3_BUCKET_FOR_ANALYSES=DEV-bayesian-core-data -f template.yaml | oc --context=dev apply -f -'
            sh 'oc --context=dev process -v DEPLOYMENT_PREFIX=DEV -v WORKER_ADMINISTRATION_REGION=api -v S3_BUCKET_FOR_ANALYSES=DEV-bayesian-core-data -f template.yaml | oc --context=dev apply -f -'
        }

        stage('Deploy - rh-idev') {
            unstash 'template'
            sh 'oc --context=rh-idev process -v DEPLOYMENT_PREFIX=STAGE -v WORKER_ADMINISTRATION_REGION=ingestion -v S3_BUCKET_FOR_ANALYSES=STAGE-bayesian-core-data -f template.yaml | oc --context=rh-idev apply -f -'
            sh 'oc --context=rh-idev process -v DEPLOYMENT_PREFIX=STAGE -v WORKER_ADMINISTRATION_REGION=api -v S3_BUCKET_FOR_ANALYSES=STAGE-bayesian-core-data -f template.yaml | oc --context=rh-idev apply -f -'
        }
    }
}
