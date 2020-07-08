#!/usr/bin/env groovy

def confluent_package_version = string(name: 'CONFLUENT_PACKAGE_VERSION',
    defaultValue: '',
    description: 'Confluent Version to install and test'
)

def confluent_common_repository_baseurl = string(name: 'CONFLUENT_PACKAGE_BASEURL',
    defaultValue: '',
    description: 'Packaging Base URL from where to download packages (ie: https://packages.confluent.io)'
)

def confluent_release_quality = choice(name: 'CONFLUENT_RELEASE_QUALITY',
    choices: ['prod', 'snapshot'],
    defaultValue: 'prod',
    description: 'Determines the release extention used for testing ("prod" for public releases, "snapshot" for nightly builds)',
)

def config = jobConfig {
    nodeLabel = 'docker-oraclejdk8'
    slackChannel = '#ansible-eng'
    timeoutHours = 4
    runMergeCheck = false
    properties = [parameters([confluent_package_version, confluent_common_repository_baseurl, confluent_release_quality])]
}

def job = {
    // Temp disable for testing
    /*stage('Install Molecule and Latest Ansible') {
        sh '''
            sudo pip install --upgrade 'ansible==2.9.*'
            sudo pip install molecule docker
        '''
    }*/

    def rpm_suffix = ''
    def deb_suffix = ''
    switch(config.confluent_release_quality) {
        case "prod":
            rpm_suffix = "${parans.CONFLUENT_PACKAGE_VERSION}-1"
            deb_suffix = "${parans.CONFLUENT_PACKAGE_VERSION}-1"
        break
        case "snapshot":
            rpm_suffix = "${parans.CONFLUENT_PACKAGE_VERSION}-0.1.SNAPSHOT"
            deb_suffix = "${parans.CONFLUENT_PACKAGE_VERSION}~SNAPSHOT-1"
        break
        default:
            error("Unknown release quality ${config.confluent_release_quality}")
        break
    }

    writeFile file: "roles/confluent.test/base-config.yml" text: """---
provisioner:
  inventory:
    group_vars:
      all:
        confluent_common_repository_baseurl: "${params.CONFLUENT_PACKAGE_BASEURL}"
        confluent_package_version: "${parans.CONFLUENT_PACKAGE_VERSION}"
        confluent_package_redhat_suffix: "-${rpm_suffix}"
        confluent_package_debian_suffix: "=${deb_suffix}"
        bootstrap: false
"""

    }

    withDockerServer([uri: dockerHost()]) {
        stage('Plaintext') {
            sh '''
                cd roles/confluent.test
                cat base-config.yml
            '''
        }
    }
}

runJob config, job
