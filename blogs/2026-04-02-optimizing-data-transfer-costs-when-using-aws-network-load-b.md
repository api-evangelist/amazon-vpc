---
title: "Optimizing data transfer costs when using AWS Network Load Balancer"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/optimizing-data-transfer-costs-when-using-aws-network-load-balancer/"
date: "Thu, 02 Apr 2026 22:23:15 +0000"
author: "Luis Felipe Silveira da Silva"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
<p>Following our previous post, <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/exploring-data-transfer-costs-for-aws-network-load-balancers/"><em>Exploring Data Transfer Costs for AWS Network Load Balancers</em></a>, this post explores architectural patterns to help optimize these expenses.</p> 
<h2>Understanding inter-zone data transfer costs</h2> 
<p>When network traffic flows across Amazon Web Services (AWS) <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html">Availability Zones (AZs)</a>, whether from clients to <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/network/">Network Load Balancers (NLBs)</a> or from NLBs to targets, AWS applies an <a href="https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer">inter-zone data transfer</a> charge of $0.01 per GB in each direction. This charge applies to both the load balancer and <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html">Amazon Elastic Compute Cloud (Amazon EC2)</a> instances involved in the transfer. For example, when sending 1 GB of data from an EC2 client instance to an NLB in a different AZ, data transfer charges are incurred at both ends of the connection: the client and the NLB.</p> 
<p>To better understand this pricing structure, consider the following scenario, shown in Figure 1:</p> 
<ul> 
 <li>Data transfer <a href="https://aws.amazon.com/ec2/pricing/on-demand/">between AZs</a> costs $0.02 per GB.</li> 
 <li>The traffic flows from a client in AZ A to an NLB in&nbsp;AZ B, the charge breaks down as: 
  <ul> 
   <li>$0.01 per GB at the sender’s end</li> 
   <li>$0.01 at the receiver’s end</li> 
  </ul> </li> 
</ul> 
<p>Data transfer between an NLB and its target within the same AZ incurs no other charges.</p> 
<div class="wp-caption aligncenter" id="attachment_31889" style="width: 718px;">
 <img alt="Figure 1: Client is in a different AZ, NLB and target are in the same AZ" class="wp-image-31889 size-full" height="710" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/20/Picture1-4.png" width="708" />
 <p class="wp-caption-text" id="caption-attachment-31889"><strong>Figure 1: Client is in a different AZ, NLB and target are in the same AZ</strong></p>
</div> 
<ol> 
 <li>When data crosses between AZs, a transfer cost of $0.02 per GB is incurred.</li> 
 <li>When traffic crosses AZ boundaries twice—once from client to NLB and once from NLB to target—the total data transfer cost amounts to $0.04 per GB in each direction. This occurs when the client is in AZ A, the NLB in AZ B, and the target back in AZ A, as shown in Figure 2.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31890" style="width: 761px;">
 <img alt="Figure 2: Client and NLB are in different AZs, and the Target is not in the same AZ as the NLB" class="wp-image-31890 size-full" height="751" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/20/Picture2-3.png" width="751" />
 <p class="wp-caption-text" id="caption-attachment-31890"><strong>Figure 2: Client and NLB are in different AZs, and the Target is not in the same AZ as the NLB</strong></p>
</div> 
<ol start="3"> 
 <li>If the traffic flow remains within the same AZ, then there is no charge for data transfer, as shown in Figure 3:</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31892" style="width: 761px;">
 <img alt="Figure 3: The Client, NLB, and Target are all in the same AZ" class="wp-image-31892 size-full" height="755" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/20/Picture3-3.png" width="751" />
 <p class="wp-caption-text" id="caption-attachment-31892"><strong>Figure 3: The Client, NLB, and Target are all in the same AZ</strong></p>
