pipeline {
    agent any

    stages {
        stage('Determine Environment') {
            steps {
                script {
                    // Logic to map Branch -> Environment (Dev/Stg/Prod)
                    if (env.REF == 'refs/heads/main') {
                        env.DEPLOY_ENV = "prod"
                    } else if (env.REF == 'refs/heads/staging') {
                        env.DEPLOY_ENV = "stg"
                    } else {
                        env.DEPLOY_ENV = "dev"
                    }
                    
                    echo "Deploying component: ${env.COMPONENT} to environment: ${env.DEPLOY_ENV}"
                    echo "Image: ${env.IMAGE}"
                }
            }
        }

        stage('Deploy to Swarm') {
            steps {
                script {
                    def stackName = "${env.DEPLOY_ENV}"
                    
                    // Auto-add registry prefix if missing
                    def fullImageName = env.IMAGE
                    if (!fullImageName.contains("registry.gaumeodathanh.me/")) {
                        fullImageName = "registry.gaumeodathanh.me/${env.IMAGE}"
                    }
                    
                    echo "Updating Component: ${env.COMPONENT} in Stack: ${stackName} with Image: ${fullImageName}"
        
                    sh """
                        cd /workspace/config/swarm
                        
                        if [ "${env.COMPONENT}" = "frontend" ]; then
                            echo "Updating frontend service..."
                            docker service update \\
                                --image ${fullImageName} \\
                                --with-registry-auth \\
                                --update-parallelism 1 \\
                                --update-delay 10s \\
                                ${stackName}_frontend
                                
                        elif [ "${env.COMPONENT}" = "backend" ]; then
                            echo "Updating backend service..."
                            docker service update \\
                                --image ${fullImageName} \\
                                --with-registry-auth \\
                                --update-parallelism 1 \\
                                --update-delay 10s \\
                                ${stackName}_backend
                        fi
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def stackName = "${env.DEPLOY_ENV}"
                    def serviceName = "${stackName}_${env.COMPONENT}"
                    
                    // Skip verification for 'both' component
                    if (env.COMPONENT == 'both') {
                        serviceName = "${stackName}_frontend ${stackName}_backend"
                    }
                    
                    echo "Verifying deployment for: ${serviceName}"
                    
                    // Wait a bit for services to update
                    sleep(time: 15, unit: 'SECONDS')
                    
                    // Check specific service status
                    sh """
                        echo "=== Service Status ==="
                        if [ "${env.COMPONENT}" = "both" ]; then
                            docker service ls --filter name=${stackName}
                        else
                            docker service ls --filter name=${stackName}_${env.COMPONENT}
                        fi
                        
                        echo "=== Service Tasks ==="
                        if [ "${env.COMPONENT}" = "both" ]; then
                            docker service ps ${stackName}_frontend --no-trunc
                            docker service ps ${stackName}_backend --no-trunc
                        else
                            docker service ps ${stackName}_${env.COMPONENT} --no-trunc
                        fi
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo "Deployment Successful! Stack: ${env.DEPLOY_ENV}, Component: ${env.COMPONENT}, Image: ${env.IMAGE}"
        }
        failure {
            script {
                echo "Deployment Failed. Service details:"
                sh """
                    echo "=== Stack Services ==="
                    docker stack services ${env.DEPLOY_ENV} || true
                    
                    echo "=== Service Logs ==="
                    if [ "${env.COMPONENT}" != "both" ]; then
                        docker service logs ${env.DEPLOY_ENV}_${env.COMPONENT} --tail 50 || true
                    else
                        echo "Frontend logs:"
                        docker service logs ${env.DEPLOY_ENV}_frontend --tail 25 || true
                        echo "Backend logs:"
                        docker service logs ${env.DEPLOY_ENV}_backend --tail 25 || true
                    fi
                    
                    echo "=== Failed Tasks ==="
                    if [ "${env.COMPONENT}" != "both" ]; then
                        docker service ps ${env.DEPLOY_ENV}_${env.COMPONENT} --filter "desired-state=running" --no-trunc || true
                    fi
                """
            }
        }
    }
}
