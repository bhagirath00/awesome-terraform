# The Deployment Script: Implementing a Terraform Jenkinsfile

A `Jenkinsfile` is a text file that contains the definition of a Jenkins Pipeline and is checked into source control. In a professional Terraform project, the Jenkinsfile defines the entire lifecycle of the infrastructure—from the initial syntactic validation to the final production application. By using the Declarative Pipeline syntax, you can create a readable, versioned, and highly maintainable automation script that serves as the "Runbook" for your cloud foundation. This chapter provides a definitive deep dive into the structure of a production-grade Terraform Jenkinsfile, focusing on environment scoping, the implementation of manual approval gates, and error handling.

## 1. Declarative Pipeline Structure

A modern Jenkinsfile for Terraform follows a standard "Stages" model.

```groovy
pipeline {
    agent { label 'terraform-agent' }

    environment {
      TF_IN_AUTOMATION = 'true'
      TF_LOG           = 'INFO'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Init') {
            steps {
                sh 'terraform init -input=false'
            }
        }

        stage('Security Scan') {
            steps {
                sh 'tfsec .'
            }
        }

        stage('Terraform Plan') {
            steps {
                sh 'terraform plan -out=tfplan -input=false'
                // Archive the plan for audit and for the Apply stage
                sh 'terraform show -no-color tfplan > tfplan.txt'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'tfplan, tfplan.txt'
                }
            }
        }

        stage('Manual Approval') {
            when { branch 'main' }
            steps {
                input message: "Review the plan and approve the deployment to production?"
            }
        }

        stage('Terraform Apply') {
            when { branch 'main' }
            steps {
                sh 'terraform apply -input=false tfplan'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            // Slack/Teams notification integration
            echo "Deployment Failed!"
        }
    }
}
```

## 2. Managing Environment Variables

Terraform looks for specifically named environment variables to control its behavior in non-interactive shells.
- **`TF_IN_AUTOMATION`**: When set to `true`, Terraform adjusts its output to be more suitable for logs (e.g., removing suggestions to run other commands).
- **`TF_CLI_ARGS`**: Can be used to pass global flags like `-no-color` to every command in the pipeline without repeating them in every script block.

## 3. The "Stage" Logic: Isolation and Visibility

### 3.1 The Plan Artifact
It is a technical best practice to archive the `tfplan` binary. Jenkins allows you to "Stash" and "Unstash" files between different agents or keep them as "Artifacts" for historical auditing. If a production failure occurs, you can download the exact plan that was applied to see what Terraform thought the reality was at that moment.

### 3.2 Branch Protection
Note the use of the `when { branch 'main' }` directive. This is a critical security control. It ensures that while any developer can trigger a `Plan` by pushing to a feature branch, ONLY a push (merge) to the `main` branch can trigger the `Apply` stage.

## 4. Handling Terraform Inputs

In a professional environment, you often have variable files for different environments (`prod.tfvars`, `dev.tfvars`). Your Jenkinsfile can dynamically select these based on a parameter.

```groovy
stage('Terraform Plan') {
    steps {
        sh "terraform plan -var-file=${params.ENVIRONMENT}.tfvars -out=tfplan"
    }
}
```

## 5. Post-Deployment Cleanup

A robust Jenkinsfile must always include a `post` block to clean up the workspace. Terraform providers often leave behind temporary files or cached plugins in the `.terraform` directory. Using `cleanWs()` ensures that the next build starts with a clean slate, preventing state contamination between different job runs.

## 6. Summary

The Jenkinsfile is the "Constitution" of your infrastructure automation. By encoding every step of the validation and deployment process into a version-controlled script, you eliminate the variability and risk of manual operations. Through the use of declarative stages, manual approval gates, and environment-scoped logic, you build a deployment system that is both highly automated and strictly governed. Mastering the creation and maintenance of a production-grade Jenkinsfile is an essential skill for any senior DevOps professional managing professional cloud foundations at enterprise scale.
