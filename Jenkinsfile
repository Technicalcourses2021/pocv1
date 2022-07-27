#!/usr/bin/env groovy
void checkoutRevisionBranch(String branchName) {
    checkout([$class: 'GitSCM', branches: [[name: branchName]],
              doGenerateSubmoduleConfigurations: false,
              extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'rev']],
              submoduleCfg: [],
              userRemoteConfigs: [[credentialsId: 'pocuser', url: 'https://mygithub.gsk.com/sxp14858/pocproject.git']]])
}


void checkoutRevisionFile() {
    echo sh(returnStdout: true, script: 'env')
    checkoutRevisionBranch('Master')
    println 'Checking is remote branch ' + env.branch + ' exist'

    def branchExists=sh (script: 'cd rev && git show-ref ${branch} || true',returnStdout: true).trim()

    print 'Results of checking ' + branchExists

    if (branchExists != ''){
        env.BranchExist=true
        println 'Branch exist , checking out the revision number'
        checkout([$class: 'GitSCM', branches: [[name: env.branch]],
                  doGenerateSubmoduleConfigurations: false,
                  extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'rev'], [$class: 'CleanBeforeCheckout']],
                  submoduleCfg: [],
                  userRemoteConfigs: [[credentialsId: 'pocuser', url: 'https://mygithub.gsk.com/sxp14858/pocproject.git']]])
    }else{
        env.BranchExist=false
        println 'Branch do not exist creating branch'
        sh 'cd rev && git branch ${branch}'
    }
}

void runAntTask(String action, String userCredentials, String sfEnv) {
    errFileAddon = 'ErrorLog.txt'
    subfolder = 'C:/Program Files (x86)/Jenkins/workspace/validation/'
    sfLoginUrl = 'https://test.salesforce.com'
    def validationFilePath = ''
    def errorFile = userCredentials +  errFileAddon
    try{
        if (sfEnv == 'pocorg') {
            sfLoginUrl = 'https://login.salesforce.com'
        }
        println 'sfLoginUrl ' + sfLoginUrl + ' source ' + env.source + 'errorFile ' + errorFile
        codeAction = 'validateCode'
        if (action == 'deploy') {
            codeAction = 'deployCode'
        }
        println 'pull request ' + env.CHANGE_ID
        def prNumber = sh (script: 'git log --oneline -1 | grep -o -m 1 "#[0-9]*" || true',returnStdout: true).trim().replace('#','')
        validationFilePath = subfolder + userCredentials + prNumber + '.txt'
        println 'validationFilePath ' + validationFilePath
        if (action == 'deploy' && fileExists(validationFilePath) && prNumber != null && prNumber != '') {
            def valId = sh (script: 'cat "' + validationFilePath + '"', returnStdout: true).trim()
            println 'validationId ' + valId
            withCredentials([usernamePassword(credentialsId: userCredentials, passwordVariable: 'SFDC_Password', usernameVariable: 'SFDC_Username')]) {
                sh 'ant quickDeploy -f build/build.xml -Dsf.password=$SFDC_Password -Dsf.username=$SFDC_Username -Dsf.serverurl=' + sfLoginUrl + ' -Dsf.recentValidationId=' + valId + ' &> ' + errorFile
            }
            sh 'rm "'+ validationFilePath + '"'
        } else {
            withCredentials([usernamePassword(credentialsId: userCredentials, passwordVariable: 'SFDC_Password', usernameVariable: 'SFDC_Username')]) {
                sh 'ant ' + codeAction + ' -f build/build.xml -Dsf.password=$SFDC_Password -Dsf.username=$SFDC_Username -Dsf.src=${source} -Dsf.serverurl=' + sfLoginUrl + ' -Dsf.testLevel=${TestLevel} -Dsf.allowMissingFiles=${allowMissingFiles} -Dsf.autoUpdatePackage=${autoUpdatePackage} -Dsf.ignoreWarnings=true &> ' + errorFile
            }
        }
        //create file with validation id for specific credential and PR Id
        if (codeAction == 'validateCode' && devEnv == false) {
            sh 'cat '+ errorFile
            sh 'egrep -o -m 1 [a-zA-Z0-9]{18} ' + errorFile + ' |& tee "' + subfolder + userCredentials + env.CHANGE_ID + '.txt"'
            sh 'cat "' + subfolder + userCredentials + env.CHANGE_ID + '.txt"'
        }
    } catch (err) {
        echo err.getMessage()
        def invalidQuickDeploy = ''
        if (fileExists(errorFile)){
            sh 'cat ' + errorFile
            def errorMessage = sh (script: 'cat ' + errorFile,returnStdout: true).trim()
            invalidQuickDeploy = sh (script: 'if grep -q INVALID_ID '+ errorFile +'; then echo true; fi', returnStdout: true).trim()
            // handle full deploy when invalid Quick Deploy Id
            if (invalidQuickDeploy == 'true'){
                sh 'rm "'+ validationFilePath + '"'
                runAntTask(action, userCredentials, sfEnv)
            }
        }
        if (invalidQuickDeploy != 'true') {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'deployment failed') {
                sh 'exit 1'
            }
            currentBuild.result = 'FAILED'
        }
    }
}

