pipeline {
  agent {
    node {
      label 'base'
    }

  }
  stages {
    stage('stage-rej3t') {
      agent none
      steps {
        echo "$params.NS,$params.APP_ID,$params.APP_KEY"
      }
    }

    stage('clone code') {
      agent none
      when {
        not {
          environment name: 'NS', value: 'dootask-test'
        }

      }
      steps {
        container('nodejs') {
          git(url: 'https://github.com/innet8/devops-k8s-deploy.git', credentialsId: '<no value>', branch: 'main', changelog: true, poll: false)
        }

      }
    }

    stage('deploy to dev') {
      agent none
      when {
        not {
          environment name: 'NS', value: 'dootask-test'
        }

      }
      steps {
        container('base') {
          input(id: 'deploy-to-dev', message: 'deploy to dev?')
          withCredentials([kubeconfigContent(credentialsId: 'k8s', variable: 'KUBECONFIG_CONFIG')]) {
            sh 'mkdir -p ~/.kube/'
            sh 'echo "$KUBECONFIG_CONFIG" > ~/.kube/config'
            sh 'envsubst < deploy/config.yaml| kubectl -n $NS apply -f - ;'
            sh 'kubectl wait --for=condition=Ready pod/dootask-mariadb-0 -n $NS --timeout=600s;kubectl -n $NS apply -f init-job.yaml'
            sh '''for file in deploy/*; do  [[ "$file" != "deploy/config.yaml" && "$file" != "deploy/ingress.yaml" ]] && kubectl -n $NS apply -f $file; done'''
            sh 'kubectl -n default get secret dootask.top -o yaml| sed "/namespace:/d;/uid:/d;/resourceVersion:/d" | kubectl -n $NS apply -f  -'
            sh 'envsubst < deploy/ingress.yaml| kubectl -n $NS apply -f - ;'
          }

        }

      }
      steps {
        container('base') {
          withCredentials([kubeconfigContent(credentialsId: 'k8s', variable: 'KUBECONFIG_CONFIG')]) {
            sh '''
              mkdir -p ~/.kube/
              echo "$KUBECONFIG_CONFIG" > ~/.kube/config
              kubectl patch configmap nginx-share-config -n dootask-saas-share --patch "{\"data\": {\"${APP_ID}.conf\": \"server {\\n    listen 80;\\n    server_name ${APP_ID}.dootask.top;\\n\\n    location / {\\n        proxy_set_header Host \\$host;\\n        proxy_pass http://nginx.dootask-${APP_ID}.svc;\\n    }\\n}\"}}"
            '''
          }
        }
      }
    }

  }
  environment {
    TAG = "$params.TAG"
    DB_PASSWORD = "$params.DB_PASSWORD"
    DB_ROOT_PASSWORD = "$params.DB_ROOT_PASSWORD"
    APP_KEY = "$params.APP_KEY"
    APP_ID = "$params.APP_ID"
    NS = "$params.NS"
  }
  parameters {
    string(name: 'TAG', defaultValue: 'pro', description: '')
    string(name: 'DB_PASSWORD', defaultValue: '', description: '')
    string(name: 'DB_ROOT_PASSWORD', defaultValue: '', description: '')
    string(name: 'APP_KEY', defaultValue: '', description: '')
    string(name: 'APP_ID', defaultValue: '', description: '')
    string(name: 'NS', defaultValue: 'dootask-test', description: '')
  }
}