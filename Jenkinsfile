pipeline {
    agent any

    environment {
        // --- KONFIGURASI DOCKER & GIT ---
        DOCKER_IMAGE = 'diwamln/fastapi-frontend' 
        DOCKER_CREDS = 'dockerhub-credentials-id' 
        GIT_CREDS    = 'github-credentials-id'    
        REPO_URL     = 'github.com/USERNAME/NAMA_REPO.git' // Ganti USERNAME & REPO

        // --- KONFIGURASI URL BACKEND (KUNCI PERBEDAANNYA DISINI) ---
        
        // 1. URL Backend untuk Environment TEST
        // (Misal: IP Kubernetes Node + NodePort Service Backend Test)
        TEST_API_URL = 'http://192.168.1.50:30001' 

        // 2. URL Backend untuk Environment PRODUCTION
        // (Misal: Domain asli atau IP Load Balancer)
        PROD_API_URL = 'http://api.domain-anda.com' 
    }

    stages {
        stage('Checkout & Versioning') {
            steps {
                git branch: 'main', url: "https://${REPO_URL}", credentialsId: GIT_CREDS
                script {
                    def commitHash = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    // Base Tag, nanti akan ditambah -test atau -prod
                    env.BASE_TAG = "build-${BUILD_NUMBER}-${commitHash}" 
                }
            }
        }

        // =========================================
        // PROSES UNTUK ENVIRONMENT TESTING
        // =========================================
        stage('Build & Push (TEST Image)') {
            steps {
                script {
                    dir('frontend') {
                        docker.withRegistry('', DOCKER_CREDS) {
                            // Tag khusus test: ...-test
                            def testTag = "${env.BASE_TAG}-test"
                            
                            echo "Building TEST Image connect to: ${env.TEST_API_URL}"
                            
                            // Build dengan URL TEST
                            def testImage = docker.build("${DOCKER_IMAGE}:${testTag}", "--build-arg VITE_API_URL=${env.TEST_API_URL} .")
                            
                            testImage.push()
                        }
                    }
                }
            }
        }

        stage('Deploy to TEST (Update YAML)') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh """
                            git config user.email "jenkins@bot.com"
                            git config user.name "Jenkins Pipeline"
                            git pull origin main
                            
                            # Update file test.yaml dengan image tag berakhiran -test
                            sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-test|g' frontend/k8s/test.yaml
                            
                            git add frontend/k8s/test.yaml
                            git commit -m "Deploy TEST: ${env.BASE_TAG}-test [skip ci]"
                            git push https://${GIT_USER}:${GIT_PASS}@${REPO_URL} main
                        """
                    }
                }
            }
        }

        // =========================================
        // APPROVAL GATE
        // =========================================
        stage('Approval for Production') {
            steps {
                input message: "Aplikasi versi TEST (${env.BASE_TAG}-test) sudah dideploy. Lanjut build untuk PRODUCTION?", ok: "Gas ke Prod!"
            }
        }

        // =========================================
        // PROSES UNTUK ENVIRONMENT PRODUCTION
        // =========================================
        stage('Build & Push (PROD Image)') {
            steps {
                script {
                    dir('frontend') {
                        docker.withRegistry('', DOCKER_CREDS) {
                            // Tag khusus prod: ...-prod
                            def prodTag = "${env.BASE_TAG}-prod"
                            
                            echo "Building PROD Image connect to: ${env.PROD_API_URL}"
                            
                            // Build ULANG dengan URL PROD
                            // Kita pakai --no-cache agar benar-benar bersih
                            def prodImage = docker.build("${DOCKER_IMAGE}:${prodTag}", "--no-cache --build-arg VITE_API_URL=${env.PROD_API_URL} .")
                            
                            prodImage.push()
                            
                            // Tag 'latest' diarahkan ke versi Prod
                            prodImage.push('latest')
                        }
                    }
                }
            }
        }

        stage('Deploy to PRODUCTION (Update YAML)') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh """
                            git pull origin main
                            
                            # Update file prod.yaml dengan image tag berakhiran -prod
                            sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-prod|g' frontend/k8s/prod.yaml
                            
                            git add frontend/k8s/prod.yaml
                            git commit -m "Promote PROD: ${env.BASE_TAG}-prod [skip ci]"
                            git push https://${GIT_USER}:${GIT_PASS}@${REPO_URL} main
                        """
                    }
                }
            }
        }
    }
}
