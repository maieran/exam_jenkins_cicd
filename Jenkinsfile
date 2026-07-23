def synchronizeDatabaseSecrets(String namespace, String environmentName) {
    withEnv([
        "TARGET_NAMESPACE=${namespace}",
        "DB_ENVIRONMENT=${environmentName}"
    ]) {
        withCredentials([
            usernamePassword(
                credentialsId: 'cast-db-credentials',
                usernameVariable: 'CAST_DB_USER',
                passwordVariable: 'CAST_DB_PASSWORD'
            ),
            usernamePassword(
                credentialsId: 'movie-db-credentials',
                usernameVariable: 'MOVIE_DB_USER',
                passwordVariable: 'MOVIE_DB_PASSWORD'
            )
        ]) {
            sh '''
                set -eu
                set +x
                

                CAST_DB_NAME="cast_db_${DB_ENVIRONMENT}"
                MOVIE_DB_NAME="movie_db_${DB_ENVIRONMENT}"

                kubectl create namespace "$TARGET_NAMESPACE" \
                    --dry-run=client \
                    --output=yaml |
                    kubectl apply -f -
                
                kubectl create secret generic exam-cast-credentials \
                    --namespace "$TARGET_NAMESPACE" \
                    --from-literal=cast-db-user="$CAST_DB_USER" \
                    --from-literal=cast-db-password="$CAST_DB_PASSWORD" \
                    --from-literal=cast-db-name="$CAST_DB_NAME" \
                    --from-literal=cast-database-uri="postgresql://${CAST_DB_USER}:${CAST_DB_PASSWORD}@jenkins-psql-cast-db:5432/${CAST_DB_NAME}" \
                    --dry-run=client \
                    --output=yaml |
                    kubectl apply -f -

                kubectl create secret generic exam-movie-credentials \
                    --namespace "$TARGET_NAMESPACE" \
                    --from-literal=movie-db-user="$MOVIE_DB_USER" \
                    --from-literal=movie-db-password="$MOVIE_DB_PASSWORD" \
                    --from-literal=movie-db-name="$MOVIE_DB_NAME" \
                    --from-literal=movie-database-uri="postgresql://${MOVIE_DB_USER}:${MOVIE_DB_PASSWORD}@jenkins-psql-movie-db:5432/${MOVIE_DB_NAME}" \
                    --dry-run=client \
                    --output=yaml |
                    kubectl apply -f -

                echo "Database Secrets updated in namespace: $TARGET_NAMESPACE"
            '''
        }
    }
}

def deployEnvironment(
    String namespace,
    String environmentName,
    String releaseName,
    String valuesFile

) {
    synchronizeDatabaseSecrets(namespace, environmentName)

    withEnv([
        "TARGET_NAMESPACE=${namespace}",
        "TARGET_RELEASE=${releaseName}",
        "VALUES_FILE=${valuesFile}"
    ]) {
        sh '''
            set -eu

            helm upgrade --install "$TARGET_RELEASE" "$CHART_PATH" \
                --namespace "$TARGET_NAMESPACE" \
                --values "$CHART_PATH/$VALUES_FILE" \
                --set-string images.castService.tag="$IMAGE_TAG" \
                --set-string images.movieService.tag="$IMAGE_TAG" \
                --wait \
                --atomic \
                --timeout 5m
        '''
    }
}

def smokeTestEnvironment(String namespace, String releaseName) {
    withEnv([
        "TARGET_NAMESPACE=${namespace}",
        "TARGET_RELEASE=${releaseName}"
    ]) {
        sh '''
            set -eu

            helm test "$TARGET_RELEASE" \
                --namespace "$TARGET_NAMESPACE" \
                --logs \
                --timeout 2m
        '''
    }
}