void runPostDepScripts(String userCredentials, String sfEnv) {
    try {
        sfLoginUrl = 'https://test.salesforce.com'
        if (sfEnv == 'prod') {
            sfLoginUrl = 'https://login.salesforce.com'
        }
        if (fileExists('release.txt')) {
            releaseName = sh (script: 'cat "release.txt"',returnStdout: true).trim()
        }
        def scriptsDirectory = './Manual Steps/' + releaseName + '/scripts'
        if (fileExists(scriptsDirectory)){
            withCredentials([usernamePassword(credentialsId: userCredentials, passwordVariable: 'SFDC_Password', usernameVariable: 'SFDC_Username')]) {
                sh 'sfdx sfpowerkit:auth:login -u $SFDC_Username -p $SFDC_Password -r ' + sfLoginUrl
                sh 'cd "' + 'Manual Steps/' + releaseName + '/scripts' + '";ls;' + 'for file in ./*.apex; do sfdx force:apex:execute -u $SFDC_Username -f  "$file"; done'
                sh 'echo y | sfdx force:auth:logout --targetusername $SFDC_Username'
            }
        }
    } catch (err) {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'script execution failed') {
            sh 'exit 1'
        }
    }
}

def getDifferChanges(String revision){
    sh 'chmod 755 build/gitDiffer.jar'
    withAnt(installation: 'Ant', jdk: 'Java 9') {
        sh 'java -jar ./build/gitDiffer.jar src ${revision}'
    }
}

