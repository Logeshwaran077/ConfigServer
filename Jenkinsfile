node {
    // 1. Define Tools and Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
    // Using environment variables or defaults
    def project = "cyber-security-48"
    def repo = "spring-microservices"
    def registryUrl = "us-west4-docker.pkg.dev"
    def fullRepoPath = "${registryUrl}/${project}/${repo}/config-server"
    def clusterName = "microservices-cluster-1"
    def zone = "us-west4"

    try {
        stage('Cleanup') {
            cleanWs() 
        }

        stage('Checkout') {
            checkout([
                $class: 'GitSCM', 
                branches: [[name: '*/main']], 
                userRemoteConfigs: [[
                    url: 'https://github.com/Logeshwaran077/ConfigServer.git',
                    credentialsId: 'git' 
                ]]
            ])
        }

        // Use the 'gcp' secret file for the entire lifecycle
        withCredentials([file(credentialsId: 'gcp', variable: 'GC_KEY')]) {
            
            stage('Build and Push Image') {
                // Using single quotes for sh to avoid Groovy interpolation warnings
                sh '''
                    gcloud auth activate-service-account --key-file=$GC_KEY
                    gcloud auth configure-docker ''' + registryUrl + ''' --quiet
                '''
                
                // Build and push using Jib
                sh "${mvnCMD} clean compile jib:build -Djib.to.image=${fullRepoPath}"
            }

            stage('Deploy to GKE') {
                sh '''
                    # 1. Authenticate with Service Account
                    gcloud auth activate-service-account --key-file=$GC_KEY
                    
                    # 2. Configure kubectl for GKE (Uses the gke-gcloud-auth-plugin automatically)
                    gcloud container clusters get-credentials ''' + clusterName + ''' \
                        --zone ''' + zone + ''' \
                        --project ''' + project + '''
                    
                    # 3. Update the image placeholder in the deployment manifest
                    sed -i "s|IMAGE_URL|''' + fullRepoPath + '''|g" k8s/deployment.yaml
                    
                    # 4. Apply to Cluster
                    kubectl apply -f k8s/deployment.yaml
                '''
            }
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}