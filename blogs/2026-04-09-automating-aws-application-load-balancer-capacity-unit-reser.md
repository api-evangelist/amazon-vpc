---
title: "Automating AWS Application Load Balancer Capacity Unit Reservation"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/automating-aws-application-load-balancer-capacity-unit-reservation/"
date: "Thu, 09 Apr 2026 15:39:57 +0000"
author: "Abhishek Dey"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
<p>Building resilient and fault-tolerant systems in <a href="https://aws.amazon.com/">Amazon Web Services (AWS)</a> is essential for maintaining stable workloads. When designing cloud architecture, the ability to handle sudden traffic surges becomes a critical consideration. <a href="https://aws.amazon.com/elasticloadbalancing/">Elastic Load Balancing (ELB)</a> serves as the primary entry point for distributing both external and internal traffic efficiently across applications. In this post, we explore a solution that automates <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/application/capacity-unit-reservation.html">Load Balancer Capacity Unit (LCU)</a> reservation for <a href="https://aws.amazon.com/elasticloadbalancing/application-load-balancer/">Application Load Balancers (ALB)</a> in an <a href="https://aws.amazon.com/organizations/">AWS Organizations</a> multi-account environment, helping prepare for sharp traffic increases across multiple load balancers.</p> 
<p>This solution uses the <a href="https://boto3.amazonaws.com/v1/documentation/api/1.24.83/reference/services/elbv2.html">ELBV2 API</a> to automate ALB capacity provisioning, pre-warming ALBs before anticipated traffic surges and resetting LCU values after the event. The solution automates these processes and thus helps optimize LCU costs while maintaining performance during peak traffic periods.</p> 
<p>This solution can scale to handle hundreds of ALBs running in an Organizations environment. It identifies ALBs based on specific tags and sets the LCU reservation values accordingly. Manually managing such a large fleet of ALBs could be cumbersome and error-prone, making this automated approach particularly valuable. You can deploy this solution in your AWS account by downloading the <a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> template hosted on AWS public <a href="https://github.com/aws/elastic-load-balancing-tools/tree/master/automating-aws-application-load-balancer-capacity-reservation">GitHub repository.</a></p> 
<p>Some of the typical use cases that may necessitate LCU reservation across multiple load balancers include the following:</p> 
<ul> 
 <li>Stock trading companies (FSI Capital Markets)</li> 
 <li>Sports event organizers</li> 
 <li>Ticket sales platforms</li> 
 <li>Large advertising campaigns</li> 
