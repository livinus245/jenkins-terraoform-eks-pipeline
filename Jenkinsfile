pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    skipDefaultCheckout(true)
    timeout(time: 120, unit: 'MINUTES')
  }

  parameters {
    choice(name: 'ACTION', choices: ['PLAN', 'INIT', 'APPLY', 'DESTROY'], description: 'Terraform action. Pull requests always run PLAN regardless of this value.')
    string(name: 'AWS_CREDENTIALS_ID', defaultValue: 'aws-jenkins-terraform', description: 'Jenkins AWS Credentials credential ID.')
    string(name: 'EXPECTED_AWS_ACCOUNT_ID', defaultValue: '768477844960', description: 'Expected 12-digit AWS account ID guardrail.')
    string(name: 'TF_STATE_BUCKET', defaultValue: '2026-state', description: 'Existing us-east-1 S3 bucket for Terraform state.')
    string(name: 'TF_STATE_KEY', defaultValue: 'eks/jenkins-eks.tfstate', description: 'S3 key for Terraform state.')
    string(name: 'TFVARS_FILE', defaultValue: 'terraform.tfvars.example', description: 'Optional repository-relative Terraform variable file.')
    string(name: 'CLUSTER_NAME', defaultValue: 'jenkins-eks', description: 'EKS cluster name.')
    string(name: 'KUBERNETES_VERSION', defaultValue: '1.36', description: 'EKS Kubernetes version. Empty uses the current AWS default.')
    string(name: 'ENVIRONMENT', defaultValue: 'production', description: 'Environment tag value.')
    string(name: 'NODE_INSTANCE_TYPE', defaultValue: 't3.medium', description: 'Managed-node EC2 instance type.')
    string(name: 'NODE_MIN_SIZE', defaultValue: '1', description: 'Minimum managed-node count.')
    string(name: 'NODE_DESIRED_SIZE', defaultValue: '2', description: 'Desired managed-node count.')
    string(name: 'NODE_MAX_SIZE', defaultValue: '4', description: 'Maximum managed-node count.')
    booleanParam(name: 'INIT_UPGRADE', defaultValue: false, description: 'Allow terraform init to upgrade provider selections.')
    booleanParam(name: 'AUTO_APPROVE', defaultValue: false, description: 'Skip the manual Jenkins approval prompt for APPLY or DESTROY.')
    booleanParam(name: 'CONFIRM_APPLY', defaultValue: false, description: 'Required confirmation for APPLY.')
    string(name: 'CONFIRM_DESTROY', defaultValue: '', description: 'Type DESTROY to authorize cluster destruction.')
  }

  environment {
    AWS_REGION = 'us-east-1'
    AWS_DEFAULT_REGION = 'us-east-1'
    AWS_PAGER = ''
    TF_IN_AUTOMATION = 'true'
    TF_INPUT = 'false'
    TF_CLI_ARGS = '-no-color'
    TF_PLAN_FILE = "eks-${BUILD_NUMBER}.tfplan"
    TF_DESTROY_PLAN_FILE = "eks-destroy-${BUILD_NUMBER}.tfplan"
    TF_STATE_BUCKET = "${params.TF_STATE_BUCKET}"
    TF_STATE_KEY = "${params.TF_STATE_KEY}"
    TFVARS_FILE = "${params.TFVARS_FILE}"
    INIT_UPGRADE = "${params.INIT_UPGRADE}"
    TF_VAR_cluster_name = "${params.CLUSTER_NAME}"
    TF_VAR_kubernetes_version = "${params.KUBERNETES_VERSION}"
    TF_VAR_environment = "${params.ENVIRONMENT}"
    TF_VAR_node_instance_type = "${params.NODE_INSTANCE_TYPE}"
    TF_VAR_node_min_size = "${params.NODE_MIN_SIZE}"
    TF_VAR_node_desired_size = "${params.NODE_DESIRED_SIZE}"
    TF_VAR_node_max_size = "${params.NODE_MAX_SIZE}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Resolve Action') {
      steps {
        script {
          if (env.CHANGE_ID) {
            if (env.CHANGE_FORK) {
              error('Pull requests from forks are not permitted to use this AWS-backed Jenkins plan job.')
            }
            env.EFFECTIVE_ACTION = 'PLAN'
            currentBuild.displayName = "#${env.BUILD_NUMBER} PR-${env.CHANGE_ID} PLAN"
          } else {
            env.EFFECTIVE_ACTION = params.ACTION
            currentBuild.displayName = "#${env.BUILD_NUMBER} ${params.ACTION}"
          }

          echo "Requested action: ${params.ACTION}; effective action: ${env.EFFECTIVE_ACTION}; AWS region: us-east-1"
        }
      }
    }

    stage('Validate Parameters and Tools') {
      steps {
        sh '''
          set -eu
          command -v terraform >/dev/null
          command -v aws >/dev/null
          terraform version
          aws --version

          for value in "$TF_STATE_BUCKET" "$TF_STATE_KEY" "$TF_VAR_cluster_name" "$TF_VAR_environment" "$TF_VAR_node_instance_type"; do
            test -n "$value"
          done

          case "$TF_VAR_cluster_name" in *[!A-Za-z0-9-]*) echo 'CLUSTER_NAME contains invalid characters'; exit 1;; esac
          case "$TF_VAR_environment" in *[!A-Za-z0-9_-]*) echo 'ENVIRONMENT contains invalid characters'; exit 1;; esac
          case "$TF_VAR_kubernetes_version" in ''|*[!0-9.]*) [ -z "$TF_VAR_kubernetes_version" ] || { echo 'KUBERNETES_VERSION is invalid'; exit 1; };; esac
          case "$TFVARS_FILE" in /*|*..*) echo 'TFVARS_FILE must be a safe repository-relative path'; exit 1;; esac

          for value in "$TF_VAR_node_min_size" "$TF_VAR_node_desired_size" "$TF_VAR_node_max_size"; do
            case "$value" in ''|*[!0-9]*) echo 'Node counts must be non-negative integers'; exit 1;; esac
          done

          if [ "$TF_VAR_node_min_size" -gt "$TF_VAR_node_desired_size" ] || [ "$TF_VAR_node_desired_size" -gt "$TF_VAR_node_max_size" ]; then
            echo 'Node counts must satisfy NODE_MIN_SIZE <= NODE_DESIRED_SIZE <= NODE_MAX_SIZE'
            exit 1
          fi

          if [ -n "$TFVARS_FILE" ] && [ ! -f "$TFVARS_FILE" ]; then
            echo "Terraform variable file not found: $TFVARS_FILE"
            exit 1
          fi

          if [ "$EFFECTIVE_ACTION" = 'APPLY' ] && [ "${CONFIRM_APPLY:-false}" != 'true' ]; then
            echo 'Set CONFIRM_APPLY=true before running APPLY.'
            exit 1
          fi

          if [ "$EFFECTIVE_ACTION" = 'DESTROY' ] && [ "${CONFIRM_DESTROY:-}" != 'DESTROY' ]; then
            echo 'Type DESTROY in CONFIRM_DESTROY before running DESTROY.'
            exit 1
          fi
        '''
      }
      environment {
        CONFIRM_APPLY = "${params.CONFIRM_APPLY}"
        CONFIRM_DESTROY = "${params.CONFIRM_DESTROY}"
      }
    }

    stage('Terraform Format') {
      steps {
        sh 'terraform fmt -recursive -check -diff'
      }
    }

    stage('Verify AWS Identity') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDENTIALS_ID]]) {
          sh '''
            set -eu
            ACTUAL_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
            echo "Authenticated to AWS account: $ACTUAL_ACCOUNT_ID in region $AWS_REGION"

            if [ -n "$EXPECTED_AWS_ACCOUNT_ID" ] && [ "$ACTUAL_ACCOUNT_ID" != "$EXPECTED_AWS_ACCOUNT_ID" ]; then
              echo "Expected AWS account $EXPECTED_AWS_ACCOUNT_ID but authenticated to $ACTUAL_ACCOUNT_ID"
              exit 1
            fi
          '''
        }
      }
      environment {
        EXPECTED_AWS_ACCOUNT_ID = "${params.EXPECTED_AWS_ACCOUNT_ID}"
      }
    }

    stage('Terraform Init') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDENTIALS_ID]]) {
          sh '''
            set -eu
            set -- terraform init -input=false -reconfigure \
              -backend-config="bucket=$TF_STATE_BUCKET" \
              -backend-config="key=$TF_STATE_KEY" \
              -backend-config="region=$AWS_REGION" \
              -backend-config="encrypt=true"

            if [ "$INIT_UPGRADE" = 'true' ]; then
              set -- "$@" -upgrade
            fi

            "$@"
          '''
        }
      }
    }

    stage('Terraform Plan') {
      when {
        expression { env.EFFECTIVE_ACTION in ['PLAN', 'APPLY'] }
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDENTIALS_ID]]) {
          sh '''
            set -eu
            set -- terraform plan -input=false -lock-timeout=5m -detailed-exitcode -out="$TF_PLAN_FILE"
            if [ -n "$TFVARS_FILE" ]; then
              set -- "$@" -var-file="$TFVARS_FILE"
            fi

            set +e
            "$@"
            PLAN_EXIT_CODE=$?
            set -e

            if [ "$PLAN_EXIT_CODE" -ne 0 ] && [ "$PLAN_EXIT_CODE" -ne 2 ]; then
              exit "$PLAN_EXIT_CODE"
            fi

            terraform show "$TF_PLAN_FILE" > terraform-plan.txt
            if [ "$PLAN_EXIT_CODE" -eq 2 ]; then
              echo 'Terraform plan contains changes.'
            else
              echo 'Terraform plan contains no changes.'
            fi
          '''
        }
      }
    }

    stage('Terraform Destroy Plan') {
      when {
        expression { env.EFFECTIVE_ACTION == 'DESTROY' }
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDENTIALS_ID]]) {
          sh '''
            set -eu
            set -- terraform plan -destroy -input=false -lock-timeout=5m -out="$TF_DESTROY_PLAN_FILE"
            if [ -n "$TFVARS_FILE" ]; then
              set -- "$@" -var-file="$TFVARS_FILE"
            fi
            "$@"
            terraform show "$TF_DESTROY_PLAN_FILE" > terraform-plan.txt
          '''
        }
      }
    }

    stage('Approval') {
      when {
        allOf {
          expression { env.EFFECTIVE_ACTION in ['APPLY', 'DESTROY'] }
          expression { !params.AUTO_APPROVE }
        }
      }
      steps {
        input message: "Run Terraform ${env.EFFECTIVE_ACTION} for EKS cluster '${params.CLUSTER_NAME}' in us-east-1?", ok: "Run ${env.EFFECTIVE_ACTION}"
      }
    }

    stage('Terraform Apply') {
      when {
        expression { env.EFFECTIVE_ACTION == 'APPLY' }
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDENTIALS_ID]]) {
          sh 'terraform apply -input=false -auto-approve "$TF_PLAN_FILE"'
        }
      }
    }

    stage('Terraform Destroy') {
      when {
        expression { env.EFFECTIVE_ACTION == 'DESTROY' }
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDENTIALS_ID]]) {
          sh 'terraform apply -input=false -auto-approve "$TF_DESTROY_PLAN_FILE"'
        }
      }
    }

    stage('Verify EKS Cluster') {
      when {
        expression { env.EFFECTIVE_ACTION == 'APPLY' }
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDENTIALS_ID]]) {
          sh '''
            set -eu
            aws eks wait cluster-active --region us-east-1 --name "$TF_VAR_cluster_name"
            aws eks describe-cluster \
              --region us-east-1 \
              --name "$TF_VAR_cluster_name" \
              --query 'cluster.{Name:name,Status:status,Version:version,Endpoint:endpoint}' \
              --output table
            terraform output > terraform-outputs.txt
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'terraform-plan.txt, terraform-outputs.txt', allowEmptyArchive: true, fingerprint: true
    }
    success {
      echo "Terraform ${env.EFFECTIVE_ACTION} completed successfully in us-east-1."
    }
  }
}
