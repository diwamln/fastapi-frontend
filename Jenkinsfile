pipeline {
    agent any

    environment {
        // --- KONFIGURASI DOCKER ---
        DOCKER_IMAGE = 'diwamln/fastapi-frontend' 
        DOCKER_CREDS = 'docker-hub' 
        
        // --- KONFIGURASI GIT (REPO MANIFEST) ---
        GIT_CREDS    = 'git-token'
        
        // URL Repo Manifest
        MANIFEST_REPO_URL = 'github.com/diwamln/intern-devops-manifests' 
        
        // --- URL BACKEND ---
        TEST_API_URL = 'http://192.168.89.127:30093' 
        PROD_API_URL = 'http://192.168.89.127:30094' 
        
        // --- PATH MANIFEST (Supaya Rapi & Tidak Salah Path) ---
        // Sesuaikan dengan struktur foldermu
        MANIFEST_TEST_PATH = 'fastapi-frontend/dev/deployment.yaml'
        MANIFEST_PROD_PATH = 'fastapi-frontend/prod/deployment.yaml' // <--- Pastikan folder prod sudah ada!
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
                    sh 'rm -rf temp_manifests'
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            // 1. Clone
                            sh "git clone https://${GIT_USER}:${GIT_PASS}@${MANIFEST_REPO_URL} ."
                            sh 'git config user.email "jenkins@bot.com"'
                            sh 'git config user.name "Jenkins Pipeline"'
                            
                            // 2. Update YAML (Pakai variabel path agar aman)
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-test|g' ${MANIFEST_TEST_PATH}"
                            
                            // 3. Push dengan Pengecekan (FIX ERROR EXIT CODE 1)
                            sh """
                                git add .
                                # Cek apakah ada perubahan di index git?
                                if ! git diff-index --quiet HEAD; then
                                    git commit -m 'Deploy TEST: ${env.BASE_TAG}-test [skip ci]'
                                    git push origin main
                                else
                                    echo "Tidak ada perubahan pada manifest TEST, skip commit."
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
                    dir('frontend') { // Pastikan Dockerfile ada di folder 'frontend'
                        docker.withRegistry('', DOCKER_CREDS) {
                            def prodTag = "${env.BASE_TAG}-prod"
                            echo "Building PROD Image connect to: ${env.PROD_API_URL}"
                            
                            // Build ULANG dengan URL PROD + No Cache
                            def prodImage = docker.build("${DOCKER_IMAGE}:${prodTag}", "--no-cache --build-arg VITE_API_URL=${env.PROD_API_URL} .")
                            
                            prodImage.push()
                            prodImage.push('latest')
                        }
                    }
                }
            }
        }

        stage('Update Manifest (PROD)') {
            steps {
                script {
                    dir('temp_manifests') {
                        withCredentials([usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            
                            sh "git pull origin main"
                            
                            // Update file PROD (Perhatikan variabel PATH yg dipakai)
                            sh "sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${env.BASE_TAG}-prod|g' ${MANIFEST_PROD_PATH}"
                            
                            // Push dengan Pengecekan
                            sh """
                                git add .
                                if ! git diff-index --quiet HEAD; then
                                    git commit -m 'Promote PROD: ${env.BASE_TAG}-prod [skip ci]'
                                    git push origin main
                                else
                                    echo "Tidak ada perubahan pada manifest PROD, skip commit."
                                fi
                            """
                        }
                    }
                }
            }
        }
    }
}
