---
title: "Implementing fine-grained Amazon Route 53 access using IAM condition keys (Part 2)"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/implementing-fine-grained-amazon-route-53-access-using-iam-condition-keys-part-2/"
date: "Tue, 28 Apr 2026 23:22:39 +0000"
author: "Daniel Yu"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
<p>In <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/implementing-fine-grained-amazon-route-53-access-using-aws-iam-condition-keys-part-1/" rel="noopener noreferrer" target="_blank">Part 1</a> of this series, we demonstrated a scalable solution of using Amazon Web Services <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">Identity and Access Management (AWS IAM)</a> conditional keys and AWS principal tags for fine-grained access control of shared <a href="https://aws.amazon.com/route53/" rel="noopener noreferrer" target="_blank">Amazon Route 53</a> hosted zones, public or private, in the same AWS account. As user environments grow, AWS administrators and network engineers need to manage DNS permissions across AWS accounts for shared Route 53 hosted zones. This post guides you through a scalable solution to grant fined-grained access between AWS accounts with the Route 53 hosted zone and IAM users across different accounts.</p> 
<h2>Solution overview</h2> 
<p>We assume that you have familiarity with the solution from Part 1 that create fine-grained permissions based on user attributes to access Route 53 hosted zones in the same account using the four conditional operations (<code>ForAllValues:StringEquals</code>, <code>ForAllValues:StringNotEquals</code>, <code>ForAllValues:StringLike</code>, <code>ForAllValues:StringNotLike</code>). This solution combines the fine-grained permission with the functionality of assuming <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" rel="noopener noreferrer" target="_blank">IAM roles</a> between AWS accounts and setting session tags to allow fine-grained access between AWS accounts to shared hosted zones. The assume role between accounts creates a set of temporary credentials that you can use to access AWS resources between the accounts. The setting of session tags for the assume role session pass the IAM user tag key pairs for evaluation of fine-grained permissions. This solution streamlines access management between AWS accounts by allowing you to create fine-grained permissions based on user attributes and reducing the permissions to align with least-privilege principles.</p> 
<p>The following diagram shows the fine-grained Route 53 access architecture across AWS accounts. The diagram shows how IAM users (left) are assigned a custom tag key <code>dnsrecord</code> with the value of their permitted DNS updates in AWS account A. The IAM user assumes the role in account B and sets the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-requesttag" rel="noopener noreferrer" target="_blank">RequestTag</a> of the session with the tag key-value pairs of the IAM user custom tag key pair to modify the Route 53 hosted zone in account B (right). The IAM policy attached to the assume role (center) evaluates the <a href="https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonroute53.html#amazonroute53-actions-as-permissions" rel="noopener noreferrer" target="_blank"><code>route53:ChangeResourceRecordSets</code></a> action and compares the DNS record name update against the session tag value using the condition key <a href="https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonroute53.html#amazonroute53-policy-keys" rel="noopener noreferrer" target="_blank"><code>route53:ChangeResourceRecordSetsNormalizedRecordNames</code></a> to grant access.</p> 
<div class="wp-caption aligncenter" id="attachment_31835" style="width: 1722px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/17/route53-cross-account-architecture.png" rel="noopener" target="_blank"><img alt="The diagram shows how IAM users (left) are assigned a custom tag key “dnsrecord” with value of their permitted DNS updates in AWS account A. The IAM user assumes role in account B and set RequestTag of the session with the tag key-value pairs of IAM user custom tag key pair to modify the Route 53 hosted zone in account B (right). The IAM policy attached to the assume role (center) evaluates the route53:ChangeResourceRecordSets action and compares the DNS record name update against the session tag value using the condition key route53:ChangeResourceRecordSetsNormalizedRecordNames to grant access" class="wp-image-31835 size-full" height="623" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/17/route53-cross-account-architecture.png" width="1712" /></a>
 <p class="wp-caption-text" id="caption-attachment-31835">Figure 1: Fine-grained Route 53 access using IAM condition keys across AWS accounts (Click image to open larger version in new tab)</p>
