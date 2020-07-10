#!/usr/bin/env groovy

import static groovy.json.JsonOutput.*

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
    stage('Install Molecule and Latest Ansible') {
        sh '''
            sudo pip install --upgrade 'ansible==2.9.*'
            sudo pip install molecule docker
        '''
    }

    def override_config = [:]

    if(params.CONFLUENT_PACKAGE_BASEURL) {
        override_config['confluent_common_repository_baseurl'] = params.CONFLUENT_PACKAGE_BASEURL
    }

    if(params.CONFLUENT_PACKAGE_VERSION) {
        override_config['confluent_package_version'] = params.CONFLUENT_PACKAGE_VERSION

        if(confluent_release_quality != 'prod') {
            switch(params.CONFLUENT_RELEASE_QUALITY) {
                case "snapshot":
                    override_config['confluent_package_redhat_suffix'] = "-${params.CONFLUENT_PACKAGE_VERSION}-0.1.SNAPSHOT"
                    override_config['confluent_package_debian_suffix'] = "=${params.CONFLUENT_PACKAGE_VERSION}~SNAPSHOT-1"
                break
                default:
                    error("Unknown release quality ${params.CONFLUENT_RELEASE_QUALITY}")
                break
            }
        }
    }

    def molecule_args = ""
    if(override_config) {
        override_config['bootstrap'] = false
        def base_config = [
            'provisioner': [
                'inventory': [
                    'group_vars': [
                        'all': override_config
                    ]
                ]
            ]
        ]
        echo "Overriding Ansible vars for testing with base-config:\n" + prettyPrint(toJson(override_config))

        writeYaml file: "roles/confluent.test/base-config.yml", data: base_config

        molecule_args = "--base-config base-config.yml"
    }

    withDockerServer([uri: dockerHost()]) {
        stage('Plaintext') {
            sh """
cd roles/confluent.test
molecule ${molecule_args} test -s plaintext-rhel
            """
        }
    }
}

runJob config, job