</div> 
<p>Now that we have discussed these scenarios, let’s examine how traffic flows between AZs and explore the features that help optimize and reduce associated costs.</p> 
<h2>Client to NLB traffic</h2> 
<p>In the preceding examples, we discussed an internal NLB being accessed by an internal client within the same <a href="https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html">Amazon Virtual Private Cloud (Amazon VPC)</a>. However, the same principles apply to clients accessing the internal NLB through an <a href="https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html">AWS Transit Gateway</a> or <a href="https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html">VPC peering</a> connection.</p> 
<p>By default, NLBs have an Availability Zone DNS affinity setting of 0 percent zonal affinity. This means that when clients make DNS queries using <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html">Amazon Route 53 resolver</a>, they receive responses containing any healthy NLB IP addresses across all AZs where the NLB is present. Although this approach typically achieves better traffic distribution, it also means that clients may send traffic to any NLB AZ with equal probability, regardless of their own location. When clients send traffic to an NLB Elastic Network Interface (ENI) in a different AZ, data transfer charges apply.</p> 
<p>To minimize inter-AZ data transfer costs, you can enable 100% zonal affinity, which keeps client-to-NLB traffic within the same AZ. However, if no healthy NLB IP addresses are available in the client’s AZ, then DNS queries may still route traffic to other zones.</p> 
<h4>Considerations</h4> 
<p>Enabling 100% zonal affinity can lead to uneven traffic distribution at the NLB level. This imbalance may extend to the target level when cross-zone load balancing is disabled. To address this situation:</p> 
<ul> 
 <li>Aim to maintain balanced client traffic distribution across all of the healthy IPs addresses and AZs of your NLBs. 
  <ul> 
   <li>This can be achieved most effectively by following DNS and TCP connectivity best practices.</li> 
  </ul> </li> 
 <li>If an even distribution cannot be achieved, then adjust the target capacity in each zone to match expected traffic patterns.</li> 
 <li>The following AWS services can help manage your capacity: 
  <ul> 
   <li><a href="https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html">Amazon EC2 Auto Scaling</a></li> 
   <li><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html">Amazon Elastic Container Service (Amazon ECS)</a></li> 
   <li><a href="https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html">Amazon Elastic Kubernetes Service (Amazon EKS)</a></li> 
   <li><a href="https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/Welcome.html">AWS Elastic Beanstalk</a></li> 
  </ul> </li> 
</ul> 
<h4>How to turn on zonal affinity</h4> 
<p>To turn on AZ affinity using the <a href="https://console.aws.amazon.com/ec2/">AWS Management Console</a>, follow these steps, also shown in Figure 4:</p> 
<ul> 
 <li>Open the <a href="https://console.aws.amazon.com/ec2/">Amazon EC2 console</a>.</li> 
 <li>In the navigation pane, choose <strong>Load Balancers</strong>.</li> 
 <li>Choose the name of the NLB to open its details page.</li> 
 <li>On the <strong>Attributes</strong> tab, choose <strong>Edit</strong>.</li> 
 <li>Under <strong>Availability Zone routing configuration</strong>, <strong>Client routing policy (DNS record)</strong>, choose <strong>Availability Zone affinity</strong> or <strong>Partial Availability Zone affinity</strong>.</li> 
 <li>Choose <strong>Save changes</strong>.</li> 
</ul> 
<div class="wp-caption aligncenter" id="attachment_31894" style="width: 800px;">
 <img alt="Figure 4: Setting AZ affinity for NLB" class="wp-image-31894 size-full" height="392" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/20/Picture4-2.png" width="790" />
 <p class="wp-caption-text" id="caption-attachment-31894"><strong>Figure 4: Setting AZ affinity for NLB</strong></p>
</div> 
<p>To turn on AZ affinity using the <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html">AWS Command Line Interface (AWS CLI)</a>:</p> 
<ul> 
 <li>Use the <a href="https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-load-balancer-attributes.html"><code>modify-load-balancer-attributes</code></a> command with the <code>dns_record.client_routing_policy</code> attribute.</li> 
