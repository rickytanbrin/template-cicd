def git_repo = 'https://bitbucket.cimbniaga.co.id/scm/cl_dev/poin-xtra-service.git'
def public_route_prefix = 'clicks'
def public_route_path = 'api/poinxtra'
def max_replica_count = 1

def cpu_limit = '300m'
def memory_limit = '768Mi'

def git_branch = 'development'

def nexus_base_url = 'https://artifactory.cimbniaga.co.id'
def nexus_deps_repo = "$nexus_base_url/artifactory/maven_cimb/"
def nexus_deploy_repo = "$nexus_base_url/artifactory/clicks-backend/"

def ocp_project = 'clicks-dev'
def env = 'dev'
def secret_file = ''

def appName
def appFullVersion
def gitCommitId

node ('master'){
        stage('Checkout') {
            git url: "${git_repo}", branch: "${git_branch}", credentialsId: "buildUser"
        }

        stage ('Prepare'){
            withCredentials([[$class: 'UsernamePasswordMultiBinding',
                credentialsId: 'buildUser',
                usernameVariable: 'nexus_username', passwordVariable: 'nexus_password']]) {
                sh """
                    echo 'Downloading ci-cd templates...'
                    curl --fail -u ${nexus_username}:${nexus_password} -o cicd-template-${env}.tar.gz ${nexus_base_url}/artifactory/clicks-general/cicd-template-${env}.tar.gz
                    rm -rf cicd-template || true
                    mkdir cicd-template && tar -xzvf ./cicd-template-${env}.tar.gz -C "\$(pwd)/cicd-template"
                    """
                    prepareSettingsXml(nexus_deps_repo, nexus_username, nexus_password)
                    addDistributionToPom(nexus_deploy_repo)
            }

            appName = getFromPom('name')
            if(appName == null || appName.trim() == ""){
            appName = getFromPom('artifactId')
            }

            withMaven (maven: 'mvn-master') {
            sh "export JAVA_HOME=/u01/Java/jdk-11.0.1 && mvn -s ./cicd-template/maven/settings.xml build-helper:parse-version versions:set -DnewVersion='\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.0-\${parsedVersion.qualifier}' versions:commit"
            }

            appFullVersion = getFromPom('version')
            gitCommitId = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
            echo "appName: '${appName}', appFullVersion:'${appFullVersion}', gitCommitId:'${gitCommitId}'"
        }

        stage('BuildDataModel'){
            withMaven(maven: 'mvn-master') {
            sh """
                export JAVA_HOME=/u01/Java/jdk-11.0.1 && mvn clean package -D skipTests -P data-model -s ./cicd-template/maven/settings.xml
            """
           }
        }

         stage('ArchiveDataModel'){
             withMaven(maven: 'mvn-master') {
             sh """
               export JAVA_HOME=/u01/Java/jdk-11.0.1 && mvn deploy -D skipTests -P data-model -s ./cicd-template/maven/settings.xml
             """
             }
         }

        stage('Build') {
            withMaven(maven: 'mvn-master') {
            sh """
              export JAVA_HOME=/u01/Java/jdk-11.0.1 && mvn clean package -D skipTests -s ./cicd-template/maven/settings.xml
            """
            }
        }

        stage ('Test'){
            withMaven(maven: 'mvn-master') {
            sh """
                export JAVA_HOME=/u01/Java/jdk-11.0.1 && mvn test -s ./cicd-template/maven/settings.xml
            """
            }
        }
        stage('IntegrationTests') {
            withMaven(maven: 'mvn-master') {
            sh """
                export JAVA_HOME=/u01/Java/jdk-11.0.1 && mvn failsafe:integration-test  -s ./cicd-template/maven/settings.xml
            """
            }
        }

        // stage ('Archive'){
        //     withMaven(maven: 'mvn-master') {
        //     sh """
        //         export JAVA_HOME=/u01/Java/jdk-11.0.1 && mvn deploy  -DskipTests -s ./cicd-template/maven/settings.xml
        //     """
        //     }
        // }

        stage ('OCP Build Preparation'){
            jarFile = sh(returnStdout: true, script: 'find ./target -maxdepth 1 -regextype posix-extended -regex ".+\\.(jar|war)\$" | head -n 1').trim()
            if(jarFile == null || jarFile == ""){
                error 'Can not find the generated jar/war file from "./target" directory'
            }
            jarFile = jarFile.substring('./target/'.length());
            sh """
                set -x
                set -e

                mkdir -p ./target/publish/.s2i
                cp ./target/$jarFile ./target/publish/
                echo 'JAVA_APP_JAR=/deployments/${jarFile}' > ./target/publish/.s2i/environment
            """
        }

        stage ('OCP Build'){
            appMajorVersion = appFullVersion.substring(0, appFullVersion.indexOf('.'))
            sh """
                export JAVA_HOME=/u01/Java/jdk-11.0.1
                echo 'application name=${appName}'

                set -x
                set -e

                oc --config=/home/jenkins/oc-config/login-dev.cnf project ${ocp_project}
                oc --config=/home/jenkins/oc-config/login-dev.cnf process -f ./cicd-template/openshift/build-config-template.yaml -n ${ocp_project} \
                -p S2I_BUILD_IMAGE='openjdk-11-rhel7' \
                -p APP_NAME='${appName}' -p APP_FULL_VERSION='${appFullVersion}' -p APP_MAJOR_VERSION='${appMajorVersion}' \
                -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER}  \
                | oc --config=/home/jenkins/oc-config/login-dev.cnf apply -n ${ocp_project} -f -

                export JAVA_HOME=/u01/Java/jdk-11.0.1
                oc --config=/home/jenkins/oc-config/login-dev.cnf start-build ${appName}-v${appMajorVersion} -n ${ocp_project} --from-dir='./target/publish' --follow
            """
        }

        stage ('OCP ConfigMap'){
            sh """
                set -x
                set -e
                if [ -f './src/config/application.properties' ]; then
                    export APP_CONFIG_DATA=\$(cat src/config/application.properties)
                else
                    echo 'Please prepare your application configuration at "src/config/application.properties"'
                    exit 1
                fi

                oc --config=/home/jenkins/oc-config/login-dev.cnf project ${ocp_project}
                oc --config=/home/jenkins/oc-config/login-dev.cnf process -f ./cicd-template/openshift/configmap-template.yaml -n ${ocp_project} \
                -p APP_NAME='${appName}' -p APP_FULL_VERSION='${appFullVersion}' -p APP_MAJOR_VERSION='${appMajorVersion}' \
                -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p CONFIG_DATA="\$APP_CONFIG_DATA" \
                | oc --config=/home/jenkins/oc-config/login-dev.cnf apply -n ${ocp_project} -f -
            """
        }

        stage ('OCP Deployment'){
            sh """
                set -x
                set -e

                oc --config=/home/jenkins/oc-config/login-dev.cnf project ${ocp_project}
                oc --config=/home/jenkins/oc-config/login-dev.cnf process -f ./cicd-template/openshift/deployment-config-template.yaml -n ${ocp_project} \
                -p APP_NAME=${appName} -p APP_FULL_VERSION=${appFullVersion} -p APP_MAJOR_VERSION=${appMajorVersion}  \
                -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} \
                -p CPU_LIMIT=${cpu_limit} -p MEMORY_LIMIT=${memory_limit} \
                | oc --config=/home/jenkins/oc-config/login-dev.cnf apply -n ${ocp_project} --force=true -f -
            """
            if (secret_file != null && secret_file != ''){
                                    sh """
                                        set -x
                                        set -e
                                        sleep 3
                                        cat ./src/config/${env}.env | oc --config=/home/jenkins/oc-config/login-dev.cnf set env --from=secret/${secret_file} dc/${appName}-v${appMajorVersion} -
                                    """
                                    } else{
                                    sh """
                                        set -x
                                        set -e
                                        sleep 3
                                        cat ./src/config/${env}.env | oc --config=/home/jenkins/oc-config/login-dev.cnf set env dc/${appName}-v${appMajorVersion} -
                                    """
                                    }

            if (public_route_prefix != null && public_route_prefix != ''){
                sh """
                    set -x
                    set -e

                    oc --config=/home/jenkins/oc-config/login-dev.cnf project ${ocp_project}
                    oc --config=/home/jenkins/oc-config/login-dev.cnf process -f ./cicd-template/openshift/route-template.yaml -n ${ocp_project} \
                    -p APP_NAME=${appName} -p APP_FULL_VERSION=${appFullVersion} -p APP_MAJOR_VERSION=${appMajorVersion}  \
                    -p GIT_COMMIT_ID=${gitCommitId} -p PUBLIC_ROUTE_PREFIX=${public_route_prefix} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} \
                    -p APP_ROUTE_PATH="/${public_route_path}" \
                    | oc --config=/home/jenkins/oc-config/login-dev.cnf apply -n ${ocp_project} --force=true -f -
                """
            }

            /**
            * Add an autoscale (fill the variable max_replica_count)
            */
            if (max_replica_count != null && max_replica_count > 1){
                sh """
                    set -x
                    set -e

                    oc --config=/home/jenkins/oc-config/login-dev.cnf project ${ocp_project}
                    oc --config=/home/jenkins/oc-config/login-dev.cnf autoscale dc/${appName}-v${appMajorVersion} --min 1 --max ${max_replica_count} --cpu-percent=80 || true
                    oc --config=/home/jenkins/oc-config/login-dev.cnf rollout status dc/${appName}-v${appMajorVersion}
                    """
            }
    }
}

