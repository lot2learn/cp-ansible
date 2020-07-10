#!/usr/bin/env groovy

// XXX: Temp until https://github.com/confluentinc/jenkins-common/pull/272 is merged

import com.cloudbees.groovy.cps.NonCPS

@NonCPS
def getLastSuccessfulBuildNumber(jobName) {
    /* Find the last successful build for a job and return the build number */
    def job = Jenkins.instance.getItemByFullName(jobName)
    def build = job.getLastSuccessfulBuild()
    return build.getNumber().toString()
}

@NonCPS
def getMajorVersion(version) {
    /*  Find the target branch for a job and return the major branch version
        Eg: for 5.4.x targetBranch, returns 5.4
    */
    def errMsg = "Invalid version specified"
    try {
        def versionMatcher = version =~/v?(\d+).(\d+).(x|\d+)/
        assert versionMatcher[0][1].isNumber()
        assert versionMatcher[0][2].isNumber()
        return version.tokenize(".")[0..1].join(".")
    }
    catch(IndexOutOfBoundsException ex) {
        println(errMsg)
        throw ex
    }
    catch(Exception ex) {
        throw ex
    }
}
// XXX: Temp until https://github.com/confluentinc/jenkins-common/pull/272 is merged

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
        def base_config = [
            'provisioner': [
                'inventory': [
                    'group_vars': [
                        'all': override_config
                    ]
                ]
            ]
        ]
        /*def base_config = """---
provisioner:
  inventory:
    group_vars:
      all:
        confluent_common_repository_baseurl: "${params.CONFLUENT_PACKAGE_BASEURL}"
        confluent_package_version: "${params.CONFLUENT_PACKAGE_VERSION}"
        confluent_package_redhat_suffix: "-${rpm_suffix}"
        confluent_package_debian_suffix: "=${deb_suffix}"
        bootstrap: false
"""
        writeFile file: "roles/confluent.test/base-config.yml", text: base_config*/

        echo "Optional parameters specified, overriding with base-config:\n${base_config}"

        writeYaml file: "roles/confluent.test/base-config.yml", data: base_config

        molecule_args = "--base-config base-config.yml"
    }

    withDockerServer([uri: dockerHost()]) {
        stage('Plaintext') {
            sh """
cd roles/confluent.test
if [ -f base-config.yml ]; then
    cat base-config.yml
fi
echo molecule ${molecule_args} test -s plaintext-rhel
            """
        }
    }
}

runJob config, job
