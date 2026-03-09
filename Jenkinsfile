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
                
                // 2. Re-authenticate to ensure session is active
                sh "gcloud auth activate-service-account --key-file=${GC_KEY}"

                // 3. STATIC TOKEN WORKAROUND
                // We manually fetch the token and endpoint to bypass the missing auth plugin
                def token = sh(script: "gcloud auth print-access-token", returnStdout: true).trim()
                def clusterEndpoint = sh(script: "gcloud container clusters describe ${env.CLUSTER.trim()} --zone ${env.ZONE.trim()} --project ${project} --format='get(endpoint)'", returnStdout: true).trim()

                // 4. Apply using the token and endpoint directly
                // --insecure-skip-tls-verify is used because we are connecting via the raw IP endpoint
                sh "kubectl --token=${token} --server=https://${clusterEndpoint} --insecure-skip-tls-verify apply -f k8s/deployment.yaml"
                
                echo "Deployment successful."
            }
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}