</div> 
<p>The permission policy attached to the assume role uses conditional operations with the Route 53 condition key <code>route53:ChangeResourceRecordSetsNormalizedRecordNames</code> to control the DNS record names in the <code>route53:ChangeResourceRecordSets</code> action. The action matches the <code>${aws:PrincipalTag/{custom-attribute}}</code> key to grant permission to manage specific DNS records. The <code>${aws:PrincipalTag/{custom-attribute}}</code> condition key specifies the tag value for the assume role session of IAM user.</p> 
<p>A trust policy attached to a role defines the principals that you trust to assume the role. The following trust policy is attached to the assume role in account B to allow account A users to assume this role and must set the session tags to be the tag value of the IAM user requesting the assume role. In the policy document, replace <code>&lt;ACCOUNT&gt;</code> with the IAM user AWS account number and <code>&lt;custom-attribute&gt;</code> with the custom tag key for the DNS record.</p> 
<pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::&lt;ACCOUNT&gt;:root"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:RequestTag/&lt;custom-attribute&gt;": "${aws:PrincipalTag/&lt;custom-attribute&gt;}"
                }
            }
        }
    ]
}
</code></pre> 
<p>The IAM users need permission to assume the role and set the session tags. The following permission policy is attached to the IAM users in account A to allow assume role in account B and tag the session with the custom tag key value. In the policy document, <code>&lt;ROLE_ARN&gt;</code> needs to be replaced with the role <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener noreferrer" target="_blank">Amazon Resource Name (ARN)</a> in account B.</p> 
<pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": [
            "sts:AssumeRole",
            "sts:TagSession"
        ],
        "Resource": "&lt;ROLE_ARN&gt;"
    }
}
</code></pre> 
<h2>Solution implementation</h2> 
<p>In this post you create a solution that allows users in one AWS account to manage DNS records that match their assigned tag values in a shared hosted zone in another AWS account. Following these steps allows you to configure IAM user tags and assume role in a different account with policies that enable fine-grained access control to your Route 53 hosted zone. When completed, users can only manage DNS records that match the value in their custom tag of a shared hosted zone in another AWS account.</p> 
<p>The example uses the value in the <code>dnsrecord</code> custom attribute of the IAM user to grant conditional access to update the DNS records ending with the <code>.svc1.example.com</code> domain suffix in the shared hosted zone <code>example.com</code> used in a different account.</p> 
<h2>Prerequisites</h2> 
<p>Before proceeding, you need the following:</p> 
<ul> 
 <li>Log in to AWS account A and B with permissions to update IAM users, roles, and permissions using <a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI</a>)</li> 
 <li>IAM user in AWS account A</li> 
 <li>Hosted zone for domain <code>example.com</code> in AWS account B</li> 
</ul> 
<h3>Step 1: Create the role with trust policy in AWS account B</h3> 
<p>Create the following role trust policy document to allow role to be assumed and tagged with the same IAM user custom attribute key value from AWS account A. Replace <code>&lt;ACCOUNTA&gt;</code> with the AWS account A number.</p> 
<pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::&lt;ACCOUNTA&gt;:root"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:RequestTag/dnsrecord": "${aws:PrincipalTag/dnsrecord}"
                }
            }
        }
    ]
}
</code></pre> 
<p>Use the following command to create the role, replacing <code>&lt;ROLE_NAME&gt;</code> with your role and <code>&lt;TRUST_DOCUMENT_FILE&gt;</code> with the location of the preceding trust policy document:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam create-role --role-name &lt;ROLE_NAME&gt; --assume-role-policy-document &lt;TRUST_DOCUMENT_FILE&gt; --output text –-query Role.Arn</code></pre> 
</div> 
<h3>Step 2: Create the permission policy and attach to role in AWS account B</h3> 
<p>Create the following permission policy document for fine-grained access of DNS records. There is a wildcard <code>*</code> and period <code>.</code> in front of the <code>${aws:PrincipalTag/dnsrecord}</code> to enable the string match. The value of <code>&lt;ZONE_ID<em>&gt; </em></code>is the private hosted zone ID, and you must replace it for your hosted zone.</p> 
<pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "route53:ChangeResourceRecordSets",
            "Resource": "arn:aws:route53:::hostedzone/&lt;ZONCE_ID&gt;",
            "Condition": {
                "ForAllValues:StringLike": {
                    "route53:ChangeResourceRecordSetsNormalizedRecordNames": [
                        "*.${aws:PrincipalTag/dnsrecord}"
                    ]
                }
            }
        }
    ]
}
</code></pre> 
<p>You can use the following command to create policy and capture the policy ARN, replacing <code>&lt;POLICY_NAME&gt;</code> with your policy name and <code>&lt;POLICY_DOCUMENT_FILE&gt;</code> with the location of the preceding permission document:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam create-policy --policy-name &lt;POLICY_NAME&gt; --policy-document &lt;POLICY_DOCUMENT_FILE&gt; --output text --query Policy.Arn</code></pre> 
</div> 
<p>You can use the following command to attach the above policy to the role created in Step 1, replacing <code>&lt;ROLE_NAME&gt;</code> with the role name created in Step 1 and <code>&lt;POLICY_ARN&gt;</code> with the ARN of the preceding policy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam attach-role-policy --role-name &lt;ROLE_NAME&gt; --policy-arn &lt;POLICY_ARN&gt;</code></pre> 
</div> 
<h3>Step 3: Create principal tag in AWS account A</h3> 
<p>You can use the following command to create the <code>dnsrecord</code> key with the <code>svc1.example.com</code> value, replacing <code>&lt;USER_NAME&gt;</code> with Account A IAM user:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam tag-user --user-name &lt;USER_NAME&gt; --tags '{"Key": "dnsrecord", "Value": "svc1.example.com"}'</code></pre> 
</div> 
<h3>Step 4: Create the permission policy to assume role and associate to IAM user in AWS account A</h3> 
<p>Create the following permission policy document to allow assume role in account B and tag session, replacing <code>&lt;ROLE_ARN&gt;</code> with the ARN of the role created in Step 1.</p> 
<pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": [
            "sts:AssumeRole",
            "sts:TagSession"
        ],
        "Resource": "&lt;ROLE_ARN&gt;"
    }
}
</code></pre> 
<p>You can use the following command to create the permission policy and capture the policy ARN, replacing <code>&lt;POLICY_NAME&gt;</code> with name for policy and <code>&lt;POLICY_DOCUMENT_FILE&gt;</code> with the location of the preceding policy document:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam create-policy --policy-name &lt;POLICY_NAME&gt; --policy-&lt;POLICY_DOCUMENT_FILE&gt; --output text --query Policy.Arn</code></pre> 
</div> 
<p>You can use the following command to attach the preceding policy to the IAM user, replacing <code>&lt;POLICY_ARN&gt;</code> with the preceding policy ARN and <code>&lt;USER_NAME&gt;</code> of the user:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam attach-user-policy --policy-arn &lt;POLICY_ARN&gt; --user-name &lt;USER_NAME&gt;</code></pre> 
</div> 
<h3>Step 5: Verified DNS update permissions from AWS account A to account B</h3> 
<p>Log in to the IAM user in account A and use the following command to assume role in account B to provide temporary security credentials and tag the session with the custom tag key pair value of IAM user. Replace <code>&lt;ROLE_ARN&gt;</code> with the ARN of the role created in Step 1 and <code>&lt;SESSION_NAME&gt;</code> with name for this assumed role session. The <code>AWS_ACCESS_KEY_ID</code>, <code>AWS_SECRET_ACCESS_KEY</code> and <code>AWS_SESSION_TOKEN</code> values from the assume role session are updated with temporary security credentials for your session:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s" \
$(aws sts assume-role \
--role-arn &lt;ROLE_ARN&gt; \
--role-session-name &lt;SESSION_NAME&gt; \
--tags Key=dnsrecord,Value=svc1.example.com \
--query "Credentials.[AccessKeyId,SecretAccessKey,SessionToken]" \
--output text))
</code></pre> 
</div> 
<p>The IAM user in account A through the assume role in account B now has permission to managed DNS records in the <code>.svc1.example.com</code> domain suffix in the shared hosted zone <code>example.com</code>. You can verify the permission is working as expected by using the following commands.</p> 
<p>For example, using the following CLI command with the JSON configuration file to create the DNS record <code>dev.svc1.example.com</code> succeeds.</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws route53 change-resource-record-sets --hosted-zone-id &lt;Z1R8UBAEXAMPLE&gt; --change-batch file://svc1-create.json</code></pre> 
</div> 
<pre><code class="lang-json">{
    "Comment": "configuration to create dev.svc1.example.com record",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "dev.svc1.example.com",
                "Type": "A",
                "TTL": 300,
                "ResourceRecords": [
                    {
                        "Value": "10.1.1.1"
                    }
                ]
            }
        }
    ]
}
</code></pre> 
<p>Using the following CLI command with the JSON configuration file to create the DNS record <code>dev.svc2.example.com</code> failed.</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws route53 change-resource-record-sets --hosted-zone-id &lt;Z1R8UBAEXAMPLE&gt; --change-batch file://svc2-create.json</code></pre> 
</div> 
<pre><code class="lang-json">{
    "Comment": "configuration to create dev.svc2.example.com record",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "dev.svc2.example.com",
                "Type": "A",
                "TTL": 300,
                "ResourceRecords": [
                    {
                        "Value": "10.1.1.1"
                    }
               ]
            }
        }
    ]
}
</code></pre> 
<h2>Cleaning up</h2> 
<p>Clean up the resources created in the solution by using the following commands:</p> 
<p>Disassociate assume role permission policy from IAM user in Account A, replacing <code>&lt;USER_NAME&gt;</code> with the IAM user in Account A and <code>&lt;POLICY_ARN&gt;</code> with the ARN created in Step 4:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam detach-user-policy --user-name &lt;USER_NAME&gt; --policy-arn &lt;POLICY_ARN&gt;</code></pre> 
</div> 
<p>Delete the assume role permission policy in Account A, replacing <code>&lt;POLICY_ARN&gt;</code> with the policy ARN created in Step 4:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam delete-policy --policy-arn &lt;POLICY_ARN&gt;</code></pre> 
</div> 
<p>Delete the principal tag from the IAM user in Account A, replacing <code>&lt;USER_NAME&gt;</code> with the Account A IAM user:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam untag-user --user-name &lt;USER_NAME&gt; --tag-keys dnsrecord</code></pre> 
</div> 
<p>Detach the permission policy from the role in Account B, replacing <code>&lt;ROLE_NAME&gt;</code> with the role name created in Step 1 and <code>&lt;POLICY_ARN&gt;</code> with the ARN created in Step 2:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam detach-role-policy --role-name &lt;ROLE_NAME&gt; --policy-arn &lt;POLICY_ARN&gt;</code></pre> 
</div> 
<p>Delete the permission policy in Account B, replacing <code>&lt;POLICY_ARN&gt;</code> with the ARN created in Step 2:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam delete-policy --policy-arn &lt;POLICY_ARN&gt;</code></pre> 
</div> 
<p>Delete the role in Account B, replacing <code>&lt;ROLE_NAME&gt;</code> with the role name created in Step 1:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">aws iam delete-role --role-name &lt;ROLE_NAME&gt;</code></pre> 
</div> 
<h2>Considerations</h2> 
<p>Furthermore, not only can you grant users in another AWS account the ability to assume role, but also there are <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html" rel="noopener noreferrer" target="_blank">condition context</a>s available to control how the principals are trusted to assume a role.</p> 
<p>As you manage your IAM access, consider the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html" rel="noopener noreferrer" target="_blank">IAM quotas and character limitations</a> for users, groups, and policies.</p> 
<p>If your permissions are not working as expected, then the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_policies.html" rel="noopener noreferrer" target="_blank">troubleshoot IAM Policies</a> documentation includes useful guides for common issues.</p> 
<p>For environments not using shared hosted zones, where each team manages private hosted zones for their services, <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/profiles.html" rel="noopener noreferrer" target="_blank">Route 53 Profiles</a> is the recommended best practice for centralized management of DNS configurations for <a href="https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html" rel="noopener noreferrer" target="_blank">Amazon Virtual Private Clouds (Amazon VPCs</a>) and accounts. The post <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/using-amazon-route-53-profiles-for-scalable-multi-account-aws-environments/" rel="noopener noreferrer" target="_blank">Using Amazon Route 53 Profiles for scalable multi-account AWS environments</a> reviews the architecture.</p> 
<h2>Conclusion</h2> 
<p>In this post we reviewed a scalable solution of using IAM conditional keys and AWS principal tags for fine-grained access control of shared Amazon Route 53 hosted zones to managed subsets of DNS records between AWS accounts. The review included an example of implementing conditional access of a hosted zone between accounts. In the next post, we review federated user fine-grain access with IAM conditional keys and AWS principal tags.</p> 
<h2>About the author</h2> 
<p>
 <!-- First Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="dyuamzn-blog-author.jpeg"><img alt="" class="alignleft wp-image-28571 size-full" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/07/02/dyuamzn-blog-author.jpeg" width="125" /></p> 
 <h3 class="lb-h4">Daniel Yu</h3> 
 <p style="color: #879196; font-size: 1rem;">Daniel Yu is a Senior Technical Account Manager who partners with users to optimize their cloud transformation initiatives. With expertise in networking and security infrastructure, he specializes in delivering strategic guidance on AWS architectural design and operational excellence.</p> 
</div>
