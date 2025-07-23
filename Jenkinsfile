pipeline {
    agent any
    
    environment {
        // Application configuration
        APP_NAME = 'vampi'
        APP_VERSION = "${env.BUILD_NUMBER}"
        DOCKER_IMAGE = "${APP_NAME}:${APP_VERSION}"
        
        // Docker Registry (OPTIONAL - only needed for pushing to remote registry)
        // Examples: 'docker.io/yourusername', 'your-registry.com', 'gcr.io/project-id'
        // Leave as 'localhost' or comment out if not using a registry
        DOCKER_REGISTRY = 'localhost' // Change to your registry URL when needed
        ENABLE_REGISTRY_PUSH = 'false' // Set to 'true' to enable pushing to registry
        
        // Environment variables for the app
        VULNERABLE_MODE = '1'
        TOKEN_TTL = '60'
        
        // Security scanning
        TRIVY_CACHE_DIR = '/tmp/trivy-cache'
        
        // Python environment
        PYTHON_VERSION = '3.11'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out VAmPI repository..."
                    git branch: 'main', 
                        url: 'https://github.com/anuragvajpayee/VAmPI.git'
                }
            }
        }
        
        stage('Environment Setup') {
            steps {
                script {
                    echo "Setting up Python virtual environment..."
                    sh '''
                        python3 -m venv venv
                        . venv/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt
                        pip install pytest pytest-cov bandit safety flake8
                    '''
                }
            }
        }
        
        stage('Code Quality & Security Analysis') {
            parallel {
                stage('Lint Code') {
                    steps {
                        script {
                            echo "Running code quality checks..."
                            sh '''
                                . venv/bin/activate
                                flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
                                flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
                            '''
                        }
                    }
                }
                
                stage('Security Scan - Bandit') {
                    steps {
                        script {
                            echo "Running Bandit security analysis..."
                            sh '''
                                . venv/bin/activate
                                bandit -r . -f json -o bandit-report.json || true
                                bandit -r . -f txt -o bandit-report.txt || true
                            '''
                        }
                        archiveArtifacts artifacts: 'bandit-report.*', allowEmptyArchive: true
                    }
                }
                
                stage('Dependency Check') {
                    steps {
                        script {
                            echo "Checking for known security vulnerabilities in dependencies..."
                            sh '''
                                . venv/bin/activate
                                safety check --json --output safety-report.json || true
                                safety check --output safety-report.txt || true
                            '''
                        }
                        archiveArtifacts artifacts: 'safety-report.*', allowEmptyArchive: true
                    }
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                script {
                    echo "Running unit tests..."
                    sh '''
                        . venv/bin/activate
                        # Create basic test structure if it doesn't exist
                        mkdir -p tests
                        
                        # Create a basic test file if none exists
                        if [ ! -f tests/test_app.py ]; then
                            cat > tests/test_app.py << EOF
import pytest
import sys
import os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..'))

from app import vuln_app

@pytest.fixture
def client():
    vuln_app.app.config['TESTING'] = True
    with vuln_app.app.test_client() as client:
        yield client

def test_home_endpoint(client):
    """Test the home endpoint"""
    response = client.get('/')
    assert response.status_code == 200
    assert b'VAmPI' in response.data

def test_createdb_endpoint(client):
    """Test database creation endpoint"""
    response = client.get('/createdb')
    assert response.status_code == 200
    assert b'Database populated' in response.data

def test_users_endpoint(client):
    """Test users endpoint"""
    # First create the database
    client.get('/createdb')
    response = client.get('/users/v1')
    assert response.status_code == 200

def test_books_endpoint(client):
    """Test books endpoint"""
    # First create the database
    client.get('/createdb')
    response = client.get('/books/v1')
    assert response.status_code == 200
EOF
                        fi
                        
                        # Create __init__.py if it doesn't exist
                        touch tests/__init__.py
                        
                        # Run tests with coverage
                        pytest tests/ --cov=. --cov-report=xml --cov-report=html --cov-report=term
                    '''
                }
            }
            post {
                always {
                    // Archive test results and coverage reports
                    archiveArtifacts artifacts: 'htmlcov/**/*', allowEmptyArchive: true
                    archiveArtifacts artifacts: 'coverage.xml', allowEmptyArchive: true
                    
                    // Publish test results if you have the JUnit plugin
                    // junit 'test-results.xml'
                    
                    // Publish coverage results if you have the Coverage plugin
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh '''
                        docker build -t ${DOCKER_IMAGE} .
                        docker tag ${DOCKER_IMAGE} ${APP_NAME}:latest
                    '''
                }
            }
        }
        
        stage('Container Security Scan') {
            steps {
                script {
                    echo "Scanning Docker image for vulnerabilities..."
                    sh '''
                        # Install Trivy if not available
                        if ! command -v trivy &> /dev/null; then
                            echo "Installing Trivy..."
                            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
                        fi
                        
                        # Create cache directory
                        mkdir -p ${TRIVY_CACHE_DIR}
                        
                        # Scan the image
                        trivy image --cache-dir ${TRIVY_CACHE_DIR} --format json --output trivy-report.json ${DOCKER_IMAGE} || true
                        trivy image --cache-dir ${TRIVY_CACHE_DIR} --format table --output trivy-report.txt ${DOCKER_IMAGE} || true
                    '''
                }
                archiveArtifacts artifacts: 'trivy-report.*', allowEmptyArchive: true
            }
        }
        
        stage('Integration Tests') {
            steps {
                script {
                    echo "Running integration tests..."
                    sh '''
                        # Start the container for testing
                        docker run -d --name ${APP_NAME}-test-${BUILD_NUMBER} -p 5000:5000 \
                            -e vulnerable=${VULNERABLE_MODE} \
                            -e tokentimetolive=${TOKEN_TTL} \
                            ${DOCKER_IMAGE}
                        
                        # Wait for the application to start
                        sleep 10
                        
                        # Run integration tests
                        . venv/bin/activate
                        
                        # Create integration test if it doesn't exist
                        mkdir -p integration_tests
                        cat > integration_tests/test_api_integration.py << EOF
import requests
import time
import json

def test_api_health():
    """Test if the API is responding"""
    response = requests.get('http://localhost:5000/')
    assert response.status_code == 200
    data = response.json()
    assert 'VAmPI' in data['message']

def test_database_creation():
    """Test database creation endpoint"""
    response = requests.get('http://localhost:5000/createdb')
    assert response.status_code == 200
    data = response.json()
    assert 'Database populated' in data['message']

def test_user_registration_and_login():
    """Test user registration and login flow"""
    # First ensure database is created
    requests.get('http://localhost:5000/createdb')
    
    # Register a new user
    user_data = {
        'username': 'testuser',
        'password': 'testpass',
        'email': 'test@example.com'
    }
    response = requests.post('http://localhost:5000/users/v1/register', json=user_data)
    assert response.status_code == 200
    
    # Login with the user
    login_data = {
        'username': 'testuser',
        'password': 'testpass'
    }
    response = requests.post('http://localhost:5000/users/v1/login', json=login_data)
    assert response.status_code == 200
    data = response.json()
    assert 'auth_token' in data

def test_swagger_ui():
    """Test that Swagger UI is accessible"""
    response = requests.get('http://localhost:5000/ui/')
    assert response.status_code == 200

if __name__ == '__main__':
    test_api_health()
    test_database_creation()
    test_user_registration_and_login()
    test_swagger_ui()
    print("All integration tests passed!")
EOF
                        
                        # Install requests if not available
                        pip install requests
                        
                        # Run integration tests
                        python integration_tests/test_api_integration.py
                    '''
                }
            }
            post {
                always {
                    script {
                        // Stop and remove test container
                        sh '''
                            docker stop ${APP_NAME}-test-${BUILD_NUMBER} || true
                            docker rm ${APP_NAME}-test-${BUILD_NUMBER} || true
                        '''
                    }
                }
            }
        }
        
        stage('Performance Tests') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    echo "Running performance tests..."
                    sh '''
                        # Start the container for performance testing
                        docker run -d --name ${APP_NAME}-perf-${BUILD_NUMBER} -p 5001:5000 \
                            -e vulnerable=0 \
                            -e tokentimetolive=${TOKEN_TTL} \
                            ${DOCKER_IMAGE}
                        
                        sleep 10
                        
                        # Install Apache Benchmark if not available
                        if ! command -v ab &> /dev/null; then
                            echo "Installing Apache Benchmark..."
                            apt-get update && apt-get install -y apache2-utils || \
                            yum install -y httpd-tools || \
                            apk add --no-cache apache2-utils
                        fi
                        
                        # Initialize database
                        curl -X GET http://localhost:5001/createdb
                        
                        # Run performance tests
                        echo "Running performance tests on home endpoint..."
                        ab -n 100 -c 10 http://localhost:5001/ > performance-home.txt
                        
                        echo "Running performance tests on users endpoint..."
                        ab -n 50 -c 5 http://localhost:5001/users/v1 > performance-users.txt
                        
                        echo "Performance test completed"
                    '''
                }
                archiveArtifacts artifacts: 'performance-*.txt', allowEmptyArchive: true
            }
            post {
                always {
                    script {
                        sh '''
                            docker stop ${APP_NAME}-perf-${BUILD_NUMBER} || true
                            docker rm ${APP_NAME}-perf-${BUILD_NUMBER} || true
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    echo "Deploying to staging environment..."
                    sh '''
                        # Stop existing staging container if running
                        docker stop ${APP_NAME}-staging || true
                        docker rm ${APP_NAME}-staging || true
                        
                        # Start new staging container
                        docker run -d --name ${APP_NAME}-staging \
                            -p 5002:5000 \
                            --restart unless-stopped \
                            -e vulnerable=0 \
                            -e tokentimetolive=300 \
                            ${DOCKER_IMAGE}
                        
                        # Health check
                        sleep 10
                        curl -f http://localhost:5002/ || exit 1
                        curl -f http://localhost:5002/createdb || exit 1
                        
                        echo "Staging deployment successful!"
                        echo "Application is running at: http://localhost:5002"
                        echo "Swagger UI available at: http://localhost:5002/ui/"
                    '''
                }
            }
        }
        
        stage('Push to Registry') {
            when {
                allOf {
                    branch 'main'
                    environment name: 'ENABLE_REGISTRY_PUSH', value: 'true'
                }
            }
            steps {
                script {
                    echo "🚀 Pushing Docker image to registry..."
                    echo "Registry: ${DOCKER_REGISTRY}"
                    sh '''
                        # Tag for registry
                        echo "Tagging images for registry: ${DOCKER_REGISTRY}"
                        docker tag ${DOCKER_IMAGE} ${DOCKER_REGISTRY}/${DOCKER_IMAGE}
                        docker tag ${DOCKER_IMAGE} ${DOCKER_REGISTRY}/${APP_NAME}:latest
                        
                        # Push to registry
                        echo "Pushing ${DOCKER_REGISTRY}/${DOCKER_IMAGE}..."
                        docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}
                        
                        echo "Pushing ${DOCKER_REGISTRY}/${APP_NAME}:latest..."
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                        
                        echo "✅ Successfully pushed images to registry!"
                    '''
                }
            }
        }
        
        stage('Registry Info (Optional)') {
            when {
                allOf {
                    branch 'main'
                    not { environment name: 'ENABLE_REGISTRY_PUSH', value: 'true' }
                }
            }
            steps {
                script {
                    echo "📝 Registry push is disabled"
                    echo "💡 To enable registry push:"
                    echo "   1. Set DOCKER_REGISTRY to your registry URL"
                    echo "   2. Set ENABLE_REGISTRY_PUSH to 'true'"
                    echo "   3. Configure registry credentials in Jenkins"
                    echo ""
                    echo "🏷️  Images built locally:"
                    sh '''
                        echo "   - ${DOCKER_IMAGE}"
                        echo "   - ${APP_NAME}:latest"
                        echo ""
                        echo "Available images:"
                        docker images ${APP_NAME} --format "table {{.Repository}}:{{.Tag}}\\t{{.CreatedAt}}\\t{{.Size}}"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Pipeline execution completed."
                
                // Clean up Docker images (keep last 5 builds)
                sh '''
                    # Clean up old images
                    docker images ${APP_NAME} --format "table {{.Tag}}" | grep -E '^[0-9]+$' | sort -nr | tail -n +6 | xargs -I {} docker rmi ${APP_NAME}:{} || true
                '''
                
                // Archive all reports
                archiveArtifacts artifacts: '**/*-report.*', allowEmptyArchive: true
            }
        }
        
        success {
            script {
                echo "✅ Pipeline completed successfully!"
                echo "📊 Check the archived artifacts for detailed reports"
                echo "🔒 Security reports: bandit-report.*, safety-report.*, trivy-report.*"
                echo "📈 Coverage report: htmlcov/index.html"
                echo "⚡ Performance reports: performance-*.txt"
                echo ""
                echo "🐳 Docker images built:"
                sh 'docker images ${APP_NAME} --format "table {{.Repository}}:{{.Tag}}\\t{{.Size}}"'
                echo ""
                if (env.ENABLE_REGISTRY_PUSH == 'true') {
                    echo "📦 Images pushed to registry: ${DOCKER_REGISTRY}"
                } else {
                    echo "💡 Registry push disabled. Set ENABLE_REGISTRY_PUSH='true' to enable."
                }
                
                // Send notifications (uncomment and configure as needed)
                // emailext (
                //     subject: "✅ VAmPI Pipeline Success - Build #${env.BUILD_NUMBER}",
                //     body: "The VAmPI pipeline has completed successfully. Check Jenkins for details.",
                //     to: "team@example.com"
                // )
            }
        }
        
        failure {
            script {
                echo "❌ Pipeline failed!"
                echo "🔍 Check the console output and reports for details"
                
                // Send failure notifications (uncomment and configure as needed)
                // emailext (
                //     subject: "❌ VAmPI Pipeline Failed - Build #${env.BUILD_NUMBER}",
                //     body: "The VAmPI pipeline has failed. Please check Jenkins for details.",
                //     to: "team@example.com"
                // )
            }
        }
        
        unstable {
            script {
                echo "⚠️  Pipeline completed with warnings"
                echo "🔍 Check the test results and reports for details"
            }
        }
        
        cleanup {
            script {
                // Clean up workspace
                echo "🧹 Cleaning up workspace..."
                sh '''
                    # Clean up test containers
                    docker ps -a | grep ${APP_NAME}-test || true
                    docker ps -a | grep ${APP_NAME}-perf || true
                    
                    # Remove virtual environment
                    rm -rf venv || true
                '''
                
                // Clean workspace
                cleanWs()
            }
        }
    }
}
