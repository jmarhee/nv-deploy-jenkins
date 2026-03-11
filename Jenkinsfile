pipeline {
    agent any 

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
                    // Get list of changed files (i.e. env file, values, or new cluster dir) compared to previous commit
                    def changedFiles = sh(
                        script: "git diff --name-only HEAD~1 HEAD || echo ''",
                        returnStdout: true
                    ).trim()

                    def changedClusters = []
                    if (changedFiles) {
                        changedFiles.split('\n').each { file ->
                            // Matches any change within the clusters/{cluster-name}/ directory
                            def match = file =~ /^clusters\/([^\/]+)\//
                            if (match) {
                                def clusterName = match[0][1]
                                if (!changedClusters.contains(clusterName)) {
                                    changedClusters.add(clusterName)
                                }
                            }
                        }
                    }

                    if (changedClusters.isEmpty()) {
                        echo "No changes detected in cluster directories. Skipping deployment."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    env.CHANGED_CLUSTERS = changedClusters.join(',')
                    echo "Clusters to process: ${env.CHANGED_CLUSTERS}"
                }
            }
        }

        stage('Verify Prerequisites') {
            when {
                expression { env.CHANGED_CLUSTERS?.trim() }
            }
            steps {
                script {
                    if (sh(script: 'which helm', returnStatus: true) != 0) error "helm not found"
                    if (sh(script: 'which kubectl', returnStatus: true) != 0) error "kubectl not found"

                    sh 'helm repo add neuvector ${HELM_REPO} --force-update > /dev/null 2>&1 || true'
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
            echo "Pipeline failed. Please check the logs for specific cluster errors."
        }
    }
}

def deployCluster(String clusterName) {
    echo "--- Starting Deployment: ${clusterName} ---"

    def clusterDir = "clusters/${clusterName}"
    def valuesFile = "${clusterDir}/values.yaml"
    def envFile = "${clusterDir}/env"

    if (!fileExists(envFile)) {
        echo "Environment file missing for ${clusterName}. Skipping."
        return
    }

    def envContent = readFile(envFile).trim()
    def config = parseEnvFile(envContent)

    def debugMode = config.DEBUG?.toLowerCase() == 'true'
    def dryRun    = config.DRY_RUN?.toLowerCase() == 'true'
    
    def kubeconfigCredential = config.KUBECONFIG_CREDENTIAL ?: "${clusterName}-kubeconfig"
    def namespace = config.NAMESPACE ?: 'neuvector'
    def releaseName = config.RELEASE_NAME ?: 'neuvector'
    def chartVersion = config.HELM_CHART_VERSION ?: 'latest'

    withCredentials([file(credentialsId: kubeconfigCredential, variable: 'KUBECONFIG')]) {
        try {
            sh 'kubectl cluster-info --request-timeout=5s > /dev/null'

            def dryRunFlag = dryRun ? '--dry-run' : ''
            def versionFlag = (chartVersion == 'latest') ? '' : "--version ${chartVersion}"

            echo "Running: helm upgrade --install ${releaseName} ..."
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
                echo "Deployment for ${clusterName} successful."
            } else {
                echo "Dry run for ${clusterName} completed."
            }

        } catch (Exception e) {
            echo "Deployment Error in ${clusterName}: ${e.message} !!!"
            
            if (debugMode && !dryRun) {
                echo "DEBUG is true: Cleaning up failed release ${releaseName}..."
                sh "helm uninstall ${releaseName} -n ${namespace} --wait || true"
            } else if (dryRun) {
                echo "Dry run failed. No cleanup necessary."
            } else {
                echo "DEBUG is false: Release left in failed state for debugging."
            }
            
            throw e
        }
    }
}

def parseEnvFile(String content) {
    def config = [:]
    content.split('\n').each { line ->
        line = line.trim()
        if (line && !line.startsWith('#')) {
            def parts = line.split('=', 2)
            if (parts.length == 2) {
                def key = parts[0].trim()
                def value = parts[1].trim()
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
