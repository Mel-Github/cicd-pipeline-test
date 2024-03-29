// The MIT License
// SPDX short identifier: MIT
// Further resources on the MIT License
// Copyright 2018 Amit Thakur - amitthk - <e.amitthakur@gmail.com>
// Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
// The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.



/*         withCredentials([file(credentialsId: "${JENKINS_GCLOUD_CRED_ID}", variable: 'JENKINSGCLOUDCREDENTIAL')])
            {
                sh """
                gcloud auth activate-service-account --key-file=${JENKINSGCLOUDCREDENTIAL}
                gcloud config set compute/zone asia-southeast1-a
                gcloud config set compute/region asia-southeast1
                gcloud config set project ${GCLOUD_PROJECT_ID}
                gcloud container clusters get-credentials ${GCLOUD_K8S_CLUSTER_NAME}
                
                chmod +x $BASE_DIR/k8s/process_files.sh

                cd $BASE_DIR/k8s/
                ./process_files.sh "$GCLOUD_PROJECT_ID" "${IMAGE_NAME}" "${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}" "./${IMAGE_NAME}/" ${TIMESTAMP}

                cd $BASE_DIR/k8s/${IMAGE_NAME}/.
                kubectl apply -f $BASE_DIR/k8s/${IMAGE_NAME}/
                kubectl rollout status --v=5 --watch=true -f $BASE_DIR/k8s/$IMAGE_NAME/$IMAGE_NAME-deployment.yml
                
                gcloud auth revoke --all
                """

                createDynatraceDeploymentEvent(envId: 'Dynatrace Tenant', tagMatchRules: [[tags: [[context: 'CONTEXTLESS', key: 'app', value: '${env.APP_NAME}'], [context: 'CONTEXTLESS', key: 'environment', value: 'development']]]]) {
    // some block
    // test
}

            } */


def getTimeStamp(){
    return sh (script: "date +'%Y%m%d%H%M%S%N' | sed 's/[0-9][0-9][0-9][0-9][0-9][0-9]\$//g'", returnStdout: true);
}
def getEnvVar(String paramName){
    return sh (script: "grep '${paramName}' env_vars/project.properties|cut -d'=' -f2", returnStdout: true).trim();
}
def getTargetEnv(String branchName){
    def deploy_env = 'none';
    switch(branchName) {
        case 'master':
            deploy_env='uat'
        break
        case 'develop':
            deploy_env = 'dev'
        default:
            if(branchName.startsWith('release')){
                deploy_env='sit'
            }
            if(branchName.startsWith('feature')){
                deploy_env='none'
            }
    }
    return deploy_env
}

