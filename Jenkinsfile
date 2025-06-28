// Jenkinsfile - Tinh gọn cho Đồ án 2 (Continuous Delivery)

pipeline {
    // Chạy trên bất kỳ agent nào có sẵn
    agent any

    // Định nghĩa các biến môi trường
    environment {
        // ID của credential Docker Hub đã tạo trong Jenkins
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds'
        // Thay 'your-dockerhub-username' bằng username Docker Hub thực tế của bạn
        DOCKERHUB_USERNAME = 'lshigami' 
    }
 
    stages {
        // STAGE 1: PHÁT HIỆN CÁC SERVICE CÓ THAY ĐỔI
        stage('Detect Changes') {
            steps {
                script {
                    echo "Checking for changes in branch: ${env.BRANCH_NAME}"
                    
                    // =================== SỬA LỖI ROBUST HƠN ===================
                    // Fetch tất cả branches từ remote để đảm bảo có đủ thông tin
                    sh 'git fetch origin'
                    
                    // Kiểm tra xem branch main có tồn tại không
                    def mainExists = sh(script: 'git rev-parse --verify origin/main', returnStatus: true) == 0
                    
                    def changedFiles = ''
                    
                    if (mainExists && env.BRANCH_NAME != 'main') {
                        echo "Comparing with origin/main..."
                        // So sánh branch hiện tại với 'origin/main'
                        changedFiles = sh(script: "git diff --name-only origin/main...HEAD", returnStdout: true).trim()
                        
                        // Nếu không có thay đổi, có thể branch này là main hoặc mới tách từ main
                        if (changedFiles.isEmpty()) {
                            echo "No differences with main branch, comparing with previous commit..."
                            changedFiles = sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim()
                        }
                    } else {
                        echo "Main branch not found or current branch is main, comparing with previous commit..."
                        // Fallback: so sánh với commit trước đó
                        changedFiles = sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim()
                    }
                    // ============================================================

                    echo "Changed files:\n${changedFiles}"

                    // Danh sách tất cả các service và tên thư mục của chúng
                    def allServices = [
                        'customers-service', 'vets-service', 'visits-service', 'genai-service',
                        'api-gateway', 'discovery-server', 'config-server', 'admin-server'
                    ]
                    
                    // Lọc ra danh sách các thư mục service có thay đổi
                    def changedServiceDirs = allServices.findAll { serviceDir ->
                        changedFiles.contains("spring-petclinic-${serviceDir}")
                    }

                    if (changedServiceDirs.isEmpty()) {
                        echo "No changes detected in any service directory. Skipping build and push."
                        // Dùng biến SKIP để bỏ qua stage sau
                        env.SKIP_BUILD_PUSH = 'true'
                    } else {
                        // Lưu danh sách các thư mục cần build vào biến môi trường
                        // để sử dụng ở stage tiếp theo.
                        env.DIRS_TO_BUILD = changedServiceDirs.join(',')
                        env.SKIP_BUILD_PUSH = 'false'
                        echo "Will build and push images for: ${env.DIRS_TO_BUILD}"
                    }
                }
            }
        }
        
        // STAGE 2: BUILD, TAG VÀ PUSH DOCKER IMAGE
        stage('Build & Push Image') {
            // Chỉ chạy stage này khi có service thay đổi
            when {
                expression { env.SKIP_BUILD_PUSH == 'false' }
            }
            steps {
                script {
                    // Lấy commit ID (7 ký tự đầu) để làm tag cho image
                    def imageTag = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

                    // Sử dụng khối withCredentials để đăng nhập Docker Hub một cách an toàn
                    withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        echo "Logging into Docker Hub..."
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                    }
                    
                    // Chuyển chuỗi các thư mục thành một List để lặp qua
                    def dirList = env.DIRS_TO_BUILD.split(',')

                    // Lặp qua từng thư mục service đã thay đổi
                    dirList.each { serviceDir ->
                        echo "--- Processing: ${serviceDir} ---"
                        
                        // Chuyển vào thư mục của service đó
                        dir("spring-petclinic-${serviceDir}") {
                            // Xây dựng tên image: <username>/<tên-thư-mục>
                            def imageName = "${DOCKERHUB_USERNAME}/${serviceDir}"
                            
                            echo "Building image: ${imageName}:${imageTag}"
                            // Build image từ Dockerfile trong thư mục này
                            sh "docker build -t ${imageName}:${imageTag} ."
                            
                            echo "Pushing image: ${imageName}:${imageTag}"
                            // Push image với tag là commit ID lên Docker Hub
                            sh "docker push ${imageName}:${imageTag}"

                            // Yêu cầu 1: Nếu là nhánh main, push thêm tag 'main'
                            if (env.BRANCH_NAME == 'main') {
                                echo "Tagging and pushing 'main' tag for main branch."
                                sh "docker tag ${imageName}:${imageTag} ${imageName}:main"
                                sh "docker push ${imageName}:main"
                            }
                        }
                    }
                }
            }
            post {
                // Luôn chạy khối này, dù stage thành công hay thất bại
                always {
                    // Dọn dẹp: Đăng xuất khỏi Docker Hub
                    echo "Logging out from Docker Hub."
                    sh 'docker logout'
                }
            }
        }
    }
}