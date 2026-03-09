node {
    // 1. FORCE AUTH PLUGIN USAGE
    env.USE_GKE_GCLOUD_AUTH_PLUGIN = 'true'

    // 2. Define Tools and Variables
    def mvnHome = tool name: 'maven', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn "
    
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

        withCredentials([file(credentialsId: 'gcp', variable: 'GC_KEY')]) {
            
            stage('Build and Push Image') {
                sh '''
                    gcloud auth activate-service-account --key-file=$GC_KEY
                    gcloud auth configure-docker ''' + registryUrl + ''' --quiet
                '''
                sh "${mvnCMD} clean compile jib:build -Djib.to.image=${fullRepoPath}"
            }

            stage('Deploy to GKE') {
                sh '''
                    # 1. Authenticate
                    gcloud auth activate-service-account --key-file=$GC_KEY
                    
                    # 2. Get cluster credentials
                    gcloud container clusters get-credentials ''' + clusterName + ''' \
                        --zone ''' + zone + ''' \
                        --project ''' + project + '''
                    
                    # 3. Update the YAML
                    sed -i "s|IMAGE_URL|''' + fullRepoPath + '''|g" k8s/deployment.yaml
                    
                    # 4. Apply
                    kubectl apply -f k8s/deployment.yaml
                '''
            }
        }
        
    } catch (exc) {
        echo "Pipeline failed: ${exc.message}"
        throw exc
    }
}