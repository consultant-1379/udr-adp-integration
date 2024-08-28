// These are the only parameters that should
// be edited when reusing the pipeline script

SERVICE_NAME="renegadedb-ldapfe"
SERVICE_VERSION="0.1.0"
GIT_PROJECT="5gcicd/udr-adp-integration"
GIT_PROJECT_INTEGRATION_TEST="renegadedb-integration-test/renegadedb-integration-test"
REPO_ROOT_DIR="udr-adp-integration"
REPO_INTEGRATION_TEST_DIR="renegadedb-integration-test"
ARM_PATH="sekidocker.rnd.ki.sw.ericsson.se"
ARM_PROJECT_PATH="${ARM_PATH}/proj-renegade"
BINARY_PATH="/usr/src/app/target"
TOOLS_REPO="https://arm.rnd.ki.sw.ericsson.se/artifactory/proj-renegade-cpp-generic-local"
CLEANER="${TOOLS_REPO}/images_common/arm_cleaner.sh"
HELM_REPO="${TOOLS_REPO}/helm"
CHART_INDEX="${HELM_REPO}/index.yaml"
HELM_RELEASE_NAME="renegade-jenkins-renegade-jenkins"
FAIL_TIMEOUT="3600"
PIPELINE="jenkins-pipeline-ldap"


