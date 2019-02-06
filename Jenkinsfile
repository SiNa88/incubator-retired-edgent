#!groovy

/*
 *
 *  Licensed to the Apache Software Foundation (ASF) under one or more
 *  contributor license agreements.  See the NOTICE file distributed with
 *  this work for additional information regarding copyright ownership.
 *  The ASF licenses this file to You under the Apache License, Version 2.0
 *  (the "License"); you may not use this file except in compliance with
 *  the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 *
 */
pipeline {

    agent {
        node {
            label 'ubuntu'
        }
    }

    environment {
        // It seems the login the jenkins slave uses, doesn't pick up the environment changes,
        // so we have to try to manually add theme here.
        MAVEN_HOME = '/opt/maven'
        PATH = "${MAVEN_HOME}/bin:${env.PATH}"

        PLC4X_BUILD_ON_JENKINS = true
        JENKINS_PROFILE = 'jenkins-build'
        // On non develop build we don't want to pollute the global m2 repo
        MVN_LOCAL_REPO_OPT = '-Dmaven.repo.local=.repository'
        // Test failures will be handled by the jenkins junit steps and mark the build as unstable.
        MVN_TEST_FAIL_IGNORE = '-Dmaven.test.failure.ignore=true'

        JAVA_HOME="${tool 'JDK 1.8 (latest)'}"
    }

    tools {
        maven 'Maven 3 (latest)'
        jdk 'JDK 1.8 (latest)'
    }

    options {
        // Kill this job after one hour.
        timeout(time: 1, unit: 'HOURS')
        // When we have test-fails e.g. we don't need to run the remaining steps
        skipStagesAfterUnstable()
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '3'))
    }

    stages {
        stage('Initialization') {
            steps {
                echo 'Building Branch: ' + env.BRANCH_NAME
                echo 'Using PATH = ' + env.PATH
            }
        }

        stage('Cleanup') {
            steps {
                echo 'Cleaning up the workspace'
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                echo 'Checking out branch ' + env.BRANCH_NAME
                checkout scm
            }
        }

        stage('Build') {
            when {
                expression {
                    env.BRANCH_NAME != 'develop'
                }
            }
            steps {
                echo 'Building'
                sh 'mvn -P${JENKINS_PROFILE} ${MVN_TEST_FAIL_IGNORE} ${MVN_LOCAL_REPO_OPT} clean install'
            }
            post {
                always {
                    junit(testResults: '**/surefire-reports/*.xml', allowEmptyResults: true)
                    junit(testResults: '**/failsafe-reports/*.xml', allowEmptyResults: true)
                }
            }
        }

        stage('Build develop') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Building'
                // We'll deploy to a relative directory so we can save
                // that and deploy in a later step on a different node
                sh 'mvn ${MVN_TEST_FAIL_IGNORE} -P${JENKINS_PROFILE},platform-android,platform-java7,distribution,toolchain -Djava8.home=${JAVA_HOME} -Dedgent.build.ci=true -DaltDeploymentRepository=snapshot-repo::default::file:./local-snapshots-dir clean deploy'
            }
            post {
                always {
                    junit(testResults: '**/surefire-reports/*.xml', allowEmptyResults: true)
                    junit(testResults: '**/failsafe-reports/*.xml', allowEmptyResults: true)
                }
            }
        }

        stage('Code Quality') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Checking Code Quality'
                withSonarQubeEnv('ASF Sonar Analysis') {
                    sh 'mvn -P${JENKINS_PROFILE} sonar:sonar'
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Deploying'
                // Deploy the artifacts using the wagon-maven-plugin.
                sh 'mvn -f jenkins.pom -X -P deploy-snapshots wagon:upload'
            }
        }

        stage ('Build Site') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Building Site'
                sh 'mvn -P${JENKINS_PROFILE} site'
            }
        }


        stage('Stage site') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Staging Site'
                sh 'mvn -P${JENKINS_PROFILE} site:stage'
                // Stash the generated site so we can publish it on the 'git-website' node.
//                stash includes: 'target/staging/**/*', name: 'edgent-site'
            }
        }

        // Commented out for now as currently the site is generated differently / manually
/*        stage('Deploy site') {
            when {
                branch 'develop'
            }
            // Only the nodes labeled 'git-websites' have the credentials to commit to the.
            agent {
                node {
                    label 'git-websites'
                }
            }
            steps {
                echo 'Deploying Site'
                // Clean up the site directory.
                dir("target/staging") {
                    deleteDir()
                }

                // Unstash the previously stashed site.
                unstash 'edgent-site'
                // Publish the site with the scm-publish plugin.
                sh 'mvn -f jenkins.pom -X -P deploy-site scm-publish:publish-scm'

                // Clean up the snapshots directory (freeing up more space after deploying).
                dir("target/staging") {
                    deleteDir()
                }
            }
        }
    }*/

    // Send out notifications on unsuccessful builds.
    post {
        // If this build failed, send an email to the list.
        failure {
            script {
                if(env.BRANCH_NAME == "develop") {
                    emailext(
                        subject: "[BUILD-FAILURE]: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]'",
                        body: """
BUILD-FAILURE: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]':

Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]</a>"
""",
                        to: "dev@edgent.apache.org",
                        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                    )
                }
            }
        }

        // If this build didn't fail, but there were failing tests, send an email to the list.
        unstable {
            script {
                if(env.BRANCH_NAME == "develop") {
                    emailext(
                        subject: "[BUILD-UNSTABLE]: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]'",
                        body: """
BUILD-UNSTABLE: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]':

Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]</a>"
""",
                        to: "dev@edgent.apache.org",
                        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                    )
                }
            }
        }

        // Send an email, if the last build was not successful and this one is.
        success {
            // Cleanup the build directory if the build was successful
            // (in this cae we probably don't have to do any post-build analysis)
            deleteDir()
            script {
                if ((env.BRANCH_NAME == "develop") && (currentBuild.previousBuild != null) && (currentBuild.previousBuild.result != 'SUCCESS')) {
                    emailext (
                        subject: "[BUILD-STABLE]: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]'",
                        body: """
BUILD-STABLE: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]':

Is back to normal.
""",
                        to: "dev@edgent.apache.org",
                        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                    )
                }
            }
        }

        always {
            script {
                if(env.BRANCH_NAME == "master") {
                    emailext(
                        subject: "[COMMIT-TO-MASTER]: A commit to the master branch was made'",
                        body: """
COMMIT-TO-MASTER: A commit to the master branch was made:

Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]</a>"
""",
                        to: "dev@edgent.apache.org",
                        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                    )
                }
            }
        }
    }


}