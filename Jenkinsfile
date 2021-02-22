#!/usr/bin/env groovy
@Library('sec_ci_libs@v2-latest') _

master_branches = ['release-v3.12-d2iq', ] as String[]
slack_creds_id = '8b793652-f26a-422f-a9ba-0d1e47eb9d89'
slack_channel = '#dcos-networking-ci'
aws_credentials_id = 'eng-mesosphere-dcos-Jenkins-PowerUser'
github_credentials_id = 'd146870f-03b0-4f6a-ab70-1d09757a51fc'
github_repo = 'dcos/calicoctl'

build_days_to_keep = 7

prometheus_url = 'https://dcosmon.mesosphere.com'
container_name = ''


def store_image_name() {
    container_name = sh(
        returnStdout: true,
        script: 'echo ${CONTAINER_NAME:=${MESOS_CONTAINER_NAME:=""}}'
    ).trim()
}

def post_comment(issue, body) {
    withCredentials([
        string(
            credentialsId: github_credentials_id,
            variable: "GITHUB_TOKEN")
    ]) {
        def url = 'https://api.github.com/repos/' +
                  "${github_repo}/issues/${issue}/comments"
        def payload = JsonOutput.toJson(["body": body])

        def headers = [
            "Authorization": "Token ${env.GITHUB_TOKEN}",
            "Accept": "application/json",
            "Content-type": "application/json"
        ]

        def command = "curl -s -X POST -d '${payload}' "
        headers.each {
            command += "-H '${it.key}: ${it.value}' "
        }

        command += url

        docker.image(docker_curl_image).inside {
            sh(command)
        }
    }
}

def post_install_instructions() {

    if (!env.CHANGE_ID) {
        return
    }

    def message = [
        '# Success!',
        '',
        '## Download Links:',
        '',
        "  - [calicoctl](${edgelb_repo_url})",
    ].join('\n')

    post_comment(env.CHANGE_ID, message)
}

def publish_vanity() {
    def copy = { String prefix->
        sh('aws s3 cp --acl public-read ' +
           "./bin/calicoctl-darwin-amd64 " +
           "s3://dcos-calicoctl-artifacts/${prefix}/bin/calicoctl-darwin-amd64"
        )
        sh('aws s3 cp --acl public-read ' +
           "./bin/calicoctl-linux-amd64 " +
           "s3://dcos-calicoctl-artifacts/${prefix}/bin/calicoctl-linux-amd64"
        )
        sh('aws s3 cp --acl public-read ' +
           "./bin/calicoctl-windows-amd64.exe " +
           "s3://dcos-calicoctl-artifacts/${prefix}/bin/calicoctl-windows-amd64.exe"
        )
    }

    if (env.IS_RELEASE == 'true') {
        // Beware that tags are not automatically build, as per:
        // https://stackoverflow.com/a/48276262
        // one needs to also trigger them manually
        publish_release_artifacts()
        return
    }

    if (env.CHANGE_ID) {
        copy("autodelete7d/pr/${env.CHANGE_ID}")
    } else if (master_branches.contains(env.BRANCH_NAME)) {
        copy("autodelete7d/${env.BRANCH_NAME}")
    }
}

def publish_release_artifacts() {
    // use first sevent chars of git commit hash and the first seven of the
    // sha-sum of the resulting sha-sums of the binaries:
    gitCommitSha = sh(
        returnStdout: true,
        script: 'git rev-parse HEAD'
    ).trim()
    gitCommitShortSha = gitCommitSha.take(7)

    binariesSha = sh(
        returnStdout: true,
        script: 'sha256sum ./bin/* | sort | sha256sum'
    ).trim()
    binariesShortSha = binariesSha.take(7)
    calicoctlSha = "${gitCommitShortSha}-${binariesShortSha}"
    sh('aws s3 cp --acl bucket-owner-full-control ' +
       "./bin/calicoctl-darwin-amd64 " +
       "s3://downloads.mesosphere.io/dcos-calicoctl/bin/${env.TAG_NAME}/${calicoctlSha}/calicoctl-darwin-amd64"
    )
    sh('aws s3 cp --acl bucket-owner-full-control ' +
       "./bin/calicoctl-linux-amd64 " +
       "s3://downloads.mesosphere.io/dcos-calicoctl/bin/${env.TAG_NAME}/${calicoctlSha}/calicoctl-linux-amd64"
    )
    sh('aws s3 cp --acl bucket-owner-full-control ' +
       "./bin/calicoctl-windows-amd64.exe " +
       "s3://downloads.mesosphere.io/dcos-calicoctl/bin/${env.TAG_NAME}/${calicoctlSha}/calicoctl-windows-amd64.exe"
    )
}

