//
// TRIGGER:
//   Changes to clusters/{cluster-name}/values.yaml or clusters/{cluster-name}/env
//   on the release branch will trigger a deployment to that cluster.
//
// CLUSTER DIRECTORY STRUCTURE:
//   clusters/
//   ├── prod-openshift-east/
//   │   ├── values.yaml          # Helm values
//   │   └── env                   # Deployment parameters
//   ├── prod-openshift-west/
//   │   ├── values.yaml
//   │   └── env
//   └── staging-k3s-01/
//       ├── values.yaml
//       └── env
//
// ENV FILE FORMAT:
//   KUBECONFIG_CREDENTIAL=prod-openshift-east-kubeconfig
//   HELM_CHART_VERSION=5.3.0
//   NAMESPACE=neuvector
//   RELEASE_NAME=neuvector
//
// TO DO:
// * Helm Repo Login
// * Add Helm Repo Login step to Jenkinsfile

pipeline {
    agent any // confirm which agent to use for k8s/helm

    // triggers {
    //     // Trigger on changes to the release branch
    //     // Configure your SCM (GitHub, GitLab, etc.) to send webhooks
    //     pollSCM('')
    // }

    environment {
        HELM_REPO = 'https://neuvector.github.io/neuvector-helm/'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changed Clusters') {
            steps {
                script {
                    // Get list of changed files compared to previous commit
                    def changedFiles = sh(
                        script: "git diff --name-only HEAD~1 HEAD || echo ''",
                        returnStdout: true
                    ).trim()



                    // Find unique cluster directories that have changes
                    def changedClusters = []
                    changedFiles.split('\n').each { file ->
                        def match = file =~ /^clusters\/([^\/]+)\/(values\.yaml|env)$/
                        if (match) {
                            def clusterName = match[0][1]
                            if (!changedClusters.contains(clusterName)) {
                                changedClusters.add(clusterName)
                            }
                        }
                    }

                    if (changedClusters.isEmpty()) {
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    env.CHANGED_CLUSTERS = changedClusters.join(',')
                }
            }
        }

        stage('Verify Prerequisites') {
            when {
                expression { env.CHANGED_CLUSTERS?.trim() }
            }
            steps {
                script {
                    def helmCheck = sh(script: 'which helm', returnStatus: true)
                    def kubectlCheck = sh(script: 'which kubectl', returnStatus: true)

                    if (helmCheck != 0) {
                        error "helm is not installed or not in PATH"
                    }
                    if (kubectlCheck != 0) {
                        error "kubectl is not installed or not in PATH"
                    }

                    sh 'helm repo add neuvector ${HELM_REPO} > /dev/null 2>&1 || true'
                    sh 'helm repo update > /dev/null'
                }
            }
        }

        stage('Deploy Clusters') {
            when {
                expression { env.CHANGED_CLUSTERS?.trim() }
            }
            steps {
                script {
                    def clusters = env.CHANGED_CLUSTERS.split(',')

                    clusters.each { clusterName ->
                        deployCluster(clusterName.trim())
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Deployment failed. Check logs for details."
        }
    }
}

def deployCluster(String clusterName) {
    echo "Deploying: ${clusterName}"

    def clusterDir = "clusters/${clusterName}"
    def valuesFile = "${clusterDir}/values.yaml"
    def envFile = "${clusterDir}/env"

    // Validate required files exist
    if (!fileExists(valuesFile)) {
        error "Values file not found: ${valuesFile}"
    }
    if (!fileExists(envFile)) {
        error "Environment file not found: ${envFile}"
    }

    // Read and parse the env file
    def envContent = readFile(envFile).trim()
    def config = parseEnvFile(envContent)

    // Extract configuration with defaults
    def kubeconfigCredential = config.KUBECONFIG_CREDENTIAL ?: "${clusterName}-kubeconfig"
    def namespace = config.NAMESPACE ?: 'neuvector'
    def releaseName = config.RELEASE_NAME ?: 'neuvector'
    def chartVersion = config.HELM_CHART_VERSION ?: 'latest'
    def dryRun = config.DRY_RUN?.toLowerCase() == 'true'

    withCredentials([file(credentialsId: kubeconfigCredential, variable: 'KUBECONFIG')]) {
        sh 'kubectl cluster-info --request-timeout=5s > /dev/null'

        // Build helm command
        def dryRunFlag = dryRun ? '--dry-run' : ''
        def versionFlag = chartVersion == 'latest' ? '' : "--version ${chartVersion}"

        // Deploy NeuVector
        sh """
            helm upgrade --install ${releaseName} neuvector/core \
                --namespace ${namespace} \
                --create-namespace \
                --values ${valuesFile} \
                ${versionFlag} \
                ${dryRunFlag} \
                --wait \
                --timeout 10m
        """

        if (!dryRun) {
            sh """
                kubectl wait --for=condition=ready pod -l app=neuvector-controller-pod -n ${namespace} --timeout=300s
                kubectl wait --for=condition=ready pod -l app=neuvector-manager-pod -n ${namespace} --timeout=300s
            """
        }
    }
}

def parseEnvFile(String content) {
    def config = [:]
    content.split('\n').each { line ->
        line = line.trim()
        // Skip empty lines and comments
        if (line && !line.startsWith('#')) {
            def parts = line.split('=', 2)
            if (parts.length == 2) {
                def key = parts[0].trim()
                def value = parts[1].trim()
                // Remove surrounding quotes if present
                if ((value.startsWith('"') && value.endsWith('"')) ||
                    (value.startsWith("'") && value.endsWith("'"))) {
                    value = value[1..-2]
                }
                config[key] = value
            }
        }
    }
    return config
}
