import org.jenkinsci.plugins.configfiles.GlobalConfigFiles
import org.jenkinsci.lib.configprovider.model.Config

Freestring = 'Satu'
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
        stage('Add Image Stream') {
            steps {
                script {
                    openshift.withCluster() {
                        secret = [
                        'kind': 'ImageStream',
                        'apiVersion': 'v1',
                        'metadata': [
                            'name': "${params.APP_NAME}",
                            'annotations':[
                                'description': 'Keeps track of changes in the application image'
                            ]
                        ]
                    ]
                        try {
                            is = openshift.selector( "is/${params.APP_NAME}" )
                            if (!is.exists()) {
                                println('IS not exists, creating new IS')
                                is = openshift.create( secret, '--save-config', '--validate' )
                                is.describe()
                                println('IS exists')
                            }
                    }catch (Exception ex) {
                            println('IS not exists, cant create')
                            println(ex.getMessage())
                        }
                    }
                }
            }
        }
        stage('Add Build Config') {
            steps {
                script {
                    openshift.withCluster() {
                        bcParam = [
                            'kind': 'BuildConfig',
                            'apiVersion': 'v1',
                            'metadata': [
                                'name': "${params.APP_NAME}",
                                'annotations': [
                                'description': 'Defines how to build the application',
                                'template.alpha.openshift.io/wait-for-ready': 'true'
                                ]
                            ],
                            'spec': [
                                'source': [
                                'contextDir': "${params.CONTEXT_DIR}",
                                'type': 'Git',
                                'git': [
                                    'uri': "${params.SOURCE_REPOSITORY_URL}",
                                    'ref': "${params.SOURCE_REPOSITORY_REF}"
                                ]
                                ],
                                'strategy': [
                                'type': 'Source',
                                'sourceStrategy': [
                                    'from': [
                                    'kind': 'ImageStreamTag',
                                    'namespace': 'openshift',
                                    'name': "php:${params.PHP_VERSION}"
                                    ],
                                    'env': [
                                    [
                                        'name': 'COMPOSER_MIRROR',
                                        'value': "${params.COMPOSER_MIRROR}"
                                    ]
                                    ]
                                ]
                                ],
                                'output': [
                                'to': [
                                    'kind': 'ImageStreamTag',
                                    'name': "${params.APP_NAME}:latest"
                                ]
                                ],
                                'triggers': [
                                [
                                    'type': 'ImageChange'
                                ],
                                [
                                    'type': 'ConfigChange'
                                ],
                                [
                                    'github': [
                                    'secret': 'qazxs10298qazxs10298qazxs10298qazxs10298'
                                    ],
                                    'type': 'GitHub'
                                ]
                                ]
                            ]

                        ]
                        bc = null
                        builds = null
                        needBuild = true
                        try {                            
                            bc = openshift.selector( "bc/${params.APP_NAME}" )
                            builds = bc.related('builds')                            
                            if (!bc.exists()) {
                                println('BC not exists')
                                bc = openshift.create( bcParam, '--save-config', '--validate' )
                                bc.describe()
                                builds = bc.related('builds')
                                needBuild = false
                                timeout(10) {
                                    builds.logs('-f')
                                }
                            }
                        } catch (Exception ex) {
                            println('Error')
                            println(message)
                            currentBuild.result = 'ABORTED'
                            error('Build Failed')
                        }
                        try {
                            if ( needBuild || builds.count() < 1 ) {
                                // Start Build
                                result = null
                                result = bc.startBuild()
                                timeout(10) {
                                    result.logs('-f')
                                }
                            }
                        }catch (Exception ex) {
                            println('Already Built, keep continue')
                            println(message)
                        // currentBuild.result = 'ABORTED'
                        // error('Build Failed')
                        }

                    }
                }
            }
        }
        stage('Execute') {
            steps {
                script {
                    tower_job = ansibleTower(
                        async: false,
                        jobTemplate: "${params.ANSIBLE_WORKFLOW}",
                        templateType: 'workflow',
                        towerServer: 'ansible_openshift',
                        extraVars: '''---
        app_name: "'''+"${params.APP_NAME}"+'''"
        namespace: "'''+"${params.NAMESPACE}"+'''"
        deployment_environment: "'''+"${params.SOURCE_REPOSITORY_REF}"+'''"
        app_debug: false
        namespace: "'''+"${params.NAMESPACE}"+'''"''',
                    )
                }
            }
        }
    }

}