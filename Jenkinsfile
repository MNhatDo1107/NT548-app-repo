// Jenkinsfile – gốc NT548-app-repo
// Lưu ý: 'ai-agent' tạm bỏ khỏi ALL vì src/ai-agent chưa có (Hoàng commit sau). Khi có thì thêm lại vào cuối danh sách.
def ALL = ['frontend', 'cartservice', 'productcatalogservice', 'currencyservice', 'paymentservice',
           'shippingservice', 'emailservice', 'checkoutservice', 'recommendationservice',
           'adservice', 'loadgenerator']
def GO_SVCS = ['frontend', 'productcatalogservice', 'shippingservice', 'checkoutservice']
def SONAR_SKIP = ['cartservice', 'adservice'] // C# và Java cần scanner riêng
def ctx(String s) { return s == 'cartservice' ? 'src/cartservice/src' : "src/${s}" }
def changed() { return env.CHANGED?.trim() ? true : false }

pipeline {
  agent any
  options {
    disableConcurrentBuilds()
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }
  environment {
    REG = 'asia-southeast1-docker.pkg.dev/project-d833ce9b-518c-4957-bad/boutique'
    CONFIG_REPO = 'git@github.com:nhom8-nt548/nt548-manifest-repo.git'
    UPSTREAM = 'us-central1-docker.pkg.dev/online-boutique-ci/microservices-demo'
  }
  stages {
    stage('1. Detect changes') {
      steps {
        script {
          env.SHA = env.GIT_COMMIT.take(7)
          def base = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          if (!base) {
            env.CHANGED = ALL.join(' ') // lần đầu: build tất cả
          } else {
            def files = sh(returnStdout: true, script: "git diff --name-only ${base} HEAD").trim().split('\n') as List
            env.CHANGED = ALL.findAll { s -> files.any { it.startsWith("src/${s}/") } }.join(' ')
          }
          echo "Service thay doi: ${env.CHANGED ?: '(khong co)'}"
        }
      }
    }

    stage('2. IaC scan - Checkov') {
      when { allOf { branch 'staging'; expression { changed() } } }
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'config-repo-ssh', keyFileVariable: 'KEY')]) {
          sh '''
            export GIT_SSH_COMMAND="ssh -i $KEY -o StrictHostKeyChecking=accept-new"
            rm -rf cfg-scan && git clone --depth 1 $CONFIG_REPO cfg-scan
            FILES=""
            for d in overlays/staging overlays/prod platform/ai-agent; do
              [ -d cfg-scan/$d ] || continue
              n=$(echo $d | tr / -)
              kustomize build cfg-scan/$d > rendered-$n.yaml && FILES="$FILES -f rendered-$n.yaml"
            done
            CFG=""; [ -f cfg-scan/.checkov.yaml ] && CFG="--config-file cfg-scan/.checkov.yaml"
            checkov $FILES --framework kubernetes $CFG --compact
          '''
        }
      }
    }

    stage('3. Unit test') {
      when { expression { changed() } }
      steps {
        script {
          for (s in env.CHANGED.split(' ')) {
            if (s in GO_SVCS) {
              sh """
                IMG=\$(grep -m1 -oE 'golang:[^ ]+' src/${s}/Dockerfile | cut -d@ -f1)
                docker run --rm -e CGO_ENABLED=0 -v go-mod-cache:/go/pkg/mod -v "\$PWD/src/${s}":/src -w /src \$IMG go test ./...
              """
            } else if (s == 'cartservice') {
              sh '''
                IMG=$(grep -m1 -oE 'dotnet/sdk:[^ ]+' src/cartservice/src/Dockerfile | cut -d@ -f1)
                docker run --rm -v "$PWD/src/cartservice":/src -w /src mcr.microsoft.com/$IMG dotnet test tests/
              '''
            } else {
              echo "${s}: chua co unit test"
            }
          }
        }
      }
    }

    stage('4. SCA - OWASP Dependency-Check') {
      when { allOf { branch 'staging'; expression { changed() } } }
      steps {
        script {
          for (s in env.CHANGED.split(' ')) {
            sh "mkdir -p odc-${s}"
            dependencyCheck odcInstallation: 'OWASP-DC',
              additionalArguments: "--scan src/${s} --project ${s} --format HTML --format XML --out odc-${s} --noupdate --enableExperimental"
          }
        }
      }
      post { always { dependencyCheckPublisher pattern: 'odc-*/dependency-check-report.xml' } }
    }

    stage('5. SAST - SonarQube') {
      when { allOf { branch 'staging'; expression { changed() } } }
      steps {
        script {
          def scanner = tool 'SonarScanner'
          def failed = []
          for (s in env.CHANGED.split(' ')) {
            if (s in SONAR_SKIP) {
              echo "${s}: bo qua SonarQube"
            } else {
              try {
                withSonarQubeEnv('SonarQube') {
                  sh "${scanner}/bin/sonar-scanner -Dsonar.projectKey=boutique-${s} -Dsonar.projectName=${s} -Dsonar.sources=src/${s} -Dsonar.exclusions=**/*_test.go,**/node_modules/**,**/genproto/**"
                }
                timeout(time: 5, unit: 'MINUTES') {
                  def qg = waitForQualityGate()
                  if (qg.status != 'OK') { error "${s}: Quality Gate ${qg.status}" }
                }
              } catch (hudson.AbortException e) {
                echo "SONAR FAIL ${s}: ${e.message}"
                failed << s
              }
            }
          }
          if (failed) { error "SonarQube khong dat: ${failed.join(', ')}" }
        }
      }
    }

    stage('6. Build image') {
      when { allOf { branch 'staging'; expression { changed() } } }
      steps {
        script {
          for (s in env.CHANGED.split(' ')) {
            sh "docker build -t ${env.REG}/${s}:${env.SHA} ${ctx(s)}"
          }
        }
      }
    }

    stage('7. Image scan - Trivy') {
      when { allOf { branch 'staging'; expression { changed() } } }
      steps {
        script {
          def failed = []
          for (s in env.CHANGED.split(' ')) {
            try {
              sh "trivy image --no-progress --severity CRITICAL --ignore-unfixed --exit-code 1 --output trivy-${s}.txt ${env.REG}/${s}:${env.SHA}"
            } catch (hudson.AbortException e) {
              echo "TRIVY FAIL ${s}"
              failed << s
            }
          }
          if (failed) { error "Trivy CRITICAL: ${failed.join(', ')}" }
        }
      }
    }

    stage('8. Push to Artifact Registry') {
      when { allOf { branch 'staging'; expression { changed() } } }
      steps {
        script {
          for (s in env.CHANGED.split(' ')) {
            sh "docker push ${env.REG}/${s}:${env.SHA}"
          }
        }
      }
    }

    stage('9. Update NT548-manifest') {
      when { allOf { branch 'staging'; expression { changed() } } }
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'config-repo-ssh', keyFileVariable: 'KEY')]) {
          sh '''
            export GIT_SSH_COMMAND="ssh -i $KEY -o StrictHostKeyChecking=accept-new"
            rm -rf cfg && git clone --depth 1 $CONFIG_REPO cfg && cd cfg
            for s in $CHANGED; do
              if [ "$s" = "ai-agent" ]; then D=platform/ai-agent; N=$s; else D=overlays/staging; N=$UPSTREAM/$s; fi
              (cd $D && kustomize edit set image $N=$REG/$s:$SHA)
            done
            git -c user.name=jenkins -c user.email=jenkins@nhom8.local commit -am "staging: $CHANGED -> $SHA" || exit 0
            git pull --rebase origin main
            git push origin HEAD:main
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'trivy-*.txt, odc-*/dependency-check-report.html', allowEmptyArchive: true
      sh 'docker image prune -f || true'
    }
    failure {
      withCredentials([string(credentialsId: 'discord-webhook', variable: 'HOOK')]) {
        sh '''curl -s -H "Content-Type: application/json" -d "{\\"content\\": \\"CI that bai: $JOB_NAME #$BUILD_NUMBER ($CHANGED) $BUILD_URL\\"}" "$HOOK" || true'''
      }
    }
  }
}
