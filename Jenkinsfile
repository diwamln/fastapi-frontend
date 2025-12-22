pipeline {
    agent any

    environment {
        // --- KONFIGURASI DOCKER ---
        DOCKER_IMAGE = 'diwamln/fastapi-frontend' 
        DOCKER_CREDS = 'docker-hub' 
        
        // --- KONFIGURASI GIT (REPO MANIFEST) ---
        GIT_CREDS    = 'git-token'
        
        // PENTING: Ini URL ke Repo Manifest (bukan repo aplikasi ini)
        // Format: github.com/USERNAME/NAMA_REPO_MANIFESTS.git
        MANIFEST_REPO_URL = 'github.com/diwamln/intern-devops-manifests' 
        
        // --- KONFIGURASI URL BACKEND (Sesuaikan dengan IP/Domain Asli) ---
        // URL ini akan "dibakar" permanen ke dalam aplikasi React saat build
        
        // 1. Backend URL untuk Environment TEST
        TEST_API_URL = 'http://192.168.89.127:30093' 

        // 2. Backend URL untuk Environment PRODUCTION
        PROD_API_URL = 'http://192.168.89.127:30094' 
    }

    stages {
        stage('Checkout & Versioning') {
            steps {
                checkout scm
                script {
                    // Membuat Tag Unik: build-NOMOR-HASH
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
                            
                            // Build image dengan menyuntikkan URL TEST
                            def testImage = docker.build("${DOCKER_IMAGE}:${testTag}", "--build-arg VITE_API_URL=${env.TEST_API_URL} .")
                            testImage.push()
                        }
                }
            }
        }

        stage('Update Manifest (TEST)') {
            steps {
                script {
                    // Clone Repo Manifests ke folder sementara
                    sh 'rm -rf temp_manifests' // Bersihkan sisa build sebelumnya
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            // 1. Clone Repo Manifests
                            sh "git clone https://${GIT_USER}:${GIT_PASS}@${MANIFEST_REPO_URL} ."
                            sh 'git config user.email "jenkins@bot.com"'
                            sh 'git config user.name "Jenkins Pipeline"'
                            
                            // 2. Update file test.yaml (Sesuaikan path folder di repo manifest Anda)
                            // Misal strukturnya: frontend/test.yaml
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-test|g' frontend/k8s/test.yaml"
                            
                            // 3. Push Perubahan
                            sh "git add ."
                            sh "git commit -m 'Deploy TEST: ${env.BASE_TAG}-test [skip ci]'"
                            sh "git push origin main"
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
                    dir('frontend') {
                        docker.withRegistry('', DOCKER_CREDS) {
                            def prodTag = "${env.BASE_TAG}-prod"
                            echo "Building PROD Image connect to: ${env.PROD_API_URL}"
                            
                            // Build ULANG dengan URL PROD + No Cache (Wajib!)
                            def prodImage = docker.build("${DOCKER_IMAGE}:${prodTag}", "--no-cache --build-arg VITE_API_URL=${env.PROD_API_URL} .")
                            
                            prodImage.push()
                            prodImage.push('latest') // Update tag latest ke versi prod
                        }
                    }
                }
            }
        }

        stage('Update Manifest (PROD)') {
            steps {
                script {
                    // Masuk lagi ke folder manifest (pull terbaru dulu untuk hindari konflik)
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            sh "git pull origin main"
                            
                            // Update file prod.yaml (Sesuaikan path folder di repo manifest Anda)
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-prod|g' frontend/k8s/prod.yaml"
                            
                            sh "git add ."
                            sh "git commit -m 'Promote PROD: ${env.BASE_TAG}-prod [skip ci]'"
                            sh "git push origin main"
                        }
                    }
                }
            }
        }
    }
}
