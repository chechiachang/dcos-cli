#!/usr/bin/env groovy

// These are the platforms we are building against.
def platforms = ["linux", "mac", "windows"]

// This generates the launchConfig for a particular build.
def generateConfig(deploymentName, templateUrl) {
    return """
---
launch_config_version: 1
deployment_name: ${deploymentName}
template_url: ${templateUrl}
provider: aws
aws_region: us-west-1
template_parameters:
    KeyName: default
    AdminLocation: 0.0.0.0/0
    PublicSlaveInstanceCount: 1
    SlaveInstanceCount: 1
"""
}


// This class abstracts away the functions required to create a test cluster
// for each of our platforms.
class TestCluster implements Serializable {
    WorkflowScript script
    String platform
    int createAttempts

    TestCluster(WorkflowScript script, String platform) {
        this.script = script
        this.platform = platform
        this.createAttempts = 0
    }

    def _create() {
        script.sh "./dcos-launch create -c ${this.platform}_config.yaml -i ${this.platform}_cluster_info.json"
    }

    def destroy() {
        script.sh "./dcos-launch delete -i ${this.platform}_cluster_info.json"
    }

    def _block() {
        script.sh "./dcos-launch wait -i ${this.platform}_cluster_info.json"
    }

    def create() {
        script.sh "rm -rf ${this.platform}_config.yaml"
        script.sh "rm -rf ${this.platform}_cluster_info.json"

        script.writeFile([
            "file": "${this.platform}_config.yaml",
            "text" : script.generateConfig(
                "dcos-cli-${this.platform}-${script.env.BRANCH_NAME}-${script.env.BUILD_ID}-${this.createAttempts}",
                "${script.env.CF_TEMPLATE_URL}")])

        _create()

        createAttempts++
    }

    def block() {
        while (true) {
            try {
                _block()
                break
            } catch(Exception e) {
                destroy()
                if (e instanceof InterruptedException) {
                    break
                }
                create()
            }
        }
    }

    def dcosUrl() {
        script.sh """
            ./dcos-launch describe -i ${this.platform}_cluster_info.json \
            | python -c \
                'import sys, json; \
                 contents = json.load(sys.stdin); \
                 print(contents["masters"][0]["public_ip"], end="")' \
            > ${this.platform}_dcos_url"""

        return script.readFile("${this.platform}_dcos_url")
    }

    def acsToken() {
        def dcosUrl = this.dcosUrl()

        script.sh """
            python -c \
                'import requests; \
                 requests.packages.urllib3.disable_warnings(); \
                 js={"uid":"${script.env.DCOS_ADMIN_USERNAME}", \
                     "password": "${script.env.DCOS_ADMIN_PASSWORD}"}; \
                 r=requests.post("http://${dcosUrl}/acs/api/v1/auth/login", \
                                 json=js, \
                                 verify=False); \
                 print(r.json()["token"], end="")' \
            > ${this.platform}_acs_token"""

        return script.readFile("${this.platform}_acs_token")
    }
}


// This function returns a closure that prepares binary builds for
// a specific platform on a specific node in a specific workspace.
def binaryBuilder(String platform, String nodeId, String workspace = null) {
    return { Closure _body ->
        def body = _body

        return {
            node(nodeId) {
                if (!workspace) {
                    workspace = "${env.WORKSPACE}"
                }

                ws (workspace) {
                    stage ('Cleanup workspace') {
                        deleteDir()
                    }

                    stage ("Unstash dcos-cli repository") {
                        unstash('dcos-cli')
                    }

                    body()
                }
            }
        }
    }
}


// This function returns a closure that prepares a test environment for
// a specific platform on a specific node in a specific workspace.
def testBuilder(String platform, String nodeId, String workspace = null) {
    return { Closure _body ->
        def body = _body

        return {
            def cluster = new TestCluster(this, platform)

            try {
                stage ("Create ${platform} cluster") {
                    cluster.create()
                }

                stage ("Wait for ${platform} cluster") {
                    cluster.block()
                }

                def dcosUrl = cluster.dcosUrl()
                def acsToken = cluster.acsToken()

                node(nodeId) {
                    if (!workspace) {
                        workspace = "${env.WORKSPACE}"
                    }

                    ws (workspace) {
                        stage ('Cleanup workspace') {
                            deleteDir()
                        }

                        stage ("Unstash dcos-cli repository") {
                            unstash('dcos-cli')
                        }

                        withCredentials(
                            [[$class: 'FileBinding',
                             credentialsId: '1c206779-acc0-4844-97f6-7b3ed081a456',
                             variable: 'DCOS_SNAKEOIL_CRT_PATH'],
                            [$class: 'FileBinding',
                             credentialsId: '23743034-1ac4-49f7-b2e6-a661aee2d11b',
                             variable: 'CLI_TEST_SSH_KEY_PATH']]) {

                            withEnv(["DCOS_URL=${dcosUrl}",
                                     "DCOS_ACS_TOKEN=${acsToken}"]) {
                                body()
                            }
                        }

                    }
                }
            } finally {
                stage ("Destroy ${platform} cluster") {
                    try { cluster.destroy() }
                    catch (Exception e) {}
                }
            }
        }
    }
}

// These are the builds that can be run in parallel.
def builders = [:]


