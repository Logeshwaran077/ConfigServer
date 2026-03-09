node {
    // 1. Define Tools and Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
    // SANITIZATION: Removes accidental spaces and forces lowercase for Docker rules
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

        // Use the 'gcp' secret file for both Build and Deploy stages
        withCredentials([file(credentialsId: 'gcp', variable: 'GC_KEY')]) {
            
            stage('Build and Push Image') {
                sh "gcloud auth activate-service-account --key-file=${GC_KEY}"
                sh "gcloud auth configure-docker ${registryUrl} --quiet"
                
                // Build and push using Jib
                sh "${mvnCMD} clean compile jib:build \"-Djib.to.image=${fullRepoPath}\""
            }

            stage('Deploy') {
                echo "Deploying to GKE cluster: ${env.CLUSTER}"
                
                // 1. Update the image placeholder in your K8s manifest
                sh "sed -i 's|IMAGE_URL|${fullRepoPath}|g' k8s/deployment.yaml"
                
                // 2. Bypass the gke-gcloud-auth-plugin requirement using an environment variable
                withEnv(['USE_GKE_GCLOUD_AUTH_PLUGIN=False']) {
                    
                    // Configure kubectl to point to your specific GKE cluster
                    sh "gcloud container clusters get-credentials ${env.CLUSTER.trim()} --zone ${env.ZONE.trim()} --project ${project}"

                    // 3. Apply the manifest with --validate=false to prevent the plugin-check crash
                    sh "kubectl apply -f k8s/deployment.yaml --validate=false"
                    
                    // 4. Verify the rollout status
                    sh "kubectl rollout status deployment/config-server"
                }
            }
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}