node {
    // 1. Define Tools and Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
    // SANITIZATION: Removes accidental spaces and forces lowercase
    def project = env.PROJECT_ID.trim().toLowerCase()
    def repo = env.ARTIFACT_REGISTRY.trim().toLowerCase()
    
    def registryUrl = "us-west4-docker.pkg.dev"
    def fullRepoPath = "${registryUrl}/${project}/${repo}/config-server"

    try {
        stage('Cleanup') {
            cleanWs() 
        }

        stage('Checkout') {
            checkout([
                $class: 'GitSCM', 
                branches: [[name: '*/main']], 
                extensions: [], 
                userRemoteConfigs: [[
                    url: 'https://github.com/Logeshwaran077/ConfigServer.git',
                    credentialsId: 'git' 
                ]]
            ])
        }

        // We wrap both Build and Deploy in the same credential block
        withCredentials([file(credentialsId: 'gcp', variable: 'GC_KEY')]) {
            
            stage('Build and Push Image') {
                // Authenticate gcloud for the Jib push
                sh "gcloud auth activate-service-account --key-file=${GC_KEY}"
                sh "gcloud auth configure-docker ${registryUrl} --quiet"
                
                sh "${mvnCMD} clean compile jib:build \"-Djib.to.image=${fullRepoPath}\""
            }

            stage('Deploy') {
                // Update the image placeholder in your K8s manifest
                sh "sed -i 's|IMAGE_URL|${fullRepoPath}|g' k8s/deployment.yaml"
                
                // Use the GKE Builder with the validated 'gcp' credential ID
                step([
                    $class: 'KubernetesEngineBuilder',
                    projectId: project,
                    clusterName: env.CLUSTER.trim(),
                    location: env.ZONE.trim(),
                    manifestPattern: 'k8s/deployment.yaml',
                    credentialsId: 'gcp', 
                    verifyDeployments: true
                ])
            }
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}