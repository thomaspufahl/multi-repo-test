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

        DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/1333894803850657924/bB4JTfVlFw3BKdclv4KqsuPrQBWomjg_CWOkXDYHCwOpb_vgqOS3H8VLxnuN5bWUTj72"
    }

    stages {        
        stage('Healthcheck') {
            steps {
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
        }
    }    
}