def with_aws(String credentials_id, Closure body) {
    withCredentials([[
        $class: 'AmazonWebServicesCredentialsBinding',
        credentialsId: credentials_id,
        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
    ]]) {
        body()
    }
}

pipeline {
    agent {
        node {
            label 'mesos-ec2-ubuntu-18.04'
        }
    }

    environment {
        IS_RELEASE = "${env.TAG_NAME ? 'true' : 'false'}"
    }

    options {
        buildDiscarder(
            logRotator(
                daysToKeepStr: "${build_days_to_keep}",
                artifactDaysToKeepStr: "${build_days_to_keep}")
        )

        preserveStashes(buildCount: 20)
        timestamps()
    }

    triggers {
        cron(
            master_branches.contains(env.BRANCH_NAME) ? '@daily' : ''
        )
    }

    post {
        always {
            echo 'To find the monitoring metrics of this CI building, ' +
                 "please refer to ${prometheus_url} by container name " +
                 "${container_name}"
            echo 'For example, memory usage over time is exhibited in ' +
                 "${prometheus_url}/graph?g0.range_input=4h&" +
                 'g0.expr=container_memory_usage_bytes%7Bname%3D~' +
                 "\"${container_name}\"%7D&g0.tab=0"
        }

        cleanup {
            deleteDir()
        }

        failure {
            script {
                msg = "Build number `${currentBuild.number}` failed."
                msg_body = "Build url: ${env.BUILD_URL}"
                color = 'danger'
                prefix = '[FAILURE]'

                if (master_branches.contains(env.BRANCH_NAME)) {
                    withCredentials(
                            [[$class: 'StringBinding',
                            credentialsId: slack_creds_id,
                            variable: 'SLACK_TOKEN']]) {
                        slackSend(
                            channel: slack_channel,
                            message: "`${env.JOB_NAME}` ${msg}\n\n${msg_body}",
                            teamDomain: 'mesosphere',
                            token: "${env.SLACK_TOKEN}",
                            color: color)
                    }
                }
            }
        }

        success {
            script {
                post_install_instructions()
            }
        }
    }

    stages {
        stage('verify author') {
            when {
                changeRequest()
            }
            steps {
                user_is_authorized(master_branches, slack_creds_id, slack_channel)
            }
        }

        stage('clean') {
            steps {
                sh 'make clean'
            }
        }
        stage('build') {
            steps {
                store_image_name()

                sh 'curl https://github.com/tcnksm/ghr/releases/download/v0.13.0/ghr_v0.13.0_linux_amd64.tar.gz | tar -zxv -C /usr/local/bin --strip-components=1 ghr_v0.13.0_linux_amd64/ghr'

                // Supplying all targets at once speeds things up with
                // calico's build container:
                sh 'make bin/calicoctl-linux-amd64 bin/calicoctl-darwin-amd64 bin/calicoctl-windows-amd64.exe'

                stash([
                    name: 'calicoctl_binaries',
                    includes: [
                        'bin/calicoctl-linux-amd64',
                        'bin/calicoctl-darwin-amd64',
                        'bin/calicoctl-windows-amd64.exe',
                    ].join(','),
                ])
            }
        }

        stage('publish') {
            steps {
                store_image_name()

                unstash 'calicoctl_binaries'

                // Confirm that we actually have something:
                sh 'ls -l ./bin/'
                sh './bin/calicoctl-linux-amd64 version || true'

                with_aws(aws_credentials_id) {
                    publish_vanity()
                }
            }
        }
    }
}
