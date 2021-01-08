pipeline {

    agent {
        // label "" also could have been 'agent any' - that has the same meaning.
        label "master"
    }

    environment {
        // Global Vars
        NAMESPACE_PREFIX="<YOUR_NAME>"
        GITLAB_DOMAIN = "<GITLAB_FQDN>"
        GITLAB_USERNAME = "<GITLAB_USERNAME>"

        PIPELINES_NAMESPACE = "${NAMESPACE_PREFIX}-ci-cd"
        APP_NAME = "todolist"

        JENKINS_TAG = "${JOB_NAME}.${BUILD_NUMBER}".replace("/", "-")
        JOB_NAME = "${JOB_NAME}".replace("/", "-")

        GIT_SSL_NO_VERIFY = true
        GIT_CREDENTIALS = credentials("${NAMESPACE_PREFIX}-ci-cd-git-auth")
    }

    // The options directive is for configuration that applies to the whole job.
    options {
        buildDiscarder(logRotator(numToKeepStr:'5'))
        timeout(time: 15, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
    }

    stages {
        stage("prepare environment for master deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
              expression { GIT_BRANCH ==~ /(.*master)/ }
            }
            steps {
                script {
                    // Arbitrary Groovy Script executions can do in script tags
                    env.PROJECT_NAMESPACE = "${NAMESPACE_PREFIX}-test"
                    env.NODE_ENV = "test"
                    env.E2E_TEST_ROUTE = "oc get route/${APP_NAME} --template='{{.spec.host}}' -n ${PROJECT_NAMESPACE}".execute().text.minus("'").minus("'")
                }
            }
        }
        stage("prepare environment for develop deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
              expression { GIT_BRANCH ==~ /(.*develop)/ }
            }
            steps {
                script {
                    // Arbitrary Groovy Script executions can do in script tags
                    env.PROJECT_NAMESPACE = "${NAMESPACE_PREFIX}-dev"
                    env.NODE_ENV = "dev"
                    env.E2E_TEST_ROUTE = "oc get route/${APP_NAME} --template='{{.spec.host}}' -n ${PROJECT_NAMESPACE}".execute().text.minus("'").minus("'")
                }
            }
        }
        stage("node-build") {
            agent {
                node {
                    label "jenkins-agent-npm"  
                }
            }
            steps {
                sh 'printenv'

                echo '### Install deps ###'
                sh 'npm install'

                echo '### Running tests ###'
                // sh 'npm run test:all:ci'

                echo '### Running build ###'
                sh 'npm run build:ci'

                echo '### Packaging App for Nexus ###'
                sh 'npm run package'
                sh 'npm run publish'
                stash 'source'
            }
            // Post can be used both on individual stages and for the entire build.
            post {
                always {
                    archive "**"
                    // ADD TESTS REPORTS HERE
            
                    // publish html

                }
                success {
                    echo "Git tagging"
                    script {
                      env.ENCODED_PSW=URLEncoder.encode(GIT_CREDENTIALS_PSW, "UTF-8")
                    }
                    sh'''
                        git config --global user.email "jenkins@jmail.com"
                        git config --global user.name "jenkins-ci"
                        git tag -a ${JENKINS_TAG} -m "JENKINS automated commit"
                        git push https://${GIT_CREDENTIALS_USR}:${ENCODED_PSW}@${GITLAB_DOMAIN}/${GITLAB_USERNAME}/${APP_NAME}.git --tags
                    '''
                }
                failure {
                    echo "FAILURE"
                }
            }
        }

        stage("node-bake") {
            agent {
                node {
                    label "master"  
                }
            }
            when {
                expression { GIT_BRANCH ==~ /(.*master|.*develop)/ }
            }
            steps {
                echo '### Get Binary from Nexus ###'
                sh  '''
                        rm -rf package-contents*
                        curl -v -f http://admin:admin123@${NEXUS_SERVICE_HOST}:${NEXUS_SERVICE_PORT}/repository/zip/com/redhat/todolist/${JENKINS_TAG}/package-contents.zip -o package-contents.zip
                        unzip package-contents.zip
                    '''
                echo '### Create Linux Container Image from package ###'
                sh  '''
                        oc project ${PIPELINES_NAMESPACE} # probs not needed
                        oc patch bc ${APP_NAME} -p "{\\"spec\\":{\\"output\\":{\\"to\\":{\\"kind\\":\\"ImageStreamTag\\",\\"name\\":\\"${APP_NAME}:${JENKINS_TAG}\\"}}}}"
                        oc start-build ${APP_NAME} --from-dir=package-contents/ --follow
                    '''
            }
            // this post step chews up space. uncomment if you want all bake artefacts archived
            // post {
                //always {                    
                    // archive "**"
                //}
            //}
        }

        stage("node-deploy") {
            agent {
                node {
                    label "master"  
                }
            }
            when {
                expression { GIT_BRANCH ==~ /(.*master|.*develop)/ }
            }
            steps {
                script {
                  openshift.withCluster() {
                    openshift.withProject("${PROJECT_NAMESPACE}") {
                      echo '### tag image for namespace ###'
                      openshift.tag("${PIPELINES_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}", "${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}")
                      
                      echo '### set env vars and image for deployment ###'
                      openshift.raw("set","env","dc/${APP_NAME}","NODE_ENV=${NODE_ENV}")
                      openshift.raw("set", "image", "dc/${APP_NAME}", "${APP_NAME}=image-registry.openshift-image-registry.svc:5000/${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}")

                      echo '### Rollout and Verify OCP Deployment ###'
                      openshift.selector("dc", "${APP_NAME}").rollout().latest()
                      openshift.selector("dc", "${APP_NAME}").rollout().status("-w")
                      openshift.selector("dc", "${APP_NAME}").scale("--replicas=1")
                      openshift.selector("dc", "${APP_NAME}").related('pods').untilEach("1".toInteger()) {
                        return (it.object().status.phase == "Running")
                      }
                    }
                  }
                }
            }
        }
    }
}
