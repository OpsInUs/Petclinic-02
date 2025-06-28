pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
        DOCKERHUB_USERNAME = 'lshigami'
    }
    
    options {
        buildDiscarder(logRotator(
            numToKeepStr: '10',      // Giữ logs của 10 builds
            artifactNumToKeepStr: '5' // Chỉ giữ artifacts của 5 builds gần nhất
        ))
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    sh 'pwd'

                    def changedFiles = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim()
                    echo "Changed files: ${changedFiles}"

                    def changedServices = []

                    if (changedFiles.contains('spring-petclinic-genai-service')) {
                        changedServices.add('genai')
                    }
                    if (changedFiles.contains('spring-petclinic-customers-service')) {
                        changedServices.add('customers')
                    }
                    if (changedFiles.contains('spring-petclinic-vets-service')) {
                        changedServices.add('vets')
                    }
                    if (changedFiles.contains('spring-petclinic-visits-service')) {
                        changedServices.add('visits')
                    }
                    if (changedFiles.contains('spring-petclinic-api-gateway')) {
                        changedServices.add('api-gateway')
                    }
                    if (changedFiles.contains('spring-petclinic-discovery-server')) {
                        changedServices.add('discovery')
                    }
                    if (changedFiles.contains('spring-petclinic-config-server')) {
                        changedServices.add('config')
                    }
                    if (changedFiles.contains('spring-petclinic-admin-server')) {
                        changedServices.add('admin')
                    }

                    if (changedServices.isEmpty()) {
                        changedServices = ['all']
                    }

                    echo "Detected changes in services: ${changedServices}"

                    CHANGED_SERVICES_LIST = changedServices
                    CHANGED_SERVICES_STRING = changedServices.join(',')
                    echo "Changed services: ${CHANGED_SERVICES_STRING}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    if (CHANGED_SERVICES_LIST.contains('all')) {
                        echo 'Testing all modules'
                        sh './mvnw clean test'
                        // Debug: Kiểm tra nội dung thư mục target
                        sh 'find . -name "surefire-reports" -type d'
                        sh 'find . -name "jacoco.exec" -type f'
                    } else {
                        def modules = CHANGED_SERVICES_LIST.collect { "spring-petclinic-${it}-service" }.join(',')
                        echo "Testing modules: ${modules}"
                        sh "./mvnw clean test -pl ${modules}"
                        // Debug: Kiểm tra nội dung thư mục target
                        sh 'find . -name "surefire-reports" -type d'
                        sh 'find . -name "jacoco.exec" -type f'
                    }
                }
            }
            post {
                always {
                    script {
                        def testReportPattern = ''
                        def jacocoPattern = ''

                        if (CHANGED_SERVICES_LIST.contains('all')) {
                            testReportPattern = '**/surefire-reports/TEST-*.xml'
                            jacocoPattern = '**/jacoco.exec'
                        } else {
                            def patterns = CHANGED_SERVICES_LIST.collect {
                                "spring-petclinic-${it}-service/target/surefire-reports/TEST-*.xml"
                            }.join(',')
                            testReportPattern = patterns

                            def jacocoPatterns = CHANGED_SERVICES_LIST.collect {
                                "spring-petclinic-${it}-service/target/jacoco.exec"
                            }.join(',')
                            jacocoPattern = jacocoPatterns
                        }

                        echo "Looking for test reports with pattern: ${testReportPattern}"
                        sh "find . -name 'TEST-*.xml' -type f"

                        def testFiles = sh(script: "find . -name 'TEST-*.xml' -type f", returnStdout: true).trim()
                        if (testFiles) {
                            echo "Found test reports: ${testFiles}"
                            junit testReportPattern
                        } else {
                            echo 'No test reports found, likely no tests were executed.'
                        }

                        echo "Looking for JaCoCo data with pattern: ${jacocoPattern}"
                        sh "find . -name 'jacoco.exec' -type f"

                        def jacocoFiles = sh(script: "find . -name 'jacoco.exec' -type f", returnStdout: true).trim()
                        if (jacocoFiles) {
                            echo "Found JaCoCo files: ${jacocoFiles}"
                            jacoco(
                                execPattern: jacocoPattern,
                                classPattern: CHANGED_SERVICES_LIST.contains('all') ?
                                    '**/target/classes' :
                                    CHANGED_SERVICES_LIST.collect { "spring-petclinic-${it}-service/target/classes" }.join(','),
                                sourcePattern: CHANGED_SERVICES_LIST.contains('all') ?
                                    '**/src/main/java' :
                                    CHANGED_SERVICES_LIST.collect { "spring-petclinic-${it}-service/src/main/java" }.join(',')
                            )
                        } else {
                            echo 'No JaCoCo execution data found, skipping coverage report.'
                        }
                    }
                }
            }
        }

        stage('Build & Push Image') {
            when {
                // Chỉ chạy khi có thay đổi trong các service
                not { expression { CHANGED_SERVICES_LIST.contains('all') && CHANGED_SERVICES_LIST.size() == 1 } }
            }
            steps {
                script {
                    // Đăng nhập vào Docker Hub
                    withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                    }

                    // Build và push cho từng service đã thay đổi
                    def servicesToBuild = CHANGED_SERVICES_LIST.findAll { it != 'all' }
                    
                    servicesToBuild.each { serviceName ->
                        def serviceDir = "spring-petclinic-${serviceName}-service"
                        if (serviceName == 'api-gateway') {
                            serviceDir = "spring-petclinic-api-gateway"
                        } else if (serviceName == 'discovery') {
                            serviceDir = "spring-petclinic-discovery-server"
                        } else if (serviceName == 'config') {
                            serviceDir = "spring-petclinic-config-server"
                        } else if (serviceName == 'admin') {
                            serviceDir = "spring-petclinic-admin-server"
                        }
                        
                        dir(serviceDir) {
                            def imageName = "${DOCKERHUB_USERNAME}/${serviceName}-service"
                            if (serviceName in ['api-gateway', 'discovery', 'config', 'admin']) {
                                imageName = "${DOCKERHUB_USERNAME}/${serviceName}-server"
                            }
                            
                            // Yêu cầu 3: tag là commit id
                            def imageTag = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            
                            echo "Building image ${imageName}:${imageTag}"
                            sh "docker build -t ${imageName}:${imageTag} ."
                            
                            echo "Pushing image ${imageName}:${imageTag}"
                            sh "docker push ${imageName}:${imageTag}"

                            // Nếu là nhánh main, push thêm tag 'main'
                            if (env.BRANCH_NAME == 'main') {
                                sh "docker tag ${imageName}:${imageTag} ${imageName}:main"
                                sh "docker push ${imageName}:main"
                            }
                        }
                    }
                }
            }
            post {
                always {
                    // Đăng xuất khỏi Docker Hub
                    sh 'docker logout'
                }
            }
        }

