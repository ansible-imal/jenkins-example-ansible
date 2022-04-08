pipeline {
    agent any
    triggers {
        pollSCM '*/1 * * * *'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    userRemoteConfigs: [[url: "${params.REPOSITORY_URL}"]],
                    branches: [[name: 'refs/heads/master']]
                ], poll: true
            }
        }
        stage('Create App') {
            steps {
                script {
                    openshift.withCluster() {
                        try {
                            def app = openshift.newApp( 'openshift/template.json', "-p", "NAME=${params.APPLICATION}","-p","SOURCE_REPOSITORY_URL=${params.REPOSITORY_URL}"  )
                            sh 'echo no > variable2022_0001.txt'
                            def dc = app.narrow('bc')
                            dc.logs('-f')
                            def bc = app.narrow('dc')
                            bc.logs('-f')
                        } catch (Exception ex) {
                            println(ex.getMessage())
                            sh 'echo yes > variable2022_0001.txt'
                        }
                    }
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    tower_job = ansibleTower(
                    async: true,
                    jobTemplate: 'build',
                    templateType: 'job',
                    towerServer: ${params.REPOSITORY_URL}
                    )
                    println("Tower job "+ tower_job.get("JOB_ID") +" was submitted. Job URL is: "+ tower_job.get("JOB_URL"))        

                }
            }
        }
        stage('Rollout') {
            steps {
                script {
                    def APPEXIST = readFile('variable2022_0001.txt').trim()
                    if (APPEXIST == 'yes'){
                        openshift.withCluster() {
                            def dc = openshift.selector( "dc/${params.APPLICATION}" )
                            try {
                                def rollout = dc.rollout()
                                rollout.latest()
                            } catch (Exception ex) {
                                println(ex.getMessage())
                                dc.logs('-f')
                            }
                        }
                    }
                }
            }
        }
    }
}