def getImageTag(String currentBranch)
{
    def image_tag = 'latest'
    if(currentBranch==null){
        image_tag = getEnvVar('IMAGE_TAG')
    }
    if(currentBranch=='master'){
        image_tag= getEnvVar('IMAGE_TAG')
    }
    if(currentBranch.startsWith('release')){
        image_tag = currentBranch.substring(8);
    }
    return image_tag
}
pipeline{

environment {
    GIT_COMMIT_SHORT_HASH = sh (script: "git rev-parse --short HEAD", returnStdout: true)
}

agent any

stages{
    stage('Init'){
        steps{
            //checkout scm;
        script{
        env.BASE_DIR = pwd()
        env.CURRENT_BRANCH = env.BRANCH_NAME
        env.IMAGE_TAG = getImageTag(env.CURRENT_BRANCH)
        env.TIMESTAMP = getTimeStamp()
        env.APP_NAME= getEnvVar('APP_NAME')
        env.IMAGE_NAME = getEnvVar('IMAGE_NAME')
        env.PROJECT_NAME=getEnvVar('PROJECT_NAME')
        env.DOCKER_REGISTRY_URL=getEnvVar('DOCKER_REGISTRY_URL')
        env.RELEASE_TAG = getEnvVar('RELEASE_TAG')
        env.DOCKER_PROJECT_NAMESPACE = getEnvVar('DOCKER_PROJECT_NAMESPACE')
        env.DOCKER_IMAGE_TAG= "${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${APP_NAME}:${RELEASE_TAG}"
        env.JENKINS_DOCKER_CREDENTIALS_ID = getEnvVar('JENKINS_DOCKER_CREDENTIALS_ID')        
        env.JENKINS_GCLOUD_CRED_ID = getEnvVar('JENKINS_GCLOUD_CRED_ID')
        env.GCLOUD_PROJECT_ID = getEnvVar('GCLOUD_PROJECT_ID')
        env.GCLOUD_K8S_CLUSTER_NAME = getEnvVar('GCLOUD_K8S_CLUSTER_NAME')
        env.JENKINS_GCLOUD_CRED_LOCATION = getEnvVar('JENKINS_GCLOUD_CRED_LOCATION')

        }

        }
    }

    stage('Cleanup'){
        steps{
            container('build') {
            sh 'apk update && apk add docker'
            sh 'docker version'
            sh '''
            hostname
            docker ps -a
            pwd
            ls -l
            '''
            }
        }

    }
    stage('Build'){
        steps{
            container('build') {
                sh 'hostname'
                sh 'echo variable ${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}'
                sh '''
                    docker version
                    pwd
                    ls -l
                    docker build -t ${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG} --build-arg APP_NAME=${IMAGE_NAME}  -f app/Dockerfile app/.
               '''
            }
/*            withEnv(["APP_NAME=${APP_NAME}", "PROJECT_NAME=${PROJECT_NAME}"]){
               sh '''
               docker build -t ${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG} --build-arg APP_NAME=${IMAGE_NAME}  -f app/Dockerfile app/.
               '''
           }    */
        }
    }

    stage('Image-Scan'){
        steps {
            container('build') {
                echo "Scanning - ${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}"
                aquaMicroscanner imageName: "${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}", notCompliesCmd: '', onDisallowed: 'ignore', outputFormat: 'html'
            }
        }
    }


    stage('Publish'){
        steps{
            container('build') {
                sh 'echo variable 1 ${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}'
                withCredentials([usernamePassword(credentialsId: 'JENKINS_DOCKER_CREDENTIALS_ID', passwordVariable: 'DOCKER_PW', usernameVariable: 'DOCKER_ID')])

                // We need to keep login section out from the push logic script else the environment DOCKER_PROJECT_NAMESPACE will be overwritten by the withCredentials values.
                {
                    sh label: '', script: 'docker login -u ${DOCKER_ID} -p ${DOCKER_PW}'
                }

                // Core login scripts dealing with Docker registry goes here.

                    echo "Publish"
                    sh 'hostname'
                    sh 'echo variable 2 ${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}'
                    
                    sh 'printenv'


                    // Pushing to Docker 

                    sh label: '', script: 'docker push ${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}'
                    sh 'docker logout' 
                    sh '''
                        docker version
                        docker image ls
                        pwd
                        ls -l
                    '''
            }
       } 
    }



    stage('Pre-Deploy'){
        steps{
        //    container('jenkinsagent') {
                echo "Pre-Deploy"
                sh 'hostname'
                sh 'echo variable ${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}'
                sh  'echo $0'
/*                 sh '''
                    docker version
                    pwd
                    
                    chmod +x $BASE_DIR/k8s/process_files.sh

                    cd $BASE_DIR/k8s/
                    pwd
                    ls -l
                    
                ''' */
                sh 'chmod +x $BASE_DIR/k8s/process_files.sh'
                sh 'cd $BASE_DIR/k8s/'
                sh 'pwd'
                sh 'ls -l'
                sh 'cd k8s'
                sh 'pwd'
                sh 'ls -l'
                dir("$BASE_DIR/k8s/") {
                    sh 'pwd'
                    //sh './process_files.sh 1 2 3 4 5'
                    sh './process_files.sh "$GCLOUD_PROJECT_ID" "${IMAGE_NAME}" "${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}" "./${IMAGE_NAME}/" ${TIMESTAMP}'
                }
                sh 'ls -l'
                //sh 'cat $BASE_DIR/k8s/process_files.sh'
                //sh '/$BASE_DIR/k8s/process_files.sh alpha beta charlie delta echo'
                //sh './process_files.sh "$GCLOUD_PROJECT_ID" "${IMAGE_NAME}" "${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}" "./${IMAGE_NAME}/" ${TIMESTAMP}'
          //  }
        }
    }


    stage('Deploy') {
        steps {
            createDynatraceDeploymentEvent(
                envId: 'Dynatrace Tenant', 
                tagMatchRules: [
                    [
                        meTypes:[[meType: 'SERVICE']],
                        tags: [
                            [context: 'CONTEXTLESS', key: 'app', value: "${APP_NAME}"], 
                            [context: 'CONTEXTLESS', key: 'environment', value: 'development']
                        ]
                    ]
                ]) {
                    container('kubectl') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'jenkins-robot', namespace: 'development', serverUrl: 'https://kubernetes.default') {
                            // some block
                            sh 'kubectl get pods'
                            sh 'echo kubectl - ${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}'
                            //sh 'printenv'
                            dir("$BASE_DIR/k8s/${IMAGE_NAME}") {
                            // cd $BASE_DIR/k8s/${IMAGE_NAME}/.
                            sh 'ls -l'
                            sh 'cat kubejencdp-app-deployment.yml'
                            sh 'kubectl apply -f $BASE_DIR/k8s/${IMAGE_NAME}/'
                            //kubectl rollout status --v=5 --watch=true -f $BASE_DIR/k8s/$IMAGE_NAME/$IMAGE_NAME-deployment.yml
                            }
                        }
                        
                    }   
            }
        }
    } 


    stage('performance test') {
        steps {
            recordDynatraceSession(
                envId: 'Dynatrace Tenant',
                testCase: 'loadtest',
                tagMatchRules: [
                    [
                        meTypes: [[meType: 'SERVICE']],
                        tags:[
                            [context: 'CONTEXTLESS', key: 'app', value: "${APP_NAME}"],
                            [context: 'CONTEXTLESS', key: 'environment', value: 'development']    
                        ]
                    ] 
                ]
            )   {
                    container('jmeter') {
                        sh 'apk update && apk add docker'
                        dir("$WORKSPACE/jmeter/") {
                            sh 'printenv'
                            sh 'cd ..'
                            sh 'pwd'
                            sh 'ls -l'
                            //sh 'ls -l /home/jenkins/agent/workspace'
                            sh 'jmeter -version'
                            //sh 'ls -l /home/jenkins/agent/workspace'
                            //sh 'jmeter  -Jdt_ltn=${dt_ltn} -Jhostname=${hostname} -Jthread=${thread} -Jjmeter.save.saveservice.output_format=xml -n -t "${APP_NAME}".jmx l "${APP_NAME}".jtl'
                            build job: 'Jmeter', 
                                parameters: [
                                    string(name: 'thread', value: '15'), 
                                    string(name: 'hostname', value: '35.187.247.198'), 
                                    string(name: 'loop', value: '2'), 
                                    string(name: 'rampup', value: '5'), 
                                    string(name: 'dt_ltn', value: 'LTN=Homepage;VU=5')
                                ]
                        }
                    }
                }

                dir("$WORKSPACE/jmeter/") {
                    sh 'pwd'    
                    sh 'ls -l'
                    sh 'cat specfile.json'
                    perfSigDynatraceReports envId: 'Dynatrace Tenant', nonFunctionalFailure: 1, specFile: 'specfile.json'
                }
        }
    }

    // stage('Report performance test result') {
    //     steps {
    //         container('jmeter') {
    //             sh 'apk update && apk add docker'
    //             dir("$WORKSPACE/jmeter/") {
    //                 perfReport filterRegex: '', sourceDataFiles: 'kubejencdp-app-ctr.jtl'
    //             }
    //         }
    //     }
    // }    

}

post {
    always {
        echo "Build# ${env.BUILD_NUMBER} - Job: ${BUILD_NUMBER} status is: ${currentBuild.currentResult}"
        // emailext(attachLog: true,
        // mimeType: 'text/html',
        // body: '''
        // <h2>Build# ${env.BUILD_NUMBER} - Job: ${env.JOB_NUMBER} status is: ${currentBuild.currentResult}</h2>
        // <p>Check console output at &QUOT;<a href='${env.BUILD_URL'>${env.JOB_NAME} - [${env.BUILD_NUMBER}]</a>&QUOT;</p>
        // ''',
        // recipientProviders: [[$class: "FirstFailingBuildSusspectRecipientProvider"]],
        // subject: "Build# ${env.BUILD_NUMBER} - Job: ${env.JOB_NUMBER} status is: ${currentBuild.currentResult}",
        // to: "e.amitthakur@gmail.com")
    }
}
}
