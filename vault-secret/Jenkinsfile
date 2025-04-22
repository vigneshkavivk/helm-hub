pipeline {
    agent any

    environment {
        VENV_PATH = 'venv-checkov'
        CHECKOV_OUTPUT = 'checkov_output.txt'
        REPO_URL = 'https://github.com/ajith-sekar/sonarqube1.git'
        REPO_BRANCH = 'master'
        INFLUX_TOKEN = credentials('INFLUX-CRED')   // Stored in Jenkins credentials
        INFLUX_ORG = 'CloudMasa'
        INFLUX_BUCKET = 'checkov'
        INFLUX_URL = 'http://3.85.34.66:8086/'
    }

    stages {
        stage('Clone Helm Repo') {
            steps {
                git branch: "${REPO_BRANCH}",
                    url: "${REPO_URL}",
                    credentialsId: 'git-credentials-id'
            }
        }

        stage('Set Up Virtual Environment and Install Checkov') {
            steps {
                sh '''
                    apt install python3.11-venv -y
                    python3 -m venv ${VENV_PATH}
                    . ${VENV_PATH}/bin/activate
                    ${VENV_PATH}/bin/pip install --upgrade pip
                    ${VENV_PATH}/bin/pip install checkov
                    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
                '''
            }
        }

        stage('Render Helm Charts') {
            steps {
                sh '''
                    echo "🧹 Cleaning previous rendered files..."
                    rm -rf rendered/
                    mkdir -p rendered
                    
                    echo " Rendering Helm charts into split files..."
                    mkdir -p rendered
                    find . -name "Chart.yaml" | while read chart_file; do
                        chart_dir=$(dirname "$chart_file")
                        chart_name=$(basename "$chart_dir" | tr '[:upper:]' '[:lower:]' | tr -cd 'a-z0-9-')
                        mkdir -p rendered/${chart_name}
                        helm template ${chart_name} ${chart_dir} --output-dir rendered/${chart_name}
                    done
                '''
            }
        }

        stage('Run Checkov on Rendered YAMLs') {
            steps {
                script {
                    def checkovExitCode = sh(
                        script: '''
                            . ${VENV_PATH}/bin/activate
                            echo "👉 Running Checkov on rendered templates..." > ${CHECKOV_OUTPUT}
                            ${VENV_PATH}/bin/checkov -d rendered/ --framework kubernetes --quiet | tee -a ${CHECKOV_OUTPUT}
                        ''',
                        returnStatus: true
                    )

                    sh "cat ${CHECKOV_OUTPUT}"

                    if (checkovExitCode != 0) {
                        error "❌Checkov found policy violations! Check ${CHECKOV_OUTPUT} for details."
                    } else {
                        echo "✅Checkov passed with no violations."
                    }
                }
            }
        }

        stage('Push Checkov Results to InfluxDB') {
            steps {
                script {
                    writeFile file: 'push_to_influx_fixed.py', text: '''\
        import os
        import re
        import sys
        from datetime import datetime
        from influxdb_client import InfluxDBClient, Point, WritePrecision
        from influxdb_client.client.write_api import SYNCHRONOUS
        
        def count_results(file_path):
            print(f"🔍 Analyzing file: {file_path}")
            failed, passed = 0, 0
            
            try:
                with open(file_path, "r") as f:
                    content = f.read()
                    
                    # Try multiple output patterns
                    patterns = [
                        r'Passed checks: (\\d+), Failed checks: (\\d+)',
                        r'(\\d+) passed, (\\d+) failed',
                        r'Passed: (\\d+), Failed: (\\d+)'
                    ]
                    
                    for pattern in patterns:
                        match = re.search(pattern, content)
                        if match:
                            passed = int(match.group(1))
                            failed = int(match.group(2))
                            print(f"✅ Found results: {passed} passed, {failed} failed")
                            return passed, failed
                    
                    print("⚠️ No matching pattern found in Checkov output")
                    return 0, 0
                    
            except Exception as e:
                print(f"❌ Error reading results file: {str(e)}")
                return 0, 0
        
        print("=== Starting InfluxDB Export ===")
        print(f"Using InfluxDB at: {os.getenv('INFLUX_URL')}")
        
        try:
            # Get results
            passed, failed = count_results(os.environ["CHECKOV_OUTPUT"])
            
            # Initialize client
            client = InfluxDBClient(
                url=os.environ["INFLUX_URL"],
                token=os.environ["INFLUX_TOKEN"],
                org=os.environ["INFLUX_ORG"]
            )
            
            # Simple connection test
            health = client.health()
            print(f"🏥 InfluxDB Health Status: {health.status}")
            print(f"📊 Writing to bucket: {os.environ['INFLUX_BUCKET']}")
            
            # Create data point
            point = Point("checkov_scan") \\
                .tag("source", "jenkins") \\
                .tag("pipeline", os.getenv("JOB_NAME", "unknown")) \\
                .field("passed", passed) \\
                .field("failed", failed) \\
                .time(datetime.utcnow(), WritePrecision.NS)
            
            print(f"📝 Data point: {point.to_line_protocol()}")
            
            # Write data
            write_api = client.write_api(write_options=SYNCHRONOUS)
            write_api.write(
                bucket=os.environ["INFLUX_BUCKET"],
                org=os.environ["INFLUX_ORG"],
                record=point
            )
            
            print("✅ Successfully wrote to InfluxDB")
            sys.exit(0)
            
        except Exception as e:
            print(f"❌ Failed to write to InfluxDB: {str(e)}")
            sys.exit(1)
        '''.stripIndent()
        
                    sh '''
                        . ${VENV_PATH}/bin/activate
                        echo "=== Running Fixed Script ==="
                        python3 push_to_influx_fixed.py || echo "Script exited with error"
                        
                        echo "=== Verifying Data ==="
                        curl -s -G "${INFLUX_URL}/query?pretty=true" \\
                          --header "Authorization: Token ${INFLUX_TOKEN}" \\
                          --data-urlencode "org=${INFLUX_ORG}" \\
                          --data-urlencode "q=SELECT * FROM checkov_scan ORDER BY time DESC LIMIT 1"
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "${CHECKOV_OUTPUT}", allowEmptyArchive: true
        }
    }
}
