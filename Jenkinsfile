pipeline {
    agent {
        kubernetes {
            // Kita definisikan spec Pod langsung di sini (Infrastructure as Code)
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:24.0.6-dind
    securityContext:
      privileged: true
    volumeMounts:
    - name: dind-storage
      mountPath: /var/lib/docker
  - name: jnlp
    image: jenkins/inbound-agent:latest
  volumes:
  - name: dind-storage
    emptyDir: {}
'''
        }
    }

    environment {
        // --- KONFIGURASI DOCKER ---
        DOCKER_IMAGE = 'diwamln/fastapi-frontend' 
        DOCKER_CREDS = 'docker-hub' 
        
        // --- KONFIGURASI GIT (REPO MANIFEST) ---
        GIT_CREDS    = 'git-token'
        
        // URL Repo Manifest
        MANIFEST_REPO_URL = 'github.com/diwamln/intern-devops-manifests.git' 
        
        // --- URL BACKEND (Akan dibakar ke dalam Image Frontend) ---
        TEST_API_URL = 'http://k8s-01.naratel.net.id:30093' 
        PROD_API_URL = 'http://k8s-01.naratel.net.id:30094' 
        
        // --- PATH FILE MANIFEST ---
        MANIFEST_TEST_PATH = 'fastapi-frontend/dev/deployment.yaml'
        MANIFEST_PROD_PATH = 'fastapi-frontend/prod/deployment.yaml' 
    }

    stages {
        stage('Checkout & Versioning') {
            steps {
                checkout scm
                script {
                    def commitHash = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    env.BASE_TAG = "build-${BUILD_NUMBER}-${commitHash}" 
                    currentBuild.displayName = "#${BUILD_NUMBER} (${env.BASE_TAG})"
                }
            }
        }

        // =========================================
        // FLOW: ENVIRONMENT TESTING
        // =========================================
        stage('Build & Push (TEST Image)') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_CREDS) {
                        def testTag = "${env.BASE_TAG}-test"
                        echo "Building TEST Image connect to: ${env.TEST_API_URL}"
                        
                        // Build image
                        def testImage = docker.build("${DOCKER_IMAGE}:${testTag}", "--build-arg VITE_API_URL=${env.TEST_API_URL} .")
                        testImage.push()
                    }
                }
            }
        }

        stage('Update Manifest (TEST)') {
            steps {
                script {
                    sh 'rm -rf temp_manifests'
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            sh "git clone https://${GIT_USER}:${GIT_PASS}@${MANIFEST_REPO_URL} ."
                            sh 'git config user.email "jenkins@bot.com"'
                            sh 'git config user.name "Jenkins Pipeline"'
                            
                            // === THE FIX ===
                            // Menggunakan regex (docker.io/)? agar cocok dengan YAML kamu yang ada tulisan docker.io-nya
                            sh "sed -i -E 's|image: (docker.io/)?${DOCKER_IMAGE}:.*|image: docker.io/${DOCKER_IMAGE}:${env.BASE_TAG}-test|g' ${MANIFEST_TEST_PATH}"
                            
                            // DEBUG: Print hasil perubahan ke console log biar kamu yakin
                            echo "--- HASIL UPDATE YAML TEST ---"
                            sh "cat ${MANIFEST_TEST_PATH} | grep image:"
                            echo "------------------------------"
                            
                            // Push dengan Safety Check (Biar ga error exit code 1)
                            sh """
                                git add .
                                if ! git diff-index --quiet HEAD; then
                                    git commit -m 'Deploy TEST: ${env.BASE_TAG}-test [skip ci]'
                                    git push origin main
                                else
                                    echo "WARNING: Tidak ada perubahan di file YAML. Cek regex sed!"
                                fi
                            """
                        }
                    }
                }
            }
        }

        // =========================================
        // GATE: APPROVAL MANUAL
        // =========================================
        stage('Approval for Production') {
            steps {
                input message: "Versi TEST (${env.BASE_TAG}-test) sukses dideploy. Lanjut ke PROD?", ok: "Gas ke Prod!"
            }
        }

        // =========================================
        // FLOW: ENVIRONMENT PRODUCTION
        // =========================================
        stage('Build & Push (PROD Image)') {
            steps {
                script {
                    // Konsisten: Build di Root, jangan masuk folder frontend
                    docker.withRegistry('', DOCKER_CREDS) {
                        def prodTag = "${env.BASE_TAG}-prod"
                        echo "Building PROD Image connect to: ${env.PROD_API_URL}"
                        
                        def prodImage = docker.build("${DOCKER_IMAGE}:${prodTag}", "--no-cache --build-arg VITE_API_URL=${env.PROD_API_URL} .")
                        
                        prodImage.push()
                        prodImage.push('latest') // Optional: Update latest tag di docker hub
                    }
                }
            }
        }

        stage('Update Manifest (PROD)') {
            steps {
                script {
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            sh "git pull origin main" // Tarik update terbaru
                            
                            // === THE FIX JUGA DI SINI ===
                            sh "sed -i -E 's|image: (docker.io/)?${DOCKER_IMAGE}:.*|image: docker.io/${DOCKER_IMAGE}:${env.BASE_TAG}-prod|g' ${MANIFEST_PROD_PATH}"
                            
                            sh """
                                git add .
                                if ! git diff-index --quiet HEAD; then
                                    git commit -m 'Promote PROD: ${env.BASE_TAG}-prod [skip ci]'
                                    git push origin main
                                else
                                    echo "WARNING: Tidak ada perubahan di file YAML PROD."
                                fi
                            """
                        }
                    }
                }
            }
        }
    }
}
