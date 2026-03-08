node {
    // 1. Define Tools and Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
    // Constructing the registry path
    def registryUrl = "us-west4-docker.pkg.dev"
    def fullRepoPath = "${registryUrl}/${env.PROJECT_ID}/${env.ARTIFACT_REGISTRY}"

    try {
        stage('Cleanup') {
            // Wipes the workspace to prevent "Revision not found" errors from old metadata
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
                
                // Build with Jib and push to Artifact Registry
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
                credentialsId: 'gcp', 
                verifyDeployments: true
            ])
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}