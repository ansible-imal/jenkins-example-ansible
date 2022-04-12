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
                    userRemoteConfigs: [[url: "${params.SOURCE_REPOSITORY_URL}"]],
                    branches: [[name: "refs/heads/${params.SOURCE_REPOSITORY_REF}"]]
                ], poll: true
            }
        }
        stage('Add Image Stream'){
            steps {
                script {
                    openshift.withCluster() {
                        def secret = [
                            "kind": "ImageStream",
                            "apiVersion": "v1",
                            "metadata": [
                                "name": "${params.APP_NAME}",
                                "annotations":[
                                    "description": "Keeps track of changes in the application image"
                                ]
                            ]
                        ]
                        try{
                            def imgStream = openshift.selector( "is/${params.APP_NAME}" )
                            println('IS exists')
                        }catch (Exception ex) {
                            println('IS not exists')
                            println(ex.getMessage())
                            imgStream = openshift.create( secret, '--save-config', '--validate' )
                        }
                    }
                }
            }
        }
        stage('Build Image'){
            steps {
                script {
                    openshift.withCluster() {
                        def bcParam = [
                        "kind": "BuildConfig",
                        "apiVersion": "v1",
                        "metadata": [
                            "name": "${params.APP_NAME}",
                            "annotations": [
                            "description": "Defines how to build the application",
                            "template.alpha.openshift.io/wait-for-ready": "true"
                            ]
                        ],
                        "spec": [
                            "source": [
                            "contextDir": "${params.CONTEXT_DIR}",
                            "type": "Git",
                            "git": [
                                "uri": "${params.SOURCE_REPOSITORY_URL}",
                                "ref": "${params.SOURCE_REPOSITORY_REF}"
                            ]
                            ],
                            "strategy": [
                            "type": "Source",
                            "sourceStrategy": [
                                "from": [
                                "kind": "ImageStreamTag",
                                "namespace": "openshift",
                                "name": "php:${params.PHP_VERSION}"
                                ],
                                "env": [
                                [
                                    "name": "COMPOSER_MIRROR",
                                    "value": "${params.COMPOSER_MIRROR}"
                                ]
                                ]
                            ]
                            ],
                            "output": [
                            "to": [
                                "kind": "ImageStreamTag",
                                "name": "${params.APP_NAME}:latest"
                            ]
                            ],
                            "triggers": [
                            [
                                "type": "ImageChange"
                            ],
                            [
                                "type": "ConfigChange"
                            ],
                            [
                                "github": [
                                "secret": "{{ lookup('password', '/tmp/passwordfile length=40 chars=digits') }}"
                                ],
                                "type": "GitHub"
                            ]
                            ]
                        ]

                        ]
                        def bc = null
                        try{
                            bc = openshift.selector( "bc/${params.APP_NAME}" )
                            println('BC exists')
                            
                        }catch (Exception ex) {
                            println('BC not exists')
                            println(ex.getMessage())
                            bc = openshift.create( bcParam, '--save-config', '--validate' )
                        }
                        // try{
                        //     def result = bc.startBuild()
                        //     timeout(10) {
                        //         result.logs('-f')
                        //     }
                        // }catch (Exception ex) {
                        //     println("can not start build")
                        //     println(ex.getMessage())
                        // }
                        

                
                        // create will marshal the model into JSON and send it to the API server.
                        // We will add some passthrough arguments (--save-config and --validate)
                    }
                }
            }   
        }
        stage('Deploy With Ansible') {
            steps {
                script {
                    tower_job = ansibleTower(
                        async: false,
                        jobTemplate: 'DC From Jenkins',
                        templateType: 'job',
                        towerServer: 'ansible_openshift',
                        extraVars: '''---
        app_name: "'''+"${params.APP_NAME}"+'''"
        deployment_environment: "'''+"${params.DEPLOYMENT_ENVIRONMENT}"+'''"
        app_debug: false
        namespace: "'''+"${params.NAMESPACE}"+'''"''',
                    )
                    println("Tower job "+ tower_job.get("JOB_ID") +" was submitted. Job URL is: "+ tower_job.get("JOB_URL"))        
                } 
            }
        }
    }
}