</ul> 
<h2>NLB to target traffic</h2> 
<p>Cross-zone traffic between a NLB and its targets is controlled by cross-zone load balancing configurations at both the load balancer and target group levels. When cross-zone load balancing is enabled, each load balancer node can distribute incoming traffic across all registered targets in every enabled AZ. However, when a load balancer node routes traffic to a target in a different AZ, it results in inter-zone traffic that incurs charges.</p> 
<p>By default, NLBs have cross-zone load balancing disabled at both the load balancer and target group levels. Although some users enable it to achieve more balanced traffic distribution across their targets, this decision results in inter-zone data transfer charges whenever traffic is sent to targets in different AZs.</p> 
<h4>Considerations</h4> 
<p>When cross-zone load balancing is disabled, traffic distribution among targets can become imbalanced. It’s recommended to maintain a proportional number of targets in each AZ. This configuration helps achieve consistent and efficient traffic distribution across your application infrastructure.</p> 
<h4>How to turn off cross-zone load balancing</h4> 
<p>To modify cross-zone load balancing for a load balancer using the console, follow these steps, also shown in Figure 5:</p> 
<ul> 
 <li>Open the Amazon EC2 console.</li> 
 <li>In the navigation pane, under <strong>Load Balancing</strong>, choose <strong>Load Balancers</strong>.</li> 
 <li>Choose the name of the load balancer to open its details page.</li> 
 <li>On the <strong>Attributes</strong> tab, choose <strong>Edit</strong>.</li> 
 <li>On the <strong>Edit load balancer attributes</strong> page, turn <strong>Cross-zone load balancing</strong> off.</li> 
 <li>Choose <strong>Save changes</strong>.</li> 
</ul> 
<div class="wp-caption aligncenter" id="attachment_31897" style="width: 607px;">
 <img alt="Figure 5: Disabling cross-zone load balancing for NLB" class="wp-image-31897 size-full" height="127" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/20/Picture5-2.png" width="597" />
 <p class="wp-caption-text" id="caption-attachment-31897"><strong>Figure 5: Disabling cross-zone load balancing for NLB</strong></p>
</div> 
<p>To modify cross-zone load balancing for your load balancer using the AWS CLI:</p> 
<ul> 
 <li>Use the <a href="https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-load-balancer-attributes.html"><code>modify-load-balancer-attributes</code></a> command with the <code>load_balancing.cross_zone.enabled</code> attribute.</li> 
</ul> 
<p>To modify cross-zone load balancing for a target group using the console, follow these steps, also shown in Figure 6:</p> 
<ul> 
 <li>Open the Amazon EC2 console.</li> 
 <li>On the navigation pane, under <strong>Load Balancing</strong>, choose <strong>Target Groups</strong>.</li> 
 <li>Choose the name of the target group to open its details page.</li> 
 <li>On the <strong>Attributes</strong> tab, choose <strong>Edit</strong>.</li> 
 <li>On the <strong>Edit target group attributes</strong> page, choose <strong>On</strong> for <strong>Cross-zone load balancing</strong>.</li> 
 <li>Choose <strong>Save changes</strong>.</li> 
</ul> 
<div class="wp-caption aligncenter" id="attachment_31896" style="width: 609px;">
 <img alt="Figure 6: Disabling cross-zone load balancing for NLB Target Group" class="wp-image-31896 size-full" height="210" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/20/Picture6-1.png" width="599" />
 <p class="wp-caption-text" id="caption-attachment-31896">Figure 6: Disabling cross-zone load balancing for NLB Target Group</p>
</div> 
<p>To modify cross-zone load balancing for a target group using the AWS CLI:</p> 
<ul> 
 <li>Use the <a href="https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-target-group-attributes.html"><code>modify-target-group-attributes</code></a> command with the <code>load_balancing.cross_zone.enabled</code> attribute.</li> 