podTemplate(serviceAccount: "${HELM_RELEASE_NAME}", label: "${PIPELINE}", containers: [
    containerTemplate(
        name: 'jnlp', image: 'armdocker.rnd.ericsson.se/proj-5g-cicd-release/jenkins/jnlp-slave-docker:3.10-1', args: '${computer.jnlpmac} ${computer.name}', 
        resourceRequestCpu: '200m', resourceLimitCpu: '400m',
        resourceRequestMemory: '256Mi', resourceLimitMemory: '1024Mi'
    ),
    containerTemplate(name: 'helm-ruby', image: 'armdocker.rnd.ericsson.se/proj-5g-cicd-release/helm/helm-ruby:1.0.0',
        envVars: [
            containerEnvVar(key:'http_proxy', value:'http://www-proxy.ericsson.se:8080'),
            containerEnvVar(key:'https_proxy', value:'https://www-proxy.ericsson.se:8080'),
            containerEnvVar(key:'no_proxy', value:'10.96.0.1,150.132.10.215,arm.rnd.ki.sw.ericsson.se')
        ],
        resourceRequestCpu: '200m', resourceLimitCpu: '400m',
        resourceRequestMemory: '256Mi', resourceLimitMemory: '1024Mi',
        command: 'cat', ttyEnabled: true
    ),
    containerTemplate(name: 'renegade-cli',
        image: 'sekidocker.rnd.ki.sw.ericsson.se/proj-renegade/renegade-cli:latest',
        envVars: [
            containerEnvVar(key:'http_proxy', value:'http://www-proxy.ericsson.se:8080'),
            containerEnvVar(key:'https_proxy', value:'https://www-proxy.ericsson.se:8080'),
            containerEnvVar(key:'no_proxy', value:'10.96.0.1,150.132.10.215,arm.rnd.ki.sw.ericsson.se'),
            containerEnvVar(key:'RENEGADE_TEST_USER_ID', value:'jenkins')
        ],
        command: 'cat',
        alwaysPullImage: true,
        resourceRequestCpu: '200m', resourceLimitCpu: '400m',
        resourceRequestMemory: '256Mi', resourceLimitMemory: '1024Mi',
        ttyEnabled: true
    )]
){

// This is a jenkinsfile-defined pipeline
node ("${PIPELINE}") {
    def pwd = pwd()
    def workdir = "${pwd}"
    def image_file_path="${pwd}"
    def chart_dir = "${pwd}/${REPO_ROOT_DIR}/charts/${SERVICE_NAME}"
    def integration_charts_dir = "${pwd}/${REPO_ROOT_DIR}/charts/ldap-kpi"
    def integration_conf_dir = "${pwd}/${REPO_ROOT_DIR}/conf"
    def namespace_id = getRandomNumber(5)

    withCredentials([usernamePassword(credentialsId: 'userpwd-cudb', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        stage('clone repos') {
            container ('jnlp') {
                sh "echo current path:`pwd`"
                sh "git clone https://${USERNAME}:${PASSWORD}@gerrit.sero.gic.ericsson.se/a/${GIT_PROJECT}"
                sh "cd ${REPO_ROOT_DIR} && git pull"
                sh "git clone https://${USERNAME}:${PASSWORD}@gerrit.sero.gic.ericsson.se/a/${GIT_PROJECT_INTEGRATION_TEST}"
                sh "cd ${REPO_INTEGRATION_TEST_DIR} && git pull"
            }
        }
        stage('update dependencies') {
            container ('helm-ruby') {
                dir("${integration_charts_dir}") {
                    sh '''
                    cp requirements.yaml requirements.old.yaml
                    ruby -ryaml -e "req=YAML.load(ARGF); \
                        req['dependencies'].each do |dep| \
                        if dep['name'] == '$CHART_NAME'; then \
                        dep['repository'] = '$CHART_REPO'; \
                        dep['version'] = '$CHART_VERSION'; \
                        end; \
                        end; \
                        puts YAML.dump(req);" < requirements.old.yaml > requirements.yaml
                    '''
                    helmConfig()
                    helmDepUpdate(chart_dir: integration_charts_dir)
                }
            }
        }
        stage ('deploy integration test environment') {
            container ('renegade-cli') {
                dir("${integration_charts_dir}"){
                    sh "/usr/src/app/renegade chart-build " +
                        "--path ${integration_charts_dir} " +
                        "--settings minikube " +
                        "--values ${workdir}/${REPO_ROOT_DIR}/values/startup_stubs.yaml " +
                        "--destination . "
                }
                sh "/usr/src/app/renegade namespace-create --name jenkins-${namespace_id}"
                dir("${integration_charts_dir}"){
                    sh "cp ${workdir}/${REPO_ROOT_DIR}/values/values_minikube.yaml " +
                        "${integration_charts_dir}"
                    sh "helm install --values values_minikube.yaml --namespace jenkins-${namespace_id} " + 
                        "--name ldap-kpi-${namespace_id} ldap-kpi-0.0.1.tgz"
                }
            }
        }
        stage ('run integration tests') {
            container ('renegade-cli') {
                dir("${integration_conf_dir}") {
                    sh "/usr/src/app/renegade configuration-create --name junit-btraffic-suite " + 
                        "--namespace jenkins-${namespace_id} --file ${integration_conf_dir}/junit-btraffic-suite.yaml"
                }
                sh "/usr/src/app/renegade task-start  --tag latest --name test-${namespace_id} " + 
                    "--namespace jenkins-${namespace_id} --image sekidocker.rnd.ki.sw.ericsson.se/proj-renegade/renegadedb-integration-tests " + 
                    "--tag 0.1 --configuration junit-btraffic-suite"
                sh "/usr/src/app/renegade task-logs --name test-${namespace_id} --namespace jenkins-${namespace_id} --follow "
                sh "/usr/src/app/renegade task-wait --name test-${namespace_id} --namespace jenkins-${namespace_id} "
            }
        }
        stage ('delete integration test environment') {
            container ('renegade-cli') {
                sh "helm delete --purge ldap-kpi-${namespace_id} "
                sh "/usr/src/app/renegade namespace-delete --name jenkins-${namespace_id}"
            }
        }

        stage ('update baseline') {
            container ('jnlp') {
                dir("${integration_charts_dir}") {
                    sh "git config --global user.email 'renegade.ci@ericsson.com'"
                    sh "git config --global user.name 'ereneci'"
                    sh "git add ${integration_charts_dir}/requirements.yaml"
                    sh "git commit -m 'Automatic new version due to $CHART_NAME $CHART_VERSION'"
                    sh "git push --force origin HEAD:master"
                }
            }
        }
    } // end (withCredentials)
}
}

def helmConfig() {
    // Setup helm connectivity to Kubernetes API and Tiller
    println "initiliazing helm client"
    sh "helm init -c"
}

def helmDepUpdate(Map args) {
    // Download and install helm dependencies
    println "initiliazing helm client"
    sh "helm init -c"
    
    sh "ls -l ${args.chart_dir}/requirements.yaml && cat ${args.chart_dir}/requirements.yaml"
    
    sh "ruby -ryaml -e 'x=YAML.load(ARGF); puts x;' < ${args.chart_dir}/requirements.yaml"
    
    
    sh "printf 'helm repo add %s %s\n' \$(ruby -ryaml -e 'x=YAML.load(ARGF); \
        x[\"dependencies\"].each do |dep| \
        puts dep[\"name\"] + \" \" + dep[\"repository\"]; \
        end' < ${args.chart_dir}/requirements.yaml) | /bin/sh"
    
    println "updating helm dependencies"
    sh "helm dependencies update ${args.chart_dir}"
}

def updateRequirements(Map args) {
    // Update helm dependencies (NOT USED for now)
    println "Updating versions in requirements.yaml"
    //sh "ruby ruby_file $args.chart_name $args.chart_version $args.chart_repo"
    sh "ruby --version"
    println "requirements.yaml udpated"
}

def getRandomNumber(int num) {
    // Generate a random integer up 10^num
    Random random = new Random()
    return random.nextInt(10 ** num)
}

