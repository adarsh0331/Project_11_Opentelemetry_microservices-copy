// Requires on the Jenkins agent: git, docker (jenkins user in the docker group).
// Sonar-scanner and Trivy are run via their official Docker images, so nothing else
// needs to be installed on the agent.
//
// Jenkins credentials required:
//   docker-cred  (Username with password) - Docker Hub push access
//   github-cred  (Username with password) - GitHub username + PAT with repo write access,
//                                            used to push the deployment-service.yml update back to main
//   sonar-token  (Secret text)            - SonarQube auth token (leave any placeholder value
//                                            until a real SonarQube server is available; the
//                                            scan stage never fails the build either way)
//
// Jenkins parameter/global env expected:
//   SONAR_HOST_URL - e.g. http://your-sonarqube-host:9000 (set as a global env var in
//                     Manage Jenkins > System, or override at Build with Parameters)

def SERVICES = [
    accounting       : [context: 'src/accounting',        dockerfile: 'src/accounting/Dockerfile',        imageSuffix: 'accountingservice'],
    ad               : [context: 'src/ad',                dockerfile: 'src/ad/Dockerfile',                imageSuffix: 'adservice'],
    cart             : [context: 'src/cart/src',               dockerfile: 'src/cart/src/Dockerfile',          imageSuffix: 'cartservice'],
    checkout         : [context: 'src/checkout',           dockerfile: 'src/checkout/Dockerfile',          imageSuffix: 'checkoutservice'],
    currency         : [context: 'src/currency',           dockerfile: 'src/currency/Dockerfile',          imageSuffix: 'currencyservice'],
    email            : [context: 'src/email',               dockerfile: 'src/email/Dockerfile',             imageSuffix: 'emailservice'],
    'flagd-ui'       : [context: 'src/flagd-ui',            dockerfile: 'src/flagd-ui/Dockerfile',          imageSuffix: 'flagdui'],
    'fraud-detection': [context: 'src/fraud-detection',     dockerfile: 'src/fraud-detection/Dockerfile',   imageSuffix: 'frauddetectionservice'],
    frontend         : [context: 'src/frontend',            dockerfile: 'src/frontend/Dockerfile',          imageSuffix: 'frontend'],
    'frontend-proxy' : [context: 'src/frontend-proxy',      dockerfile: 'src/frontend-proxy/Dockerfile',    imageSuffix: 'frontendproxy'],
    'image-provider' : [context: 'src/image-provider',      dockerfile: 'src/image-provider/Dockerfile',    imageSuffix: 'imageprovider'],
    'load-generator' : [context: 'src/load-generator',      dockerfile: 'src/load-generator/Dockerfile',    imageSuffix: 'loadgenerator'],
    payment          : [context: 'src/payment',              dockerfile: 'src/payment/Dockerfile',           imageSuffix: 'paymentservice'],
    'product-catalog': [context: 'src/product-catalog',     dockerfile: 'src/product-catalog/Dockerfile',   imageSuffix: 'productcatalogservice'],
    quote            : [context: 'src/quote',                dockerfile: 'src/quote/Dockerfile',             imageSuffix: 'quoteservice'],
    recommendation   : [context: 'src/recommendation',      dockerfile: 'src/recommendation/Dockerfile',    imageSuffix: 'recommendationservice'],
    shipping         : [context: 'src/shipping',             dockerfile: 'src/shipping/Dockerfile',          imageSuffix: 'shippingservice'],
]

