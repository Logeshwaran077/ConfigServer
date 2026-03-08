node {
    // 1. Define Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
    // Note: Ensure these environment variables are defined in your Jenkins Job 
    // or globally in Manage Jenkins > Configure System
    def registryUrl = "us-west4-docker.pkg.dev"
    def fullRepoPath = "${registryUrl}/${PROJECT_ID}/${ARTIFACT_REGISTRY}"

    try {
        stage('Cleanup') {
            cleanWs() // Deletes the workspace before starting to avoid "Revision" errors
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
                // Authenticate with Google Cloud
                sh "gcloud auth activate-service-account --key-file=${GC_KEY}"
                sh "gcloud auth configure-docker ${registryUrl}"
                
                // Build with Jib and push to Artifact Registry
                // Fixed: Changed PROJECTID to PROJECT_ID to match your standard
                sh "${mvnCMD} clean install jib:build -DREPO_URL=${fullRepoPath}"
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
                credentialsId: 'gcp', // Usually matches the GCP JSON key ID
                verifyDeployments: true
            ])
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}