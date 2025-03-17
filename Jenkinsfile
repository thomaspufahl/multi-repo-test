// Master
def dsContent = ""

def sendDiscordNotification(message) {
    def json = new groovy.json.JsonBuilder(message).toPrettyString()
    sh """
        curl -X POST \
            -H "Content-Type: application/json" \
            -d '${json}' \
            "${DISCORD_WEBHOOK_URL}"
    """
}

def waitForAppStartup(url, retries, delay) {
    for (int i = 0; i < retries; i++) {
        try {
            def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://${url}", returnStdout: true).trim()
            if (response == "200") {
                echo "Application is up and running!"
                return; // Success!
            } else {
                echo "Attempt ${i + 1}: Application not yet ready (Status: ${response}). Retrying in ${delay} seconds..."
                sleep delay
            }
        } catch (Exception e) {
            echo "Attempt ${i + 1}: Error checking application status: ${e.message}. Retrying in ${delay} seconds..."
            sleep delay
        }
    }
    throw new Exception("Application startup timed out after ${retries} retries.")
}

pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'registry.antaresautomation.com'
        DOCKER_IMAGE = 'antares-altair'
        // DOCKER_TAG = 'latest'
        // DOCKER_CONTAINER_NAME = 'antares-altair-qa'

        REPO_URL = 'https://github.com/antaresautomation/antares_hourImputations.git'
        // REPO_BRANCH = 'master'
        // REPO_BRANCH = 'qa-environment'
        REPO_CREDENTIALS = 'github-thomaspufahl'

        CONFIG_PUBLIC_URL = "altair.antaresautomation.com"

        // PROD DATABASE
        CONFIG_DATABASE_URL = "mysql://altair:1Lw,wxgea.}rzWb@mysql:3306/altair"

        CONFIG_PORT = 3002

        CONFIG_EMAIL_PASS = "XpQK~0KaO6XK"

        DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/1333894803850657924/bB4JTfVlFw3BKdclv4KqsuPrQBWomjg_CWOkXDYHCwOpb_vgqOS3H8VLxnuN5bWUTj72"
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    try {
                        git branch: "${REPO_BRANCH}",
                            url: "${REPO_URL}",
                            credentialsId: "${REPO_CREDENTIALS}"
                    } catch (Exception e) {
                        dsContent = e.message
                        error "Error en Checkout: ${e.message}"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    try {
                        docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    } catch (Exception e) {
                        dsContent = e.message
                        error "Error en Build: ${e.message}"
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    try {
                        docker.withRegistry("https://${DOCKER_REGISTRY}", 'registry') {
                            docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        }
                    } catch (Exception e) {
                        dsContent = "Error en Push: ${e.message}"
                        error dsContent
                    }
                }
            }
        }
        
        stage('Update Container') {
            steps {
                script {
                    try {
                        // Detener y eliminar el contenedor existente (si existe)
                        sh "docker stop ${DOCKER_CONTAINER_NAME} || true"
                        sh "docker rm ${DOCKER_CONTAINER_NAME} || true"

                        // Crear y ejecutar un nuevo contenedor con la imagen construida
                        sh """
                        docker run -d \
                          --name ${DOCKER_CONTAINER_NAME} \
                          --restart unless-stopped \
                          --label 'traefik.enable=true' \
                          --label 'traefik.http.routers.${DOCKER_CONTAINER_NAME}.rule=Host(`${CONFIG_PUBLIC_URL}`)' \
                          --label 'traefik.http.routers.${DOCKER_CONTAINER_NAME}.entrypoints=websecure' \
                          --label 'traefik.http.routers.${DOCKER_CONTAINER_NAME}.tls=true' \
                          --label 'traefik.http.routers.${DOCKER_CONTAINER_NAME}.tls.certresolver=letsencrypt' \
                          --label 'traefik.http.services.${DOCKER_CONTAINER_NAME}.loadbalancer.server.port=${CONFIG_PORT}' \
                          --volume ${DOCKER_CONTAINER_NAME}:/var/www/html \
                          --network reverse-proxy-net \
                          --network-alias ${DOCKER_CONTAINER_NAME} \
                          --network database-antares-net \
                          --network-alias ${DOCKER_CONTAINER_NAME} \
                          -e DATABASE_URL='${CONFIG_DATABASE_URL}' \
                          -e PORT='${CONFIG_PORT}' \
                          -e EMAIL_PASS='${CONFIG_EMAIL_PASS}' \
                          ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}
                        """

                        waitForAppStartup("${CONFIG_PUBLIC_URL}/ui", 10, 5)
                    } catch (Exception e) {
                        dsContent = e.message
                        error "Error en New Container: ${e.message}"
                    }
                }
            }
        }
        
        stage('Healthcheck') {
            steps {
                script {
                    try {
                        def response = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://${CONFIG_PUBLIC_URL}/ui', returnStdout: true).trim()
                        if (response != "200") {
                            error "Healthcheck failed. Status: ${response}"
                        } else {
                            echo "Healthcheck successful. Status: 200 OK."
                        }
                        dsContent = response
                    } catch (Exception e) {
                        dsContent = e.message
                        error "Healthcheck failed. Error: ${e.message}"
                    }
                }
            }
        }
    }    
    
    post {
        success {
            script {
                def buildNumber = env.BUILD_NUMBER
                def message = [
                    embeds: [[
                        title: ":white_check_mark: Jenkins Pipeline Success! #${buildNumber}",
                        description: "Build # ${buildNumber}",
                        color: 5763719,
                        fields: [
                            [name: "Project", value: DOCKER_IMAGE, inline: true],
                            [name: "Branch", value: REPO_BRANCH, inline: true],
                            [name: "Environment", value: "QA", inline: true],
                            [name: "URL", value: "https://${CONFIG_PUBLIC_URL}/ui", inline: true]
                        ]
                    ]]
                ]
                sendDiscordNotification(message)
            }
        }
        failure {
            script {
                def buildNumber = env.BUILD_NUMBER
                def message = [
                    embeds: [[
                        title: ":x: Jenkins Pipeline Failed!",
                        description: "Build # ${buildNumber}",
                        color: 15105570,
                        fields: [
                            [name: "Project", value: DOCKER_IMAGE, inline: true],
                            [name: "Branch", value: REPO_BRANCH, inline: true],
                            [name: "Environment", value: "QA", inline: true],
                            [name: "Error", value: dsContent, inline: false]
                        ]
                    ]]
                ]
                sendDiscordNotification(message)
            }
        }
    }
}