pipeline {
    agent any

    options {
        //Prevents automatical cloning of the source code, therefore the checkout will be performed manually after cleaning the workspace
        // because you will see the 'checkout scm' at the end, to manually pull code when we actually need it
        skipDefaultCheckout(true)

        // Prevents two builds from deploying simultaneously
        disableConcurrentBuilds()

        timestamps()

        buildDiscarder(
            logRotator(
                numToKeepStr: '20',
                artifactNumToKeepStr: '10'
            )
        )

        timeout(time: 90, unit: 'MINUTES')
    }

    environment {
        DOCKER_ID = "andrem43"
        FAILURE_EMAIL = 'maierandre43@gmail.com'

        CHART_PATH = 'microservice-charts/exam-microservices'
        
        CAST_IMAGE = 'andrem43/exam-cast-service-jenkins'
        MOVIE_IMAGE = 'andrem43/exam-movie-service-jenkins'
        
        DEV_NAMESPACE = 'jenkins-dev-exam'
        QA_NAMESPACE = 'jenkins-qa-exam'
        STAGING_NAMESPACE = 'jenkins-staging-exam'
        PROD_NAMESPACE = 'jenkins-prod-exam'

        

        DEV_RELEASE = 'exam-app-dev'
        QA_RELEASE = 'exam-app-qa'
        STAGING_RELEASE = 'exam-app-staging'
        PROD_RELEASE = 'exam-app-prod'

        // Jenkins credential type: Secret file; credential ID: config
        KUBECONFIG = credentials('config')
    }

    stages {
        stage("Clean Workspace and Checkout") {
            steps {

                // Clean Workspace part
                cleanWs(
                    //
                    deleteDirs: true,
                    //deletion takes place asynchronously to your build with the help of Resource Disposer Plugin (most of the time already pre-installed)
                    disableDeferredWipeout: true
                )

                //Checkout part
                checkout scm
            }
           
        }

        stage('Set Image Tag') {
            steps {
                script {
                    env.GIT_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = env.GIT_COMMIT.take(8)
                }

                echo "Git commit: ${env.GIT_COMMIT}"
                echo "Docker image tag: ${env.IMAGE_TAG}"
            }
        }

        stage("Test Cast Service") {
            steps {
                // TODO: replace this placeholder with the real pytest command.
                echo 'Run cast-service tests, but not implemented yet.'
            }
        }

        stage("Test Movie Service") {
            steps {
                // TODO: replace this placeholder with the real pytest command.
                echo 'Run movie-service tests, but not implemented yet.'
            }
        }

        //We test whether the chart is well-formed
        stage('Helm Lint') {
            steps {
                sh '''
                    set -eu

                    for values_file in \
                        values-dev.yml \
                        values-qa.yml \
                        values-staging.yml \
                        values-prod.yml
                    do
                        echo "Linting $values_file"
                        helm lint "$CHART_PATH" \
                            --values "$CHART_PATH/$values_file"
                    done
                '''
            }
        }

        stage('Verify Kubernetes Access') {
            steps {
                sh '''
                    set -eu

                    test -f "$KUBECONFIG"
                    kubectl cluster-info
                    kubectl get nodes
                '''
            }
        }

        stage('Helm Template Validation') {
            steps {
                sh '''
                    set -eu

                    validate_environment() {
                        namespace="$1"
                        release="$2"
                        values_file="$3"
                        output_file="$4"

                        kubectl create namespace "$namespace" \
                            --dry-run=client \
                            --output=yaml |
                        kubectl apply -f -

                    helm template "$release" "$CHART_PATH" \
                      --namespace "$namespace" \
                      --values "$CHART_PATH/$values_file" \
                      --set-string images.castService.tag="$IMAGE_TAG" \
                      --set-string images.movieService.tag="$IMAGE_TAG" \
                      > "$output_file"

                    if grep -nE \
                        '<no value>|\\{\\{|imagePullPolicy:[[:space:]]*$' \
                        "$output_file"
                    then
                        echo "Invalid or unresolved value found in $output_file"
                        exit 1
                    fi


                      kubectl apply \
                        --dry-run=server \
                        --namespace "$namespace" \
                        --filename "$output_file"
                    }

                    validate_environment \
                        "$DEV_NAMESPACE" \
                        "$DEV_RELEASE" \
                        values-dev.yml \
                        rendered-dev.yml

                    validate_environment \
                        "$QA_NAMESPACE" \
                        "$QA_RELEASE" \
                        values-qa.yml \
                        rendered-qa.yml

                    validate_environment \
                        "$STAGING_NAMESPACE" \
                        "$STAGING_RELEASE" \
                        values-staging.yml \
                        rendered-staging.yml

                    validate_environment \
                        "$PROD_NAMESPACE" \
                        "$PROD_RELEASE" \
                        values-prod.yml \
                        rendered-prod.yml
                '''
            }
        }

        stage('Build Cast and Movie images') {
            steps {
                script {
                    sh '''
                        set -eu 

                        docker build \
                            --tag ${CAST_IMAGE}:$IMAGE_TAG \
                            ./cast-service/
                        docker build \
                            --tag ${MOVIE_IMAGE}:$IMAGE_TAG \
                            ./movie-service/
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                        set -eu
                        set +x
                        
                        printf '%s' "$DOCKER_PASS" |
                        docker login \
                            --username "$DOCKER_ID" \
                            --password-stdin

                        docker push "$CAST_IMAGE:$IMAGE_TAG"
                        docker push "$MOVIE_IMAGE:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Helm Deploy Development') {
            // environment {
            //     KUBECONFIG = credentials("config")
            // }
            steps {
                script {
                    deployEnvironment(
                        env.DEV_NAMESPACE,
                        'dev',
                        env.DEV_RELEASE,
                        'values-dev.yml'
                    )
                    // synchronizeDatabaseSecrets(
                    //     env.DEV_NAMESPACE,
                    //     'dev'
                    // )

                    // sh '''
                    //     rm -rf ~/.kube
                    //     mkdir ~/.kube
                    //     ls
                    //     cat $KUBECONFIG > ~/.kube/config
                    //     helm upgrade --install ${DEV_RELEASE} "$CHART_PATH" \
                    //         --namespace ${DEV_NAMESPACE} \
                    //         --values "$CHART_PATH/values-dev.yml" \
                    //         --set-string images.castService.tag="$IMAGE_TAG" \
                    //         --set-string.images.movieService.tag="$IMAGE_TAG" \
                    //         --wait \
                    //         --timeout 5m
                    // '''
                }

            }
        }

        stage('Helm Smoke Test Development') {
            steps {
                script {
                    smokeTestEnvironment(
                        env.DEV_NAMESPACE,
                        env.DEV_RELEASE
                    )
                }
                // sh '''
                //     helm test "$DEV_RELEASE" \
                //         --namespace "$DEV_NAMESPACE"
                //         --logs \
                //         --timeout 2m
                // '''
            }
        }


        stage('Helm Deploy QA') {
            // environment {
            //     KUBECONFIG = credentials("config")
            // }
            steps {
                script {

                    deployEnvironment(
                        env.QA_NAMESPACE,
                        'qa',
                        env.QA_RELEASE,
                        'values-qa.yml'
                    )
                    // synchronizeDatabaseSecrets(
                    //     env.DEV_NAMESPACE,
                    //     'qa'
                    // )

                    // sh '''
                    //     rm -rf ~/.kube
                    //     mkdir ~/.kube
                    //     ls
                    //     cat $KUBECONFIG > ~/.kube/config
                    //     helm upgrade --install ${QA_RELEASE} "$CHART_PATH" \
                    //         --namespace ${QA_NAMESPACE} \
                    //         --values "$CHART_PATH/values-qa.yml" \
                    //         --set-string images.castService.tag="$IMAGE_TAG" \
                    //         --set-string.images.movieService.tag="$IMAGE_TAG" \
                    //         --wait \
                    //         --timeout 5m
                    // '''


                }

            }
        }

        stage('Helm Smoke Test QA') {
            steps {
                script {
                    smokeTestEnvironment(
                        env.QA_NAMESPACE,
                        env.QA_RELEASE
                    )
                }
            }
        }

        stage('Helm Deploy Staging') {
            // environment {
            //     KUBECONFIG = credentials("config")
            // }
            steps {
                script {

                    deployEnvironment(
                        env.STAGING_NAMESPACE,
                        'staging',
                        env.STAGING_RELEASE,
                        'values-staging.yml'
                    )
                }

            }
        }

        stage('Helm Smoke Test Staging') {
            steps {
                script {
                    smokeTestEnvironment(
                        env.STAGING_NAMESPACE,
                        env.STAGING_RELEASE
                    )
                }
                // sh '''
                //     helm test "$STAGING_RELEASE" \
                //         --namespace "$STAGING_NAMESPACE"
                //         --logs \
                //         --timeout 2m
                // '''

            }
        }

        stage('Production Approval') {
            steps {
                timeout(time: 20, unit: 'MINUTES') {
                    input(
                        message: "Deploy image tag ${env.IMAGE_TAG} to production?",
                        ok: 'Deploy'
                    )
                }
            }
        }

        stage('Helm Deploy Prod') {
            steps {
                script {
                    deployEnvironment(
                        env.PROD_NAMESPACE,
                        'prod',
                        env.PROD_RELEASE,
                        'values-prod.yml'
                    )
                }
            }

            // environment {
            //     KUBECONFIG = credentials("config")
            // }
            // steps {
            //     timeout(time: 15, unit: "MINUTES") {
            //         input message: 'Do you want to deploy in production ?', ok: 'Yes'
            //     }
            //     script {
            //         synchronizeDatabaseSecrets(
            //             env.DEV_NAMESPACE,
            //             'staging'
            //         )

            //         sh '''
            //             rm -rf ~/.kube
            //             mkdir ~/.kube
            //             ls
            //             cat $KUBECONFIG > ~/.kube/config
            //             helm upgrade --install ${STAGING_RELEASE} "$CHART_PATH" \
            //                 --namespace ${STAGING_NAMESPACE} \
            //                 --values "$CHART_PATH/values-dev.yml" \
            //                 --set-string images.castService.tag="$IMAGE_TAG" \
            //                 --set-string.images.movieService.tag="$IMAGE_TAG" \
            //                 --wait \
            //                 --timeout 5m
            //         '''
            //     }

            // }
        }

        stage('Helm Smoke Test Prod') {
            steps {
                script {
                    smokeTestEnvironment(
                        env.PROD_NAMESPACE,
                        env.PROD_RELEASE
                    )
                }

                // sh '''
                //     helm test "$STAGING_RELEASE" \
                //         --namespace "$STAGING_NAMESPACE"
                //         --logs \
                //         --timeout 2m
                // '''
            }
        }
    }
    post {
        always {
            echo "Pipeline result: ${currentBuild.currentResult}"

            archiveArtifacts(
                artifacts: 'rendered-*.yml',
                allowEmptyArchive: true
            )
        }

        success {
            echo 'CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'CI/CD pipeline failed.'

            script {
                try {
                    mail(
                        to: env.FAILURE_EMAIL,
                        subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} failed",
                        body: "Pipeline failure details: ${env.BUILD_URL}console"
                    )
                } catch (Exception mailError) {
                    echo "Failure email could not be sent: ${mailError.message}"
                }
            }
        }

        aborted {
            echo 'CI/CD pipeline was aborted.'
        }

        cleanup {
            sh '''
                set +e

                docker logout >/dev/null 2>&1 || true

                if [ -n "${IMAGE_TAG:-}" ]; then
                    docker image rm \
                        "$CAST_IMAGE:$IMAGE_TAG" \
                        "$MOVIE_IMAGE:$IMAGE_TAG" \
                        >/dev/null 2>&1 || true
                fi
            '''

            cleanWs(
                deleteDirs: true,
                disableDeferredWipeout: true,
                notFailBuild: true
            )
        }
    }
}