builders['linux-binary'] = binaryBuilder('linux', 'py35', '/workspace')({
    stage ("Build dcos-cli binary") {
        dir('dcos-cli/cli') {
            sh "make binary"
            sh "dist/dcos"
        }
    }
})


builders['mac-binary'] = binaryBuilder('mac', 'mac')({
    stage ("Build dcos-cli binary") {
        dir('dcos-cli/cli') {
            sh "make binary"
            sh "dist/dcos"
        }
    }
})


builders['windows-binary'] = binaryBuilder('windows', 'windows')({
    stage ("Build dcos-cli binary") {
        dir('dcos-cli/cli') {
            bat 'bash -c "make binary"'
            bat 'dist\\dcos.exe'
        }
    }
})


builders['linux-tests'] = testBuilder('linux', 'py35', '/workspace')({
    stage ("Run dcos-cli tests") {
        sh '''
           rm -rf ~/.dcos; \
           grep -q "^.* dcos.snakeoil.mesosphere.com$" /etc/hosts && \
           sed -iold "s/^.* dcos.snakeoil.mesosphere.com$/${DCOS_URL} dcos.snakeoil.mesosphere.com/" /etc/hosts || \
           echo ${DCOS_URL} dcos.snakeoil.mesosphere.com >> /etc/hosts'''

        dir('dcos-cli/cli') {
            sh '''
               export PYTHONIOENCODING=utf-8; \
               export DCOS_CONFIG=tests/data/dcos.toml; \
               chmod 600 ${DCOS_CONFIG}; \
               echo dcos_acs_token = \\\"${DCOS_ACS_TOKEN}\\\" >> ${DCOS_CONFIG}; \
               cat ${DCOS_CONFIG}; \
               unset DCOS_URL; \
               unset DCOS_ACS_TOKEN; \
               make test-binary'''
        }
    }
})


builders['mac-tests'] = testBuilder('mac', 'mac')({
    stage ("Run dcos-cli tests") {
        sh '''
           rm -rf ~/.dcos; \
           cp /etc/hosts hosts.local; \
           grep -q "^.* dcos.snakeoil.mesosphere.com$" hosts.local && \
           sed -iold "s/^.* dcos.snakeoil.mesosphere.com$/${DCOS_URL} dcos.snakeoil.mesosphere.com/" hosts.local || \
           echo ${DCOS_URL} dcos.snakeoil.mesosphere.com >> hosts.local; \
           sudo cp ./hosts.local /etc/hosts'''

        dir('dcos-cli/cli') {
            sh '''
               export PYTHONIOENCODING=utf-8; \
               export DCOS_CONFIG=tests/data/dcos.toml; \
               chmod 600 ${DCOS_CONFIG}; \
               echo dcos_acs_token = \\\"${DCOS_ACS_TOKEN}\\\" >> ${DCOS_CONFIG}; \
               cat ${DCOS_CONFIG}; \
               unset DCOS_URL; \
               unset DCOS_ACS_TOKEN; \
               make test-binary'''
        }
    }
})


builders['windows-tests'] = testBuilder('windows', 'windows', 'C:\\windows\\workspace')({
    stage ("Run dcos-cli tests") {
        bat '''
            bash -c "rm -rf ~/.dcos"'''
        bat '''
            echo %DCOS_URL% dcos.snakeoil.mesosphere.com >> C:\\windows\\system32\\drivers\\etc\\hosts &
            echo dcos_acs_token = \"%DCOS_ACS_TOKEN%\" >> dcos-cli\\cli\\tests\\data\\dcos.toml'''

        dir('dcos-cli/cli') {
            bat '''
                bash -c " \
                export PYTHONIOENCODING=utf-8; \
                export DCOS_CONFIG=tests/data/dcos.toml; \
                cat ${DCOS_CONFIG}; \
                unset DCOS_URL; \
                unset DCOS_ACS_TOKEN; \
                make test-binary"'''
        }
    }
})


// This node bootstraps everything including creating all
// the test clusters, starting the builders, and finally
// destroying all the clusters once they are done.
node('py35') {
    stage ('Cleanup workspace') {
        deleteDir()
    }

    stage ('Update node') {
        sh 'pip install requests'
    }

    stage ('Download dcos-launch') {
        sh 'wget https://downloads.dcos.io/dcos-test-utils/bin/linux/dcos-launch'
        sh 'chmod a+x dcos-launch'
    }

    stage ('Pull dcos-cli repository') {
        dir('dcos-cli') {
            checkout scm
        }
    }

    stage ('Stash dcos-cli repository') {
        stash(['includes': 'dcos-cli/**', name: 'dcos-cli'])
    }

    withCredentials(
        [[$class: 'AmazonWebServicesCredentialsBinding',
         credentialsId: '7155bd15-767d-4ae3-a375-e0d74c90a2c4',
         accessKeyVariable: 'AWS_ACCESS_KEY_ID',
         secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'],
        [$class: 'StringBinding',
         credentialsId: 'fd1fe0ae-113d-4096-87b2-15aa9606bb4e',
         variable: 'CF_TEMPLATE_URL'],
        [$class: 'UsernamePasswordMultiBinding',
         credentialsId: '323df884-742b-4099-b8b7-d764e5eb9674',
         usernameVariable: 'DCOS_ADMIN_USERNAME',
         passwordVariable: 'DCOS_ADMIN_PASSWORD']]) {

        parallel builders
    }
}
