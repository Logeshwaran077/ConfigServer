node {
    // 1. Define Tools and Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
    // Note: Ensure PROJECT_ID and ARTIFACT_REGISTRY are set in Jenkins Environment
    def registryUrl = "us-west4-docker.pkg.dev"
    def fullRepoPath = "${registryUrl}/${env.PROJECT_ID}/${env.ARTIFACT_REGISTRY}"

    try {
        stage('Cleanup') {
            // Wipes the workspace to ensure a fresh clone
            cleanWs() 
        }

        stage('Checkout') {
            checkout([
                $class: 'GitSCM', 
                branches: [[name: '*/main']], 
                doGenerateSubmoduleConfigurations: false, 
                extensions: [], 
                userRemoteConfigs: [[
                    url: 'https://github.com/Logeshwaran077/ConfigServer.git',
                    credentialsId: 'git' 
                ]]
            ])
        }

        stage('Build and Push Image') {
            withCredentials([file(credentialsId: 'gcp', variable: 'GC_KEY')]) {
                // Authenticate and configure Docker for GCP
                sh "gcloud auth activate-service-account --key-file=${GC_KEY}"
                sh "gcloud auth configure-docker ${registryUrl}"
                
                // FIXED: Removed the trailing 'spring-microservices' text
                // This command now correctly ends after the DREPO_URL parameter
                sh "${mvnCMD} clean install jib:build \"-DREPO_URL=${fullRepoPath}\""
            }
        }

        stage('Deploy') {
            // Update the image placeholder in your K8s manifest
            sh "sed -i 's|IMAGE_URL|${fullRepoPath}|g' k8s/deployment.yaml"
            
            step([
                $class: 'KubernetesEngineBuilder',
                projectId: env.PROJECT_ID,
                clusterName: env.CLUSTER,
                location: env.ZONE,
                manifestPattern: 'k8s/deployment.yaml',
                credentialsId: 'gcp', 
                verifyDeployments: true
            ])
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}