def getFromPom(key) {
    withMaven (maven: 'mvn-master') {
        return sh(returnStdout: true, script: "/u01/apache-maven-3.6.1/bin/mvn -s ./cicd-template/maven/settings.xml -q -Dexec.executable=echo -Dexec.args='\${project.${key}}' --non-recursive exec:exec").trim()
    }
}

/**
* Add a 'distribution-management' section into pom.xml
* So that the jar/war artifact could be packed and uploaded to repository
*/
def addDistributionToPom(nexus_deploy_repo) {
    pom = 'pom.xml'
    distMngtSection = readFile('./cicd-template/maven/pom-distribution-management.xml')
    distMngtSection = distMngtSection.replaceAll('\\$nexus_deploy_repo', nexus_deploy_repo)

    content = readFile(pom)
    newContent = content.substring(0, content.lastIndexOf('</project>')) + distMngtSection + '</project>'
    writeFile file: pom, text: newContent
}

/**
* Create a 'settings.xml' file for maven to use
* So that maven can use internal nexus repo as a mirror server when installing dependencies
*/
def prepareSettingsXml(nexus_deps_repo, nexus_username, nexus_password) {
    settingsXML = readFile('./cicd-template/maven/settings.xml')
    settingsXML = settingsXML.replaceAll('\\$nexus_deps_repo', nexus_deps_repo)
    settingsXML = settingsXML.replaceAll('\\$nexus_username', nexus_username)
    settingsXML = settingsXML.replaceAll('\\$nexus_password', nexus_password)

    writeFile file: './cicd-template/maven/settings.xml', text: settingsXML
}