</ul> 
<p>Our previous post, <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/using-load-balancer-capacity-unit-reservation-to-prepare-for-sharp-increases-in-traffic/">Using Load Balancer Capacity Unit Reservation to prepare for sharp increases in traffic</a>, explained the feature and its use cases. This post focuses on implementing it in a scalable and automated way, across various load balancers and AWS accounts.</p> 
<h2>ALB scaling and capacity reservation</h2> 
<p>An LCU measures the maximum resource consumption across different dimensions of ALB traffic processing, such as new connections, active connections, bandwidth, and rule evaluations. You can use LCU to reserve a minimum capacity for your load balancer for a specified period. Although ALBs automatically scale to handle organic traffic growth by adding capacity. It effectively manages most workloads and traffic variations. You should consider LCU reservation when you anticipate traffic spikes that more than double your normal traffic volume in less than five minutes.</p> 
<p>The following graph demonstrates an ALB’s LCU consumption during a traffic surge, comparing provisioned capacity against actual LCU utilization.</p> 
<p><img alt="Graph shows the ALB peak LCU v/s reserved LCU" class="aligncenter wp-image-32184" height="292" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/ALB-peak-LCU-vs-reserved-LCU-graph-1-1024x329.png" width="910" /></p> 
<p style="text-align: center;"><em>Figure 1: Load Balancer Capacity Units (LCU) peak usage vs. reserved LCU&nbsp;</em></p> 
<h2>Managing predictable yet volatile trading platform traffic</h2> 
<p style="text-align: left;"><span style="font-size: 11.0pt;">In this post we explore a scenario for a Financial Services Industry – Capital Markets use case for LCU reservation for stock trading applications. </span><span style="font-size: 11.0pt;">Stock trading applications experience predictable traffic patterns, with peak volumes during market hours (9:00 AM – 4:00 PM) and spikes at market open and close, especially on Mondays or during significant events. This creates common challenges, particularly when markets reopen after weekends, as applications face a sudden surge in traffic. </span><span style="font-size: 11.0pt;">Stock trading companies often use multi-account architectures with multiple ALBs to meet regulatory requirements, enhance security, and make sure of fault isolation. Although this setup is essential for handling millions of concurrent users, it complicates ALB capacity management across accounts. Automating ALB capacity reservation across multiple accounts is crucial for stock trading platforms to make sure of sufficient LCUs during peak trading hours. This automation eliminates manual overhead while maintaining performance and reliability. The solution focuses on using LCU reservation for two daily spikes at 9:00 AM and 4:00 PM. However, comprehensive preparation for high-load events should also include scaling load balancer targets and dependencies.</span></p> 
<h2>Solution overview</h2> 
<p>This solution enables you to centrally manages ALB’s provisioned capacity across multiple accounts in a large organization. Rather than configuring each ALB manually in the member accounts, the solution implements a centralized, automated approach.</p> 
<p>Two <a href="https://aws.amazon.com/pm/lambda/?trk=5cc83e4b-8a6e-4976-92ff-7a6198f2fe76&amp;sc_channel=ps&amp;ef_id=CjwKCAjw4efDBhATEiwAaDBpbqUzU2AllHK9hB3yy78SOGcRQv3Herv3ovtf-_EBqFa1aU6ENp9_pRoCFZQQAvD_BwE:G:s&amp;s_kwcid=AL!4422!3!651612776783!e!!g!!amazon%20lambda!19828229697!143940519541&amp;gad_campaignid=19828229697&amp;gclid=CjwKCAjw4efDBhATEiwAaDBpbqUzU2AllHK9hB3yy78SOGcRQv3Herv3ovtf-_EBqFa1aU6ENp9_pRoCFZQQAvD_BwE">AWS Lambda</a> functions are deployed in the Organization’s management account. These functions use <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html">AWS Security Token Service (AWS STS)</a> <a href="https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html">Assume Role</a> to access the ALBs running in member accounts, an <a href="https://aws.amazon.com/dynamodb/?trk=1e5631f8-a3e1-45eb-8587-22803d0da70e&amp;sc_channel=ps&amp;ef_id=CjwKCAjw4efDBhATEiwAaDBpbh8Fs2JsBwB5IZRBsGcDE8uGyIEracBrkIrFxk_TEdtvKH29wzwI4xoCd70QAvD_BwE:G:s&amp;s_kwcid=AL!4422!3!536393613268!e!!g!!amazon%20dynamodb!11539699824!109299643181&amp;gad_campaignid=11539699824&amp;gclid=CjwKCAjw4efDBhATEiwAaDBpbh8Fs2JsBwB5IZRBsGcDE8uGyIEracBrkIrFxk_TEdtvKH29wzwI4xoCd70QAvD_BwE">Amazon DynamoDB</a> table, and <a href="https://aws.amazon.com/iam/">AWS Identity and Access Management (IAM)</a> roles for the organization.</p> 
<p style="text-align: left;">The first Lambda function scans ALBs across member accounts and stores ALB information in a DynamoDB table within the management account, while the second function sets and cancels LCU reservations for the ALBs. These functions are triggered on a schedule by <a href="https://aws.amazon.com/eventbridge/scheduler/">Amazon EventBridge Schedulers.</a></p> 
<p>To identify ALBs that need LCU capacity provisioning, they must be tagged with:</p> 
<ul> 
 <li>Key = ALB-LCU-R-SCHEDULE</li> 
 <li>Value = Yes</li> 
</ul> 
<p>To set the LCU value, ALBs must be tagged with:</p> 
<ul> 
 <li>Key = LCU-SET</li> 
 <li>Value = &lt;LCU value&gt;</li> 