pipeline {
    agent any

    parameters {
        string(name: 'SERVICES_OVERRIDE', defaultValue: '', description: 'Comma-separated service names to force-build (leave empty to auto-detect changed services)')
    }

    environment {
        SONAR_TOKEN     = credentials('sonar-token')
        // Falls back to this default if no global SONAR_HOST_URL env var is configured
        // under Manage Jenkins > System.
        SONAR_HOST_URL  = "${env.SONAR_HOST_URL ?: 'http://localhost:9000'}"
        REPO_URL        = 'github.com/adarsh0331/Project_11_Opentelemetry_microservices-copy.git'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changed Services') {
            steps {
                script {
                    if (params.SERVICES_OVERRIDE?.trim()) {
                        env.CHANGED_SERVICES = params.SERVICES_OVERRIDE.trim()
                    } else {
                        def hasPrevCommit = sh(script: 'git rev-parse HEAD~1', returnStatus: true) == 0
                        def changedFiles = hasPrevCommit
                            ? sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim()
                            : sh(script: 'git ls-tree -r --name-only HEAD', returnStdout: true).trim()

                        def changed = changedFiles.split('\n')
                            .findAll { it.startsWith('src/') }
                            .collect { it.split('/')[1] }
                            .unique()
                            .findAll { SERVICES.containsKey(it) }

                        env.CHANGED_SERVICES = changed.join(',')
                    }
                    echo "Building services: ${env.CHANGED_SERVICES ?: '(none)'}"
                }
            }
        }

        stage('Build, Scan & Deploy Services') {
            when {
                expression { env.CHANGED_SERVICES?.trim() }
            }
            steps {
                script {
                    def tag = env.GIT_COMMIT?.take(7) ?: 'latest'

                    env.CHANGED_SERVICES.split(',').each { svc ->
                        def cfg = SERVICES[svc]
                        if (cfg == null) {
                            echo "Skipping unknown service '${svc}'"
                            return
                        }
                        def image = "adarshbarkunta/${svc}"

                        stage("SonarQube: ${svc}") {
                            catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                                sh """
                                    docker run --rm \
                                      -e SONAR_HOST_URL=${SONAR_HOST_URL} \
                                      -e SONAR_TOKEN=${SONAR_TOKEN} \
                                      -v ${WORKSPACE}/${cfg.context}:/usr/src \
                                      sonarsource/sonar-scanner-cli \
                                      -Dsonar.projectKey=otel-demo-${svc} -Dsonar.sources=. -Dsonar.java.binaries=.
                                """
                            }
                        }

                        stage("Build & Push: ${svc}") {
                            sh "docker build -t ${image}:${tag} -f ${cfg.dockerfile} ${cfg.context}"
                            withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                                sh "docker tag ${image}:${tag} ${image}:latest"
                                sh "docker push ${image}:${tag}"
                                sh "docker push ${image}:latest"
                            }
                        }

                        stage("Trivy Scan: ${svc}") {
                            // report-only: never fails the build, per team decision
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL --exit-code 0 ${image}:${tag} || true"
                        }

                        stage("Update Manifest: ${svc}") {
                            // Scoped withCredentials (rather than a top-level `environment` binding)
                            // keeps the GitHub PAT out of Groovy string interpolation entirely -
                            // bash resolves $GH_USER/$GH_TOKEN from its own environment, and Jenkins
                            // masks both in the console log regardless.
                            withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                                sh """
                                    git config user.email 'jenkins-ci@local'
                                    git config user.name 'jenkins-ci'
                                    for i in 1 2 3 4 5; do
                                        git fetch origin main
                                        git checkout main
                                        git reset --hard origin/main
                                        sed -i "s#image: 'ghcr.io/open-telemetry/demo:[^']*-${cfg.imageSuffix}'#image: '${image}:${tag}'#" deployment-service.yml
                                        sed -i "s#image: '${image}:[^']*'#image: '${image}:${tag}'#" deployment-service.yml
                                        if git diff --quiet -- deployment-service.yml; then
                                            echo 'deployment-service.yml already up to date for ${svc}'
                                            break
                                        fi
                                        git add deployment-service.yml
                                        git commit -m 'chore(${svc}): update image to ${tag}'
                                        if git push https://\$GH_USER:\$GH_TOKEN@${REPO_URL} HEAD:main; then
                                            break
                                        fi
                                        sleep \$(( (RANDOM % 5) + 1 ))
                                    done
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker system prune -f || true'
        }
    }
}
