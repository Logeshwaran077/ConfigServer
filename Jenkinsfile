node {
    // 1. Define Tools and Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
    // SANITIZATION: Removes accidental spaces and forces lowercase for Docker compatibility
    def project = env.PROJECT_ID.trim().toLowerCase()
    def repo = env.ARTIFACT_REGISTRY.trim().toLowerCase()
    
    def registryUrl = "us-west4-docker.pkg.dev"
    // The final image path in Google Artifact Registry
    def fullRepoPath = "${registryUrl}/${project}/${repo}/config-server"

    try {
        stage('Cleanup') {
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
                // Authenticate with Google Cloud
                sh "gcloud auth activate-service-account --key-file=${GC_KEY}"
                sh "gcloud auth configure-docker ${registryUrl} --quiet"
                
                // We use -Djib.to.image to tell Jib exactly where to push the image
                // The double quotes around the parameter handle any residual shell issues
                sh "${mvnCMD} clean compile jib:build \"-Djib.to.image=${fullRepoPath}\""
            }
        }

        stage('Deploy') {
            // Update the image placeholder in your K8s manifest
            sh "sed -i 's|IMAGE_URL|${fullRepoPath}|g' k8s/deployment.yaml"
            
            step([
                $class: 'KubernetesEngineBuilder',
                projectId: project,
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