</ul> 
<h2>The process operates in three stages</h2> 
<ol> 
 <li>A few hours before the event starts, EventBridge Scheduler invokes a Lambda function that scans for ALBs tagged with “ALB-LCU-R-SCHEDULE = Yes” across all member accounts. The function collects key information from the ALBs (such as <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html">Amazon Resource Names (ARNs),</a> names, account IDs, and LCU value set as TAG) and stores this data in a DynamoDB table within the management account.</li> 
 <li>One hour before the event starts, a second Lambda function activates by EventBridge Scheduler. This function retrieves ALB information from the DynamoDB table and gradually adjusts the capacity on these ALBs. The capacity is configured based on the “LCU-SET” tag value.</li> 
 <li>After stock market closes, another EventBridge Scheduler invokes same Lambda function from step two, resets the LCU capacity for all ALBs, optimizing costs during off-hours.</li> 
</ol> 
<p style="text-align: left;"><span style="font-size: 11.0pt;">This process significantly reduces operational overhead while making sure that ALBs are optimally configured for both performance and cost-efficiency. We have centralized the management and used AWS services such as Lambda, DynamoDB, and Organization to create a scalable solution capable of managing hundreds of ALBs across multiple accounts.</span></p> 
<p><img alt="Architecture &amp; implementation flow" class="aligncenter wp-image-32186" height="516" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/Solution-architecture-implementation-flow-1024x668.png" width="791" /></p> 
<p style="text-align: center;"><em>Figure 2: Solution architecture and implementation flow</em></p> 
<h2>End-to-end execution flow overview</h2> 
<p>The preceding architecture demonstrates the end-to-end execution flow. This can be split into two flows: First, the ALB metadata collection flow, where execution collects metadata information from ALB and stores it in a DynamoDB table. Second, the ALB update flow, which initiates before the event starts. This flow retrieves the ALB and LCU information from the DynamoDB table and sets the LCU of ALBs. When traffic normalizes, this flow triggers again and resets the LCU value.</p> 
<h3><strong>A. ALB metadata collection flow: </strong></h3> 
<p>When EventBridge Scheduler runs in the management account, it invokes the “MetadataCollectorFunction” Lambda function, which runs in the management account with logic to describe and collect ALBs deployed across member accounts with specific tags. The Lambda function “MetadataCollectorFunction” assumes an IAM STS role to describe the member account ALBs, collects their metadata, and stores this information in a DynamoDB table in the management account, as shown in the following figure.</p> 
<p><img alt="Execution flow diagram for ALB metadata collection" class="aligncenter wp-image-32192" height="331" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/Execution-flow-diagram-1024x461.png" width="735" /></p> 
<p style="text-align: center;"><em>Figure 3: Execution flow diagram for ALB metadata collection</em></p> 
<h3><strong>B. ALB update flow:</strong></h3> 
<p>When EventBridge Scheduler runs in the management account, it invokes the “LCUModificationFunction” Lambda function, which runs in the management account with logic to set the Provisioned Capacity for ALBs deployed across member accounts with specific tags. The Lambda function “LCUModificationFunction” extracts the ALB metadata information from the DynamoDB table, then assumes an IAM STS role to update the ALBs with LCU values or Reset the LCU, as shown in the following figure.</p> 
<p><img alt="Execution flow diagram for updating ALB LCU values" class="aligncenter wp-image-32195" height="332" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/Execution-flow-diagram-2-1024x463.png" width="734" /></p> 
<p align="center" style="text-align: center;"><em><span style="font-size: 11.0pt;">Figure 4: Execution flow diagram for updating ALB LCU values</span></em></p> 
<h2 style="text-align: left;">Lambda function cross-account access in AWS Organizations</h2> 
<p style="text-align: left;"><span style="font-size: 11.0pt;">The Lambda function in the management account uses AWS STS to assume roles in member accounts. This is made possible through a trust relationship that allows the management account’s Lambda execution role to assume the cross-account role. Each member account contains an IAM role with permissions to modify ALB attributes and maintains a trust relationship with the Lambda execution role in the management account.</span></p> 
<p><img alt="Cross-account trust relationship architecture for centralized ALB management" class="aligncenter wp-image-32197" height="433" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/Cross-account-trust-relationship-architecture-1024x603.png" width="735" /></p> 
<p align="center" style="text-align: center;"><em><span style="font-size: 11.0pt;">Figure 5: Cross-account trust relationship architecture for centralized ALB management</span></em></p> 
<h2>Prerequisites</h2> 
<h3><strong>Management account requirements</strong></h3> 
<ol> 
 <li>Organizations must <a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_tutorials_basic.html">set up and configure</a> all member accounts with proper enrollment.</li> 
 <li>Management accounts should not have any IAM roles that conflict with the name of the role being deployed.</li> 
</ol> 
<h3><strong>Member account requirements</strong></h3> 
<ol> 
 <li>All member accounts must be registered and active with Organizations.</li> 
 <li>Member accounts should not have any IAM roles that conflict with the name of the role being deployed.</li> 
 <li>Add the following tags to the ALB for which you want to provision capacity. For instructions on how to add tags, refer to refer to <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-tags.html">“Tag an Application Load Balancer”</a> in the documentation.</li> 
</ol> 
<table class=" aligncenter" style="background-color: #fafafa; height: 238px;" width="899"> 
 <tbody> 
  <tr> 
   <td style="text-align: left;"><strong>Key</strong></td> 
   <td style="width: 100px; text-align: left;"><strong>Value</strong></td> 
   <td style="text-align: left;"><strong>Description</strong></td> 
  </tr> 
  <tr> 
   <td style="width: 200px;">ALB-LCU-R-SCHEDULE</td> 
   <td style="width: 40px; text-align: left;">Yes</td> 
   <td>Value Yes indicates that this ALB is subject to be prewarming</td> 
  </tr> 
  <tr> 
   <td>LCU-SET</td> 
   <td>100</td> 
   <td>This TAG indicates the LCU value to be set for this ALB. The minimum LCU value is 100, set the value according to your needs.<p></p> <p style="text-align: left;">For guidance on calculating the LCU Reservation, refer to the post <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/using-load-balancer-capacity-unit-reservation-to-prepare-for-sharp-increases-in-traffic/">Using Load Balancer Capacity Unit Reservation to prepare for sharp increases in traffic</a></p> </td> 
  </tr> 
 </tbody> 
</table> 
<h3>Solution implementation using AWS CloudFormation template</h3> 
<p>This solution is comprised of two CloudFormation templates, one for the management account and the other for member accounts. To implement this solution, you need to download the CloudFormation templates from the following GitHub link. After obtaining the templates, follow the instructions to deploy them in your AWS environment.</p> 
<p>The solution is available in the following GitHub repository: <a href="https://github.com/aws/elastic-load-balancing-tools/tree/master/automating-aws-application-load-balancer-capacity-reservation">GitHub repo link for AWS ALB Capacity Reservation Automation solution </a></p> 
<ul> 
 <li><strong>Management Account: </strong>ALBCapacityAutomationMgmtAccount.yaml</li> 
 <li><strong>Member Account(s): </strong>ALBCapacityAutomationMemberAccount.yaml</li> 
</ul> 
<h2>Deployment instructions</h2> 
<p>A. To create resources using the CloudFormation template in the management account, follow these steps:</p> 
<ol> 
 <li> 
  <ol> 
   <li>Sign in to the <a href="https://aws.amazon.com/console/">AWS Management Console</a>.</li> 
   <li>Navigate to the CloudFormation console &gt; <strong>Create Stack</strong> &gt; <strong>With new resources</strong>.</li> 
   <li>Upload the YAML template file and choose</li> 
  </ol> </li> 
</ol> 
<p><img alt="Uploading YAML template" class="aligncenter wp-image-32205" height="351" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/Uploading-YAML-template-file-for-stack-creation-1024x496.png" width="724" /></p> 
<p align="center" style="text-align: center;"><span style="font-size: 11.0pt;"><em>Figure 6.a: Uploading YAML template file for stack creation</em> </span></p> 
<ol> 
 <li> 
  <ol> 
   <li>Enter a <strong>Stack name</strong>, review the parameters, and choose</li> 
  </ol> </li> 
</ol> 
<p style="padding-left: 80px;">When deploying the Management account template, you can modify the schedule time and day for each one of the three events. Refer <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-scheduled-rule-pattern.html">the cron syntax in the EventBridge documentation.</a></p> 
<p><img alt="CloudFormation stack configuration screen with pre-filled parameters" class="aligncenter wp-image-32211" height="371" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/CloudFormation-stack-configuration-1024x517.png" width="735" /></p> 
<p style="text-align: center;"><em>Figure 6.b: CloudFormation stack configuration screen with pre-filled parameters</em></p> 
<ol> 
 <li>Keep the <strong>Configure stack options</strong> at their default values and choose</li> 
 <li>Review the details on the final screen and under <strong>Capabilities,</strong> check the box for <strong>I acknowledge that AWS </strong><strong>CloudFormation</strong><strong> might create IAM resources with custom names</strong>.</li> 
 <li>Choose</li> 
</ol> 
<p>B. To create resources using the CloudFormation template in member accounts, follow these steps:</p> 
<ol> 
 <li> 
  <ol> 
   <li>Sign in to the Console.</li> 
   <li>Navigate to the CloudFormation console &gt; <strong>Create Stack</strong> &gt; <strong>With new resources.</strong></li> 
   <li>Upload the YAML template file and choose <strong>Next</strong>.</li> 
  </ol> </li> 
</ol> 
<p><img alt="Uploading YAML template file for stack creation" class="aligncenter wp-image-32213" height="356" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/Uploading-YAML-template-1024x495.png" width="736" /></p> 
<p align="center" style="text-align: center;"><span style="font-size: 11.0pt;">Figure 7.a: Uploading YAML template file for stack creation </span></p> 
<ol> 
 <li> 
  <ol> 
   <li>Enter a <strong>Stack name</strong>, review the parameters, and choose <strong>Next</strong>.</li> 
   <li>When deploying the Management account template, you can modify the schedule time and day for each one of the three events. Refer <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-scheduled-rule-pattern.html">the cron syntax in the EventBridge documentation.</a></li> 
  </ol> </li> 
</ol> 
<p><img alt="CloudFormation stack configuration screen with pre-filled parameters" class="aligncenter wp-image-32214" height="249" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/CloudFormation-stack-configuration-2-1024x347.png" width="735" /></p> 
<p style="text-align: center;">Figure 7.b: CloudFormation stack configuration screen with pre-filled parameters</p> 
<ol> 
 <li>Keep the <strong>Configure stack options</strong> at their default values and choose <strong>Next</strong>.</li> 
 <li>Review the details on the final screen and under <strong>Capabilities</strong>, choose the checkbox <strong>I acknowledge that AWS </strong><strong>CloudFormation</strong><strong> might create IAM resources with custom names.</strong></li> 
 <li>Choose</li> 
</ol> 
<h4>Upon successful deployment, the CloudFormation templates create the following resources:</h4> 
<ul> 
 <li>In the Management account: 
  <ul> 
   <li>Two Lambda Functions</li> 
   <li>One DynamoDB table</li> 
   <li>Three EventBridge Schedulers</li> 
   <li>Two IAM roles in the Management account</li> 
  </ul> </li> 
 <li>In the Member accounts: 
  <ul> 
   <li>Two IAM roles with trust policies</li> 
  </ul> </li> 
</ul> 
<h2>Invoking Lambda functions with EventBridge Scheduler</h2> 
<table style="height: 134px; border-color: #050000; background-color: #fafafa;" width="794"> 
 <tbody> 
  <tr> 
   <td style="width: 36px;" width="36"> <p style="text-align: center;"><strong>#</strong></p> </td> 
   <td style="width: 200px; height: 30px; text-align: left;" width="234"><strong>EventBridge Scheduler</strong></td> 
   <td style="width: 342px; height: 30px;" width="342"> <p style="text-align: left;"><strong>Lambda function</strong></p> </td> 
  </tr> 
  <tr> 
   <td style="width: 36px;" width="36"> <p style="text-align: center;">1</p> </td> 
   <td style="width: 200px;" width="234"> <p style="text-align: left;">ALBautomation-MetedataCollector</p> </td> 
   <td width="342"> <p style="text-align: left;">ALB-CapacityAutomation-MetadataCollector-Lambda</p> </td> 
  </tr> 
  <tr> 
   <td width="36"> <p style="text-align: center;">2</p> </td> 
   <td style="width: 200px;" width="234"> <p style="text-align: left;">ALBautomation-LCUModification</p> </td> 
   <td width="342"> <p style="text-align: left;">ALB-CapacityAutomation-LCUModification-Lambda</p> </td> 
  </tr> 
  <tr> 
   <td width="36"> <p style="text-align: center;">3</p> </td> 
   <td style="width: 200px;" width="234"> <p style="text-align: left;">ALBautomation-LCUReset</p> </td> 
   <td width="342"> <p style="text-align: left;">ALB-CapacityAutomation-LCUModification-Lambda</p> </td> 
  </tr> 
 </tbody> 
</table> 
<h2>Clean up</h2> 
<p>To clean up AWS resources deployed through CloudFormation, delete the associated CloudFormation stack using the CloudFormation console.</p> 
<ol> 
 <li>Navigate to the CloudFormation console: Sign in to the Console and open the CloudFormation console.</li> 
 <li>Choose the stack: Choose the stack you want to delete from the list of stacks.</li> 
 <li>Initiate deletion: Choose <strong>Delete</strong>.</li> 
 <li>Confirm deletion: A confirmation window may appear listing resources that might fail to delete. Choose any resources that you want to retain (if applicable), then choose <strong>Delete stack</strong>.</li> 
 <li>Monitor Progress: Watch the stack’s status in the console as it transitions to DELETE_COMPLETE.</li> 
</ol> 
<h2>Troubleshooting</h2> 
<p>The best way to troubleshoot the Lambda functions is by examining their execution logs. The solution provides detailed logging for each step. Refer to the documentation <a href="https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-view.html#monitoring-cloudwatchlogs-console">Access function logs using the console</a> for step-by-step instructions on viewing the logs.</p> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated an automated solution that efficiently manages ALB provisioned capacity across multiple load balancers within an AWS Organization. The solution’s core components are AWS Lambda functions working in conjunction with Amazon EventBridge Scheduler, creating a system that intelligently handles LCU reservations through automated probing, commitment, and release processes.</p> 
<p>This approach is especially beneficial for businesses experiencing predictable traffic spikes because it makes sure that their infrastructure is prepared for high-demand periods. The automation streamlines capacity management while optimizing costs by increasing capacity when needed and scaling down during quieter periods. Organizations can now move away from manual methods, reduce their operational workload, and implement a more efficient approach to handling LCU reservations across their AWS accounts and applications.</p> 
<h2>About the authors</h2> 
<p>
 <!-- First Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Abhishek Dey.jpg"><img alt="Abhishek Dey" class="alignleft wp-image-1288 size-thumbnail" src="//d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/abhdey.jpg" style="width: 125px; height: 145px;" /></p> 
 <h3 class="lb-h4">Abhishek Dey</h3> 
 <p style="color: #879196; font-size: 1.3rem;">Abhishek is a Senior Technical Account Manager for the Financial Services Industry at Amazon Web Services (AWS). He leads the Networking Field Community for Enterprise Support in India, where he helps customers build and design scalable, highly available, secure, resilient, and cost-effective networks. He is a lifelong learner and tech enthusiast with a strong focus on Networking, Data Engineering, Generative AI, and cutting-edge technologies.</p> 
</div> 
<p>
 <!-- Second Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="souravbb.png"><img alt="Sourav Bhattacharjee" class="alignleft wp-image-1288 size-thumbnail" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/13/souravbb_new.png" style="width: 125px; height: 140px;" /></p> 
 <h3 class="lb-h4">Sourav Bhattacharjee</h3> 
 <p style="color: #879196; font-size: 1.3rem;">Sourav is an Enterprise Support Manager at Amazon Web Services with over 20+ years of industry experience spanning consulting, advisory, and leadership. He leads the India FSI Enterprise Support customer segment, partnering with financial services organizations to drive strategic cloud adoption, foster executive relationships, and enable teams to build scalable, modern, and cost-effective solutions for customers.</p> 
</div>