stage('Check Code Coverage') {
    steps {
        script {
            def failedServices = []

            CHANGED_SERVICES_LIST.each { service ->
                if (service in ['customers', 'visits', 'vets']) {
                    def coverageReport = "spring-petclinic-${service}-service/target/site/jacoco/jacoco.xml"
                    def coverageThreshold = 70.0

                    def lineCoverage = sh(script: """
                        if [ -f ${coverageReport} ]; then
                            awk '
                                /<counter type="LINE"[^>]*missed=/ {
                                    split(\$0, a, "[ \\\"=]+");
                                    # Debug output
                                    print "Debug: Checking jacoco.xml for ${service}..." > "/dev/stderr";
                                    print "Raw line: " \$0 > "/dev/stderr";
                                    print "Array after split:" > "/dev/stderr";
                                    for (i in a) print "a[" i "] = " a[i] > "/dev/stderr";
                                    # Find missed and covered indices
                                    for (i in a) {
                                        if (a[i] == "missed") missed = a[i+1];
                                        if (a[i] == "covered") covered = a[i+1];
                                    }
                                    print "missed = " missed ", covered = " covered > "/dev/stderr";
                                    sum = missed + covered;
                                    print "sum (missed + covered) = " sum > "/dev/stderr";
                                    coverage = (sum > 0 ? (covered / sum) * 100 : 0);
                                    print "Coverage = " coverage "%" > "/dev/stderr";
                                    print "-----" > "/dev/stderr";
                                    # Output final coverage value to stdout
                                    print coverage;
                                }
                            ' ${coverageReport}
                        else
                            echo "File not found: ${coverageReport}" > "/dev/stderr"
                            echo "0"
                        fi
                    """, returnStdout: true).trim()

                    if (lineCoverage) {
                        echo "Code coverage for ${service}: ${lineCoverage}%"
                        def coverageValue = lineCoverage.toDouble()
                        if (coverageValue < coverageThreshold) {
                            failedServices.add(service)
                        }
                    } else {
                        echo "No coverage report found for ${service}, assuming 0%"
                        failedServices.add(service)
                    }
                }
            }

            if (!failedServices.isEmpty()) {
                error "The following services failed code coverage threshold (${coverageThreshold}%): ${failedServices.join(', ')}"
            }
        }
    }
}


        stage('Build') {
            steps {
                script {
                    if (CHANGED_SERVICES_LIST.contains('all')) {
                        echo 'Building all modules'
                        sh './mvnw clean package -DskipTests'
                    } else {
                        def modules = CHANGED_SERVICES_LIST.collect { "spring-petclinic-${it}-service" }.join(',')
                        echo "Building modules: ${modules}"
                        sh "./mvnw clean package -DskipTests -pl ${modules}"
                    }
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }
    }
}
