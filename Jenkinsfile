pipeline {
  agent any

  environment {
    REGISTRY           = 'index.docker.io'
    IMAGE_REPO         = 'YOUR_DOCKERHUB_USER/simple-app'
    FULL_IMAGE         = "${REGISTRY}/${IMAGE_REPO}"

    MANIFESTS_REPO_URL = 'https://github.com/ORG/simple-manifests.git'
    MANIFESTS_BRANCH   = 'main'
    DEPLOYMENT_FILE    = 'apps/myapp/prod/deployment.yaml'
    # Optional: pin to image name string used in the YAML for reliable replacement
    IMAGE_NAME_IN_YAML = "index.docker.io/YOUR_DOCKERHUB_USER/simple-app"

    GIT_USER_NAME      = 'jenkins-bot'
    GIT_USER_EMAIL     = 'jenkins-bot@example.com'

    DOCKER_CREDS_ID    = 'docker-registry-creds'
    GIT_BOT_CREDS_ID   = 'git-bot-creds'
  }

  triggers {
    // Use webhook in job settings ideally; fallback polling here
    pollSCM('H/2 * * * *')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          def ts  = sh(returnStdout: true, script: "date +%Y%m%d%H%M%S").trim()
          def sha = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
          env.IMAGE_TAG = "${ts}-${sha}"
          echo "Image tag: ${env.IMAGE_TAG}"
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh """
            echo "${DOCKER_PASS}" | docker login ${REGISTRY} -u "${DOCKER_USER}" --password-stdin
            docker build -t ${FULL_IMAGE}:${IMAGE_TAG} .
            docker push ${FULL_IMAGE}:${IMAGE_TAG}
            docker tag ${FULL_IMAGE}:${IMAGE_TAG} ${FULL_IMAGE}:latest
            docker push ${FULL_IMAGE}:latest
          """
        }
      }
    }

    stage('Update Manifests Repo (bump tag in deployment.yaml)') {
      steps {
        dir('manifests-workdir') {
          withCredentials([usernamePassword(credentialsId: "${GIT_BOT_CREDS_ID}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              set -e
              git config --global user.email "${GIT_USER_EMAIL}"
              git config --global user.name  "${GIT_USER_NAME}"

              AUTH_URL="$(echo "${MANIFESTS_REPO_URL}" | sed -E 's#https://#https://'${GIT_USER}':'${GIT_TOKEN}'@#')"
              git clone --branch "${MANIFESTS_BRANCH}" "${AUTH_URL}" .
              echo "Bumping image tag in ${DEPLOYMENT_FILE} to ${IMAGE_TAG}"

              # Replace only the tag part for the specific image
              # Finds lines like: image: index.docker.io/<user>/simple-app:anything
              sed -i -E 's#(image:\\s*'${IMAGE_NAME_IN_YAML}':).*#\\1'${IMAGE_TAG}'#' "${DEPLOYMENT_FILE}"

              echo "Updated deployment.yaml:"
              sed -n '1,120p' "${DEPLOYMENT_FILE}"

              git add "${DEPLOYMENT_FILE}"
              git commit -m "ci: bump ${FULL_IMAGE} to ${IMAGE_TAG}"
              git push origin "${MANIFESTS_BRANCH}"
            """
          }
        }
      }
    }
  }

  post {
    always {
      sh "docker logout ${REGISTRY} || true"
    }
  }
}
``