pipeline {
    agent any
    parameters {
        choice(name: 'TestLevel', choices: ['NoTestRun', 'RunLocalTests', 'RunAllTestsInOrg'], description: 'Specify test level for application')
        choice(name: 'source', choices: ['toDeploy','src'], description: 'Specify folder which will be deployed. If toDeploy folder specified it will deploy diff only, else if src specified it will deploy full repository')
        choice(name: 'autoUpdatePackage', choices: ['true','false'], description: 'Set the value for autoUpdatePackage in build.xml')
        choice(name: 'buildType', choices: ['standard', 'profilesRetrieval'], description: 'Set value profilesRetrieval to retrieve profiles from provided credential and commit to current branch')
        choice(name: 'retrievalCredential', choices: ['develop'], description: 'If you choose profilesRetrieval pick env, from which profiles will be retrieved')
        string(name: 'commitMessage', defaultValue: 'Commiting profiles from nadevint', description: 'If you choose profilesRetrieval, provide commit message')
    }
    options {
        disableConcurrentBuilds()
    }
    stages {
// workaround for empty parameters after 1st jenkins execution, in case of empty parameters
// this stage will populate env.parameters with default values
        stage('Checking parameters values') {
            steps {
                script {
                    def TestLevel = sh (script: 'echo ${TestLevel}',returnStdout: true).trim()
                    def source = sh (script: 'echo ${source}',returnStdout: true).trim()
                    def autoUpdatePackage = sh (script: 'echo ${autoUpdatePackage}',returnStdout: true).trim()
                    def buildType = sh (script: 'echo ${buildType}',returnStdout: true).trim()
                    def retrievalCredential = sh (script: 'echo ${retrievalCredential}',returnStdout: true).trim()
                    def commitMessage = sh (script: 'echo ${commitMessage}',returnStdout: true).trim()
                    devEnv = true
                    env.allowMissingFiles = true;

                    if (env.GIT_BRANCH == 'master' || env.CHANGE_TARGET == 'master' ||
                            env.GIT_BRANCH == 'develop' || env.CHANGE_TARGET == 'develop' ) {
                        source = 'src'
                        env.source = 'src'
                        TestLevel = 'RunLocalTests'
                        env.TestLevel = 'RunLocalTests'
                        devEnv = false;
                        autoUpdatePackage = false;
                        env.autoUpdatePackage = false;
                        env.allowMissingFiles = false;
                    } else if (env.GIT_BRANCH == 'merge' || env.CHANGE_TARGET == 'merge') {
                        source = 'src'
                        env.source = 'src'
                        TestLevel = 'RunLocalTests'
                        env.TestLevel = 'RunLocalTests'
                        devEnv = false;
                        autoUpdatePackage = false;
                        env.autoUpdatePackage = false;
                        env.allowMissingFiles = false;
                    }
                    if (TestLevel == ''){
                        env.TestLevel = 'RunLocalTests'
                    }
                    if (source == ''){
                        env.source = 'src'
                    }
                    if (buildType == '') {
                        env.buildType = 'standard'
                    } else if (buildType == 'profilesRetrieval') {
                        env.source = 'src'
                        source = 'src'
                    }
                    if (retrievalCredential == '') {
                        env.retrievalCredential = 'develop'
                    }
                    if(autoUpdatePackage == ''){
                        env.autoUpdatePackage = 'true'
                    }
                }
            }
        }
// verifying that all applications required for deployments are installed on an agent
        stage('Checking application version') {
            steps {
                withAnt(installation: 'Ant', jdk: 'Java 9') {
                    bat 'ant -version'
                    bat 'javac -version'
                    bat 'java -version'
                    bat 'git --version'
                }
            }
        }
        stage('Retrieval') {
            when {
                expression{ env.buildType == 'profilesRetrieval'}
            }
            steps {
                script{
                    env.branch = env.GIT_BRANCH
                }
                checkoutRevisionFile()
                script {
                    env.revision = sh(script: 'cat rev/revision.txt', returnStdout: true).trim()
                }
                script {
                    checkout([$class: 'GitSCM', branches: [[name: env.GIT_BRANCH]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions: [],
                              submoduleCfg: [],
                              userRemoteConfigs: [[credentialsId: 'pocuser', url: 'https://mygithub.gsk.com/sxp14858/pocproject.git']]])
                    def currentDirectory = pwd()
                    println 'current dir ' + currentDirectory

                    withCredentials([usernamePassword(credentialsId: env.retrievalCredential, passwordVariable: 'SFDC_Password', usernameVariable: 'SFDC_Username')]) {
                        sh 'ant retrieveProfiles -f build/build.xml -Dsf.password=$SFDC_Password -Dsf.username=$SFDC_Username -Dsf.src=${source} -Dsf.serverurl=https://test.salesforce.com '
                    }

                    sh 'cd src && git checkout ${GIT_BRANCH} && git config user.email "cec.cicd@gsk.com" && git config user.name "CEC CICD" && git add . && git commit --allow-empty -m "${commitMessage}" && git push origin ${GIT_BRANCH} && cd ..'
                }
            }
        }
//steps for PR only
        stage('Validation'){
            when{
                changeRequest()
            }
            steps{
                script{
                    env.branch = env.CHANGE_TARGET
                }
                checkoutRevisionFile()
                script {
                    env.revision = sh (script: 'cat rev/revision.txt',returnStdout: true).trim()
                }
                checkout([$class: 'GitSCM', branches: [[name: env.CHANGE_BRANCH]],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [[$class: 'PreBuildMerge',
                                        options: [fastForwardMode: 'FF',
                                                  mergeRemote: 'origin', mergeTarget: env.CHANGE_TARGET]]],
                          submoduleCfg: [],
                          userRemoteConfigs:
                                  [[credentialsId: 'pocuser', url: 'https://mygithub.gsk.com/sxp14858/pocproject.git']]])

                //getDifferChanges(revision)

                echo sh(returnStdout: true, script: 'env')

                lock(env.CHANGE_TARGET){
                    script {
                        //def messageWithChanges = sh (script: 'cat changedFiles.txt',returnStdout: true).trim()
                        if (env.CHANGE_TARGET == 'master') {
                            parallel (
                                    "naprod" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('validation', 'naprod', 'prod')
                                        }
                                    },
                                    "euprod" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('validation', 'euprod', 'prod')
                                        }
                                    }
                            )
                        } else if (env.CHANGE_TARGET == 'merge') {
                            parallel (
                                    "namerge" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('validation', 'namerge', 'test')
                                        }
                                    },
                                    "eumerge" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('validation', 'eumerge', 'test')
                                        }
                                    }
                            )
                        } else if (env.CHANGE_TARGET == 'develop') {
                            parallel (
                                    "nadevint" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('validation', 'develop', 'test')
                                        }
                                    },
                                    "eudevint" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('validation', 'eudev', 'test')
                                        }
                                    }
                            )
                        }
                        //def messageWithChanges = sh (script: 'cat changedFiles.txt',returnStdout: true).trim()
                        else {
                            def userCredentials = sh (script: 'echo ${CHANGE_TARGET}',returnStdout: true).trim()
                            userCredentials = userCredentials
                            println 'User Credentials' + userCredentials
                            withAnt(installation: 'Ant', jdk: 'Java 9') {
                                runAntTask('validation', userCredentials, 'test')
                            }
                        }
                    }
                }
            }
        }
        //steps for commit only
        stage('Deployment'){
            when {
                allOf {
                    not { changeRequest() }
                    expression { env.buildType != 'profilesRetrieval' }
                }

            }
            steps{
                script{
                    env.branch = env.GIT_BRANCH
                }
                checkoutRevisionFile()
                script {
                    env.revision = sh (script: 'cat rev/revision.txt',returnStdout: true).trim()
                }
                checkout([$class: 'GitSCM', branches: [[name: env.GIT_BRANCH]],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [],
                          submoduleCfg: [],
                          userRemoteConfigs: [[credentialsId: 'pocuser', url: 'https://mygithub.gsk.com/sxp14858/pocproject.git']]])

                //getDifferChanges(revision)

                echo sh(returnStdout: true, script: 'env')

                lock(env.JOB_BASE_NAME){
                    script {
                        if (env.GIT_BRANCH == 'master') {
                            parallel (
                                    "naprod" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('deploy', 'naprod', 'prod')
                                        }
                                    },
                                    "euprod" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('deploy', 'euprod', 'prod')
                                        }
                                    }
                            )
                        } else if (env.GIT_BRANCH == 'merge') {
                            parallel (
                                    "namerge" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('deploy', 'namerge', 'test')
                                        }
                                    },
                                    "eumerge" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('deploy', 'eumerge', 'test')
                                        }
                                    }
                            )
                        } else if (env.GIT_BRANCH == 'develop') {
                            parallel (
                                    "nadevint" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('deploy', 'develop', 'test')
                                        }
                                    },
                                    "eudevint" : {
                                        withAnt(installation: 'Ant', jdk: 'Java 9') {
                                            runAntTask('deploy', 'eudev', 'test')
                                        }
                                    }
                            )
                        }
                        //def messageWithChanges = sh (script: 'cat changedFiles.txt',returnStdout: true).trim()
                        else if(!GIT_BRANCH.contains('feature/')){
                            def userCredentials = sh (script: 'echo ${GIT_BRANCH}',returnStdout: true).trim()
                            userCredentials = userCredentials
                            println 'User Credentials' + userCredentials
                            withAnt(installation: 'Ant', jdk: 'Java 9') {
                                runAntTask('deploy', userCredentials, 'test')
                            }
                        }
                    }
                }
            }
        }
        stage ('Post Deployment Scripts'){
            when {
                allOf {
                    not { changeRequest() }
                    expression { currentBuild.currentResult == 'SUCCESS' }
                    expression { env.buildType != 'profilesRetrieval' }
                }
            }
            steps {
                lock(env.JOB_BASE_NAME){
                    script {
                        if (env.GIT_BRANCH == 'develop'){
                            runPostDepScripts('develop', 'test')
                        }
                        else if (env.GIT_BRANCH == 'master') {
                            runPostDepScripts('pocorg', 'prod')
                        }
                    }
                }
            }
        }
    }// end of stages
    post {
        success {
            script {
                if(!GIT_BRANCH.contains('PR-') && !GIT_BRANCH.contains('feature/') && !GIT_BRANCH.contains('bugfix/')){

                    def revisionCommit = sh (script: 'cat rev/revision.txt',returnStdout: true).trim()
                    def currentRevision = sh (script: 'echo ${GIT_COMMIT}',returnStdout: true).trim()

                    println revisionCommit
                    println currentRevision

                    if(currentRevision != revisionCommit && env.BranchExist){
                        sshagent (credentials: ["github-cicduser"]) {
                            sh 'cd rev && git checkout ${GIT_BRANCH} && git config user.email "cec.cicd@gsk.com" &&   git config user.name "CEC CICD" && echo ${GIT_COMMIT} > revision.txt  && cat revision.txt && git add . && git commit -m "Commiting ${GIT_COMMIT}" && git push git@mygithub.gsk.com:gsk-tech/Jenkins_Tools_CCM.git ${GIT_BRANCH}'
                        }
                    }else if (!env.BranchExist){
                        sshagent (credentials: ["github-cicduser"]) {
                            sh 'cd rev && git checkout ${GIT_BRANCH} && git config user.email "cec.cicd@gsk.com" &&   git config user.name "CEC CICD" && echo ${GIT_COMMIT} > revision.txt  && cat revision.txt && git add . && git commit -m "Commiting ${GIT_COMMIT}" && git push git@mygithub.gsk.com:gsk-tech/Jenkins_Tools_CCM.git ${GIT_BRANCH}'
                        }
                    }else{
                        println 'No changes found do not commit'
                    }

                }else{
                    println 'No changes for PR to revision files'
                }
            }
        }
        
        cleanup {
            cleanWs()
        }
    }
} //end of pipeline
