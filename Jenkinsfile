def namespace = 'jiaoproject'
def deploymentName = 'imagetest3'

def buildImage = 'index-staging.alauda.cn/testorg001/zjpublic:latest-lognew'
def imageRepo = 'index-staging.alauda.cn/testorg001/zjpublic'
def registryCredentials = "staging-secret"
def imageTag = 'latest-lognew'
def imageUrl = ""
def imageUrlPredefined = "$imageRepo:$imageTag"
def subCommand = ""

def gitAddress = 'https://github.com/jiaozhang1/logtest.git'
def contextDir = '.'
def gitRef = 'master'
def gitCredentials = ""

pipeline {
    agent {
        label 'golang'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    if (gitCredentials){
                        gitCredentials = "$namespace-$gitCredentials"
                    }
                    def scmVars = git(
                        url: gitAddress,
                        branch: gitRef,
                        credentialsId: gitCredentials,
                    )
                    def commitID = "${scmVars.GIT_COMMIT}"
                    if (commitID){
                        imageUrl = "$imageRepo:$commitID"
                    }else{
                        imageUrl = imageRepo
                    }
                    echo "commitID: $commitID; imageUrl: $imageUrl"
                }
            }
        }
        stage('CI') {
            steps {
                script {
                    if (registryCredentials){
                        registryCredentials = "$namespace-$registryCredentials"
                        withCredentials([usernamePassword(credentialsId: registryCredentials, passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                            sh """
                            docker login $imageRepo -u $USERNAME -p $PASSWORD
                            """
                        }
                    }
                    
                    if (contextDir){
                        subCommand = "$subCommand --context-dir=$contextDir"
                    }

                    def envParams = [
                        'IMPORT_URL': '',
                        'INSTALL_URL': '',
                    ]
                    envParams.each{
                        if (it.value){
                            subCommand = "$subCommand -e $it.key=$it.value"
                        }
                    }
                    echo "subcommand: $subCommand"
                }
                // s2i build ./ ${subCommand} ${buildImage} ${deploymentName}
                sh """
                docker build -t ${imageUrlPredefined} ${contextDir}
                docker tag ${imageUrlPredefined} ${imageUrl}
                docker push ${imageUrl}
                docker push ${imageUrlPredefined}
                """
            }
        }
        stage('Deploying') {
            steps {
                script {
                    alaudaDevops.withProject(namespace) {
                        echo "Updating the deployment $deploymentName with image $imageUrl in project ${alaudaDevops.project()}"

                        def p = alaudaDevops.selector('deploy', deploymentName).object()
                        p.metadata.labels['BUILD_ID']=env.BUILD_ID
                        p.spec.template.spec.containers[0]['image'] = imageUrl
                        alaudaDevops.apply(p)

                        echo "Completed the update."
                    }                    
                }
            }
        }
    }
}
