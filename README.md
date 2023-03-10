# About

This repository provides a guide for setting up a Kubernetes cluster on AWS EKS and deploying a Cosign Controller for signing and verifying container images. It also includes instructions for integrating Amazon Inspector with the EKS cluster and using AWS Lambda functions to evaluate the resulting security insights. By following these steps, users can enhance the security of their EKS cluster and gain greater visibility into potential vulnerabilities.

# Context
Containerization has changed software development, deployment, management, packaging, and distributing applications. However, securing container images is critical to guard against data breaches, especially since containers rely on a complex network of dependencies that may harbor vulnerabilities.

Since development teams frequently pulls in new code and dependencies from a variety of sources, including public repositories and community-driven codebases.

Without proper supply chain security measures, these dependencies can introduce vulnerabilities and weaknesses into the company's software, potentially leading to data breaches, malware infections, or other security incidents. A single weak link in the supply chain can lead to the entire software stack being compromised

By implementing supply chain security measures for containers, the company can ensure that all dependencies are properly vetted and validated before they are used in production environments. This includes using digital signatures and checksums to verify the authenticity and integrity of each container image, as well as implementing strict access controls and permissions to prevent unauthorized changes to the container images

 To ensure the security of container images on Amazon Elastic Kubernetes Service (EKS), the following best practices can be followed:

-   Use trusted base images from reputable sources

-   Restrict access to Docker registries and enforce role-based access control (RBAC) to manage push and pull of images

-   Employ vulnerability scanning tools to identify security risks in third-party dependencies

-   Monitor container images for vulnerabilities and patch them promptly

-   Utilize automated testing and validation to ensure that images are free of vulnerabilities and meet quality standards


By following these supply chain best practices, container images on EKS can be made more secure, and the risk of potential data breaches or attacks can be mitigated. These best practices ensure that only authenticated and trusted images are deployed in production environments, thereby improving the overall security of the software infrastructure.

Image signatures can be used to guarantee that only authorized images are deployed in production. On EKS, image signatures can be implemented using KMS + Cosign.

This blog post covers best practices for securing Docker images on Amazon EKS using KMS and Cosign, as well as how to scan ECR images using Amazon Inspector.

# Deploying the solution
For the first step, a KMS key for signing container images can be created using the following command:

    aws kms create-key --region <region>

The Cosign CLI can then be installed on the local machine used for signing container images using the following command:

    curl -LO https://github.com/sigstore/cosign/releases/download/v0.5.1/cosign-linux-amd64

    sudo install cosign-linux-amd64 /usr/local/bin/cosign

Container images can be signed using the Cosign CLI and the KMS key using the following command:

    cosign sign -key <kms_key_arn> <registry>/<image>:<tag>

The validity of container image signatures can be verified using the Cosign CLI using the following command:

    cosign verify <registry>/<image>:<tag>

By implementing image signatures using KMS + Cosign on EKS, only authorized and authenticated images can be deployed in production environments, providing an additional layer of security against unauthorized or malicious images.

Amazon Inspector is a security assessment service that can help identify potential security risks in applications and infrastructure. By scanning Elastic Container Registry (ECR) images using Amazon Inspector on EKS, security risks can be identified and addressed before deploying images to production. To scan ECR images, the following steps can be taken:

-   Create an Amazon Inspector assessment target for the ECR registry using the following command:

`aws inspector create-assessment-target --assessment-target-name <target_name> --resource-group-arns <resource_group_arns> --region <region>`

-   Schedule a recurring assessment run for the assessment target using the following command:

`aws inspector create-assessment-template --assessment-target-arn <assessment_target_arn> --assessment-template-name <template_name> --duration-in-seconds <duration> --rules-package-arns <rules_package_arns> --user-attributes-for-findings <attributes> --region <region>`

By scanning ECR images using Amazon Inspector on EKS, security vulnerabilities and risks can be identified in container images before they are deployed to production. This proactive approach enables organizations to address security issues promptly and reduces the likelihood of potential data breaches or attacks.

To automate the evaluation of Amazon Inspector findings on EKS, one suggestion is to use an EventBridge rule to trigger a Lambda function, for example. This Lambda function can be configured to evaluate the findings and take appropriate actions based on the severity of the issues found, like the example below:

-   First create the lambda and it's execution role:

`aws iam create-role --role-name my_execution_role --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}' --description "Execution role for my Lambda function" --region us-east-1`

`aws iam attach-role-policy --role-name my_execution_role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --region us-east-1`

-   Now, create the lambda based on the local file:

`aws lambda create-function --function-name my_lambda_function --runtime python3.8 --role arn:aws:iam::123456789012:role/my_execution_role --handler lambda_function.lambda_handler --zip-file fileb://./lambda.zip --region us-east-1`

-   Next, create an Amazon EventBridge rule to trigger a Lambda function when assessment findings are generated using the following commands:

`aws events put-rule --name <rule_name> --event-pattern '{"source": ["aws.inspector"],"detail-type": ["Inspector Assessment Run Completed"],"resources": ["<assessment_template_arn>"]}' --region <region>`
`aws events put-targets --rule <rule_name> --targets '[{"arn": "<lambda_function_arn>","id": "<id>"}]' --region <region>`

Next, install the CoSign controller on EKS to help validate image signatures. To install the CoSign controller, the following steps can be followed:

-   Deploy the CoSign controller to a Kubernetes cluster using the following command:

`kubectl apply -f https://raw.githubusercontent.com/sigstore/cosign/main/helm/cosign-controller.yaml`

-   Verify that the controller has been deployed successfully using the following command:

`kubectl rollout status deployment/cosign-controller`

By installing the CoSign controller on EKS, container images can be authenticated and validated before they are deployed in production environments. This helps prevent unauthorized or malicious images from being used, thereby improving the overall security of container images.

![Cosign](./cosign.jpg)

The CoSign controller on Amazon EKS validates images by intercepting requests to pull container images from a registry, verifying the signature of the image, and only allowing images with valid signatures to be deployed to the cluster. This ensures that only trusted and authenticated images are used, improving the security of container images.

In conclusion, securing container images is crucial to protect against potential data breaches and attacks. By following supply chain best practices, implementing image signatures using KMS + Cosign, scanning ECR images using Amazon Inspector, triggering a Lambda function to evaluate Inspector findings, and installing the CoSign controller to validate image signatures on EKS, container images can be made more secure. These measures enable organizations to take a proactive approach to container image security and improve the overall security of their software infrastructure.

In addition to enhancing the overall security, implementing supply chain security measures can also help the company comply with industry regulations and standards. Many regulatory frameworks, such as PCI-DSS and HIPAA, require organizations to implement strict supply chain security measures to protect sensitive data.

