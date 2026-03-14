+++
title = 'AWS Security and Compliance Tools'
date = 2026-03-14T08:47:52+05:30
draft = true
+++



# AWS Security and Compliance Tools


- **AWS Shield** is a managed Distributed Denial of Service (DDoS) protection service that safeguards applications running on AWS. AWS Shield Standard mitigates attacks that occur at layers 3 and 4 of the OSI model. (With the help of the Shield Response Team (SRT), AWS Shield Advanced includes intelligent DDoS attack detection and mitigation for application layer (layer 7) attacks as well).



- **AWS WAF** is a web application firewall that helps protect your web applications or APIs against common web exploits.

- **Amazon Inspector** is an automated security assessment service that helps improve the security and compliance of applications deployed on AWS. 

- **Amazon Macie** is a security service that uses machine learning to automatically discover, classify, and protect sensitive data in AWS.

- **Amazon GuardDuty** is a threat detection service that continuously monitors for malicious activity and unauthorized behavior to protect multiple  AWS accounts and workloads.

- **AWS Security Hub** gives you a comprehensive view of your high-priority security alerts and compliance status across AWS accounts.

- **AWS Firewall Manager** AWS Firewall Manager enforces deployment of Web Application Firewall ACLs, AWS Shield Advanced protection policies, and VPC security groups and can report findings to AWS Security Hub. It also ensures compliance of new and existing resources, but it currently does not support configuration of Network ACLs.

- **AWS Service Catalog** allows organizations to create and manage catalogs of IT services that are approved for use on AWS. Deploying applications via AWS Service Catalog can help ensure that deployments meet organizational compliance needs, but it cannot compare recorded configuration changes against desired configurations.

- **AWS Config** is a service that enables you to assess, audit, and evaluate the configurations of your AWS resources. Config continuously monitors and records your AWS resource configurations and allows you to automate the evaluation of recorded configurations against desired configurations.

**You want to be notified when an AWS WAF ACL blocks connection attempts. Which AWS services should you employ to quickly accomplish your goals?**

- You can create alarms tied to the CloudWatch metric for the WAF ACL and have the alarm trigger a Simple Notification Service topic for which you have an email or text message subscription. 

- CloudTrail audits AWS API calls, not the network activity such as blocked connection attempts. (Incorrect)

- A Lambda function that notifies you of an event could be triggered by an SNS topic, but would take more time to configure than just an SNS topic and an email or SMS subscription.(Incorrect).


**What AWS service or feature can capture packet level details of VPC traffic?**

- Traffic Mirroring is an Amazon VPC feature that you can use to copy packet-level EC2 network traffic and send it to out-of-band security and monitoring appliances.

- VPC flow logs record session-level information, such as source and destination addresses and ports, and whether the traffic was allowed or rejected, for the IP traffic going to and from network interfaces in your VPC. It does not record packet-level details. 

- Neither CloudWatch metrics nor CloudTrail can capture network traffic.

**What AWS service could be used to ensure that VPC configuration changes are in compliance with organizational policies?**

AWS Config is a service that enables you to assess, audit, and evaluate the configurations of your AWS resources. Config continuously monitors and records your AWS resource configurations and allows you to automate the evaluation of recorded configurations against desired configurations.

- CloudWatch collects monitoring and operational data in the form of logs, metrics, and events, but cannot compare recorded configuration changes against desired configurations. 

- CloudTrail logs, continuously monitors, and retains account activity related to actions across your AWS infrastructure, but cannot compare recorded configuration changes against desired configurations. 

**True or false: Configuring the cache behavior of a CloudFront distribution to require HTTPS ensures that your client requests to the data origin will be secured for the entire session**

- Configuring the cache behavior of a CloudFront distribution to require HTTPS only secures the connection between the client and the CloudFront distribution content at an Edge Location. To ensure the entire connection from requesting client to data origin is secured, you must also configure the Origin Protocol Policy to use HTTPS.


**How can you ensure that an S3 CloudFront origin is accessed securely via the CloudFront distribution, and cannot be accessed directly?**

- Access to an S3 bucket origin via CloudFront can be ensured by disabling public access to the bucket and granting read-only access to an Origin Access Identity created in the CloudFront service. None of the other options restrict access to the S3 bucket.