</ul> 
<h2>Availability Zone Independence</h2> 
<p>Another noteworthy aspect of this approach is the ability to achieve Availability Zone Independence (AZI) by disabling cross-zone load balancing and enabling 100% zonal affinity, as shown in Figure 7.</p> 
<p>This configuration not only helps mitigate data transfer costs but also serves as a best practice for building resilient architectures. Implementing AZI supports zonal evacuation strategies and can enhance availability in scenarios where a specific AZ experiences any issues. For more detailed information, refer to the <a href="https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/availability-zones.html">Availability Zone Independence whitepaper</a>.</p> 
<p>Furthermore, by keeping traffic within the same AZ, you can reduce packet latency through your load balancer. You can monitor real-time inter-AZ latency using the <a href="https://docs.aws.amazon.com/network-manager/latest/infrastructure-performance/what-is-infrastructure-performance.html">Infrastructure Performance</a> feature in <a href="https://docs.aws.amazon.com/network-manager/latest/cloudwan/what-is-network-manager.html">AWS Network Manager</a>.</p> 
<div class="wp-caption aligncenter" id="attachment_31899" style="width: 772px;">
 <img alt="Figure 7: AZI with NLB" class="wp-image-31899 size-full" height="852" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/02/20/Picture7-1.png" width="762" />
 <p class="wp-caption-text" id="caption-attachment-31899"><strong>Figure 7: AZI with NLB</strong></p>
</div> 
<h2>Conclusion</h2> 
<p>Optimizing data transfer costs for NLBs necessitates a thoughtful approach to architecture and configuration. You can implement the strategies discussed in this article to significantly reduce your inter-zone data transfer costs while maintaining high availability and performance. These strategies include the following:</p> 
<ul> 
 <li>Enabling 100% zonal affinity for client-to-NLB traffic</li> 
 <li>Disabling cross-zone load balancing</li> 
 <li>Maintaining balanced distribution of clients and targets across AZs</li> 
 <li>Consider&nbsp;Availability Zone independence</li> 
</ul> 
<p>Although these optimizations can lead to substantial cost savings, it’s crucial to monitor your application’s performance and adjust your strategy as needed. Remember that the most cost-effective solution should always be balanced against your specific requirements for availability, scalability, and performance. You can carefully consider your architecture and use these AWS features to create a more cost-efficient and resilient infrastructure that meets your business requirements. For more information on Network Load Balancer configuration and best practices, refer to the <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/network/">Network Load Balancer documentation</a>.</p> 
<h2>About the authors</h2> 
<p>
 <!-- First Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="silluis-2024.jpg"><img alt="Luis Felipe Silveira da Silva" class="alignleft size-full wp-image-5363" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/02/12/silluis-2024.jpg" width="125" /></p> 
 <h3 class="lb-h4">Luis Felipe Silveira da Silva</h3> 
 <p style="color: #879196; font-size: 1rem;">Luis Felipe is a Principal Solutions Architect. He is part of the Application Networking team and based in Dublin, Ireland. Leveraging AWS’s extensive load balancing and networking suite, he helps customers to design and enhance their workloads. Luis Felipe collaborates closely with clients and internal AWS teams to develop resilient architectures and promote the effective implementation of AWS networking solutions.</p> 
</div> 
<p>
 <!-- Second Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="rollucas-e1765429089188-150x150-1.jpg"><img alt="Lucas Rolim" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2025/12/11/rollucas-e1765429089188-150x150-1.jpg" width="125" /></p> 
 <h3 class="lb-h4">Lucas Rolim</h3> 
 <p style="color: #879196; font-size: 1rem;">Lucas Rolim, a Senior Solutions Architect at Amazon Web Services (AWS), is based in Sydney, Australia, where he works in the Application Networking team. He is dedicated to helping customers make informed decisions while building on AWS. His expertise primarily focuses on Networking and Security.</p> 
</div>
