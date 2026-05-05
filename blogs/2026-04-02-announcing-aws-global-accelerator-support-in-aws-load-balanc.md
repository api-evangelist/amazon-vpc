---
title: "Announcing AWS Global Accelerator Support in AWS Load Balancer Controller for Kubernetes"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/announcing-aws-global-accelerator-support-in-aws-load-balancer-controller-for-kubernetes/"
date: "Thu, 02 Apr 2026 20:43:55 +0000"
author: "Jaiganesh Girinathan"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
<p>We recently announced that the AWS Load Balancer Controller now supports <a href="https://aws.amazon.com/global-accelerator/" rel="noopener" target="_blank">AWS Global Accelerator</a> through a new declarative Kubernetes API. This integration brings the power of AWS’s global network infrastructure directly into your Kubernetes workflows, enabling improved application performance by up to 60% for users worldwide, all without leaving your Kubernetes environment.</p> 
<p><a href="https://aws.amazon.com/global-accelerator/" rel="noopener" target="_blank">AWS Global Accelerator</a> is a networking service that improves the performance of applications for end users by routing traffic through AWS’s private global network backbone, bypassing the unpredictability of the public internet. It delivers up to 60% performance improvement by leveraging AWS’s global network, provides two static anycast IP addresses that serve as a fixed entry point to your applications, enables automatic failover to healthy endpoints in under 30 seconds, and gives you precise control over traffic distribution across AWS regions and endpoints. This post looks at how you can integrate this important new service to get the most out of it.</p> 
<h2>The problem we’ve solved</h2> 
<p>Until now, configuring AWS Global Accelerator for Kubernetes applications required additional steps through the AWS Management Console, AWS CLI, or AWS CloudFormation templates. This approach created operational overhead by introducing a separate management plane outside of Kubernetes. Manual changes could diverge from your infrastructure-as-code definitions, leading to configuration drift. Coordinating accelerator configuration with Kubernetes deployments added complexity to already intricate workflows, and accelerator status was not reflected in Kubernetes resources, limiting visibility into the state of your infrastructure.</p> 
<h2>Introducing the AWS Global Accelerator Controller</h2> 
<p>The new AWS Global Accelerator Controller, part of the AWS Load Balancer Controller, solves these challenges by bringing Global Accelerator management natively into Kubernetes. Using a Custom Resource Definition (CRD), you can now declaratively manage your entire Global Accelerator configuration alongside your other Kubernetes resources — enabling GitOps practices for global traffic management.</p> 
<p>Here’s how you can make the AWS Global Accelerator Controller work best for your configuration:</p> 
<h2>Prerequisites</h2> 
<p>Before getting started, ensure your environment meets the following requirements. The AWS Global Accelerator Controller requires Kubernetes v1.19+, AWS Load Balancer Controller v2.17.0+, and is only available in the commercial AWS partition (not GovCloud or China). You’ll also need to configure additional IAM permissions for Global Accelerator resource management by attaching a dedicated policy (via IAM Roles for Service Accounts (IRSA) or worker node roles) alongside your existing LBC permissions. Learn more <a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/v3.1/guide/globalaccelerator/installation/#kubernetes-cluster-requirements" rel="noopener" target="_blank">https://kubernetes-sigs.github.io/aws-load-balancer-controller/v3.1/guide/globalaccelerator/installation/#kubernetes-cluster-requirements</a></p> 
<h2>High level Architecture diagram</h2> 
<div class="wp-caption aligncenter" id="attachment_32596" style="width: 891px;">
 <img alt="High level Architecture diagram" class="size-full wp-image-32596" height="389" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/04/02/AGA_CRD_Diagrams-Page-1-1.png" width="881" />
 <p class="wp-caption-text" id="caption-attachment-32596">High level Architecture diagram</p>
</div> 
<p>When you define Global Accelerator resources using the Custom Resource Definition (CRD) shown on the left side of the diagram, the AWS Global Accelerator Controller watches these custom resources and translates them into actual AWS Global Accelerator configurations. The controller works alongside other specialized controllers in the Load Balancer Controller (LBC) suite, including the Ingress Controller, Service Controller, Gateway Controller, and TargetGroup Binding Controller, each handling different aspects of load balancing and traffic management.</p> 
<p>The AWS Global Accelerator Controller’s primary responsibility is to reconcile the desired state defined in your Kubernetes manifest with the actual state of Global Accelerator resources: Accelerators (which provide static IP addresses for global traffic entry points), Listeners (which define the ports and protocols for incoming connections), EndpointGroups (which specify the AWS regions and health check configurations), and Load Balancers (which serve as the actual endpoints receiving the accelerated traffic).</p> 
<p>The controller continuously ensures that any changes to these custom resources are reflected in the AWS Global Accelerator configuration, handling the creation, updates, and deletion of accelerators and their associated components.</p> 
<h2>Key Features of the Global Accelerator Controller</h2> 
<ol> 
 <li> <h3>CRD Design</h3> <p>The controller uses a single GlobalAccelerator resource to manage the complete AWS Global Accelerator hierarchy — accelerators, listeners, endpoint groups, and endpoints — all in one place. This design keeps configuration centralized while providing granular control over every layer of the stack.</p> <pre><code class="lang-yaml">apiVersion: aga.k8s.aws/v1beta1
 kind: GlobalAccelerator
 metadata:
   name: web-app-accelerator
   namespace: production
 spec:
   name: "web-app-accelerator"
   ipAddressType: IPV4
   listeners:
     - protocol: TCP
       portRanges:
         - fromPort: 80
           toPort: 80
         - fromPort: 443
           toPort: 443
       endpointGroups:
         - endpoints:
             - type: Ingress
               name: web-app-ingress
</code></pre> </li> 
 <li> <h3>Automatic Endpoint Discovery</h3> <p>One of the most powerful features of the controller is automatic endpoint discovery. The controller can automatically discover load balancers from your existing Kubernetes resources – including Network Load Balancers (NLBs) from Service type LoadBalancer, Application Load Balancers (ALBs) from Ingress resources, and both ALBs and NLBs from Gateway API resources.</p> <p>For simple use cases, the controller can also auto-configure listener protocols and port ranges by inspecting the referenced Ingress resource, reducing the amount of configuration you need to write.</p> <pre><code class="lang-yaml">apiVersion: aga.k8s.aws/v1beta1
 kind: GlobalAccelerator
 metadata:
   name: autodiscovery-accelerator
 spec:
   name: "autodiscovery-accelerator"
   listeners:
     - endpointGroups:
         - endpoints:
             - type: Ingress
               name: web-ingress
               weight: 200
</code></pre> </li> 
 <li> <h3>Full Lifecycle Management</h3> <p>The controller manages the complete lifecycle of your Global Accelerator resources. It provisions new accelerators from CRD specifications, updates the CRD status with the current state and ARNs from AWS resources, modifies existing configurations when the CRD changes, and cleans up AWS resources when the CRD is deleted. This ensures your Kubernetes state and AWS resources remain in sync at all times. The following diagram depicts these interactions.</p> <p></p>
  <div class="wp-caption aligncenter" id="attachment_32646" style="width: 797px;">
   <img alt="Sequence during CRUD operations" class="wp-image-32646 size-full" height="951" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/04/03/AGA_CRD_Diagrams-Page-3-1.jpg" width="787" />
   <p class="wp-caption-text" id="caption-attachment-32646">Sequence during CRUD operations</p>
  </div></li> 
 <li> <h3>Multi-Region Support</h3> <p>While auto-discovery works within the same AWS Region, you can manually configure endpoints in other AWS Regions for true multi-region deployments. By specifying endpoint ARNs directly in the CRD, you can distribute traffic across AWS Regions and build resilient, globally distributed architectures.<br /> For example:</p> <pre><code class="lang-yaml">apiVersion: aga.k8s.aws/v1beta1
kind: GlobalAccelerator
metadata:
  name: cross-region-accelerator
  namespace: default
spec:
  name: cross-region-accelerator
  ipAddressType: IPV4
  listeners:
    - protocol: TCP
      portRanges:
        - fromPort: 443
          toPort: 443
      endpointGroups:
        - endpoints:
            - type: Service
              name: local-service
        - region: us-west-2
          trafficDialPercentage: 50
          endpoints:
            - type: EndpointID
              endpointID: &gt;-
                arn:aws:elasticloadbalancing:us-west-2:123456789012:loadbalancer/app/remote-lb/1234567890123456
              weight: 128</code></pre> </li> 
 <li> <h3>Advanced Traffic Management</h3> <p>The controller supports sophisticated traffic management capabilities. You can control traffic distribution across endpoints using weights — for example, configuring a 2:1 traffic ratio between two endpoints. Port overrides allow you to map listener ports to different Global Accelerator endpoint ports, and client affinity using SOURCE_IP ensures that users are consistently routed to the same endpoint, which is essential for stateful applications.</p> <pre><code class="lang-yaml">
apiVersion: aga.k8s.aws/v1beta1
kind: GlobalAccelerator
metadata:
  name: port-override-accelerator
  namespace: default
spec:
  name: port-override-accelerator
  ipAddressType: IPV4
  listeners:
    - protocol: TCP
      portRanges:
        - fromPort: 80
          toPort: 80
        - fromPort: 443
          toPort: 443
      clientAffinity: SOURCE_IP
      endpointGroups:
        - portOverrides:
            - listenerPort: 80
              endpointPort: 8080
            - listenerPort: 443
              endpointPort: 8443
          endpoints:
            - type: Service
              name: backend-service</code></pre> </li> 
 <li> <h3>Bring Your Own IP (BYOIP) Support</h3> <p>For organizations with their own IP address ranges, the controller supports Bring Your Own IP (BYOIP). You can specify custom IP addresses directly in the CRD spec, giving you full control over the static entry points to your applications.</p> <pre><code class="lang-yaml">apiVersion: aga.k8s.aws/v1beta1
kind: GlobalAccelerator
metadata:
  name: byoip-accelerator
  namespace: default
spec:
  name: byoip-accelerator
  ipAddressType: IPV4
  ipAddresses:
    - 198.51.100.10
  listeners:
    - protocol: TCP
      portRanges:
        - fromPort: 443
          toPort: 443
      endpointGroups:
        - endpoints:
            - type: Ingress
              name: secure-ingress
</code></pre> </li> 
 <li> <h3>Comprehensive Status Reporting</h3> <p>The controller keeps your CRD status up to date with real-time information from AWS, including the accelerator’s ARN, DNS name, deployment status, and assigned IP addresses. You can retrieve the status using the following command:</p> <pre><code class="lang-bash">kubectl get globalaccelerator &lt;accelerator-crd-name&gt; -o yaml
(replace the accelerator-crd-name with your crd name)

status:
&nbsp;&nbsp; acceleratorARN: arn:aws:globalaccelerator::123456789012:accelerator/abc123
&nbsp;&nbsp; dnsName: a1234567890abcdef.awsglobalaccelerator.com
&nbsp;&nbsp; status: DEPLOYED
&nbsp;&nbsp; ipSets:
&nbsp;&nbsp;&nbsp;&nbsp; - ipAddresses:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - "192.0.2.1"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - "192.0.2.2"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ipAddressFamily: IPv4
</code></pre> </li> 
</ol> 
<h2>Common AWS Global Accelerator Controller scenarios</h2> 
<h3>Application Performance</h3> 
<p>For applications with a geographically distributed user base, the AWS Global Accelerator Controller makes it straightforward to route traffic through AWS’s global network. By referencing your existing Ingress resource in a GlobalAccelerator CRD, you can reduce latency for users worldwide without changing your application code — making it ideal for latency-sensitive workloads. Here’s an example that accelerates traffic to a single ingress resource: <a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/globalaccelerator/examples/#single-ingress-acceleration" rel="noopener" target="_blank">https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/globalaccelerator/examples/#single-ingress-acceleration</a></p> 
<h3>Multi-Region Failover</h3> 
<p>The controller supports automatic failover between AWS Regions. You can configure your primary region to receive 100% of traffic while a failover region sits at 0%, ready to absorb traffic automatically if the primary region becomes unhealthy. This pattern provides high availability with minimal operational effort. See this multi-region failover example: <a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/globalaccelerator/examples/#multi-region-automatic-failover" rel="noopener" target="_blank">https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/globalaccelerator/examples/#multi-region-automatic-failover</a></p> 
<h3>Blue-Green Deployments</h3> 
<p>Endpoint weights enable gradual traffic shifts between application versions. By assigning a higher weight to your blue environment and a lower weight to green, you can incrementally roll out new versions and monitor their behavior before completing the cutover — reducing deployment risk without requiring additional infrastructure. See this blue-green deployment example: <a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/globalaccelerator/examples/#multi-region-automatic-failover" rel="noopener" target="_blank">https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/globalaccelerator/examples/#multi-region-automatic-failover</a></p> 
<h3>Pricing</h3> 
<p>AWS Global Accelerator pricing has three components: a fixed hourly fee of $0.025 per accelerator (approximately $18/month), a Data Transfer-Premium (DT-Premium) fee per GB that varies by source AWS Region, destination edge location, and standard public IPv4 address charges. You are only charged DT-Premium on the dominant direction of traffic (inbound or outbound) each hour, not both. The DT-Premium fee is in addition to standard EC2 Data Transfer Out fees. There are no upfront commitments. For full pricing details and regional rates, visit the AWS Global Accelerator pricing page: <a href="https://aws.amazon.com/global-accelerator/pricing/" rel="noopener" target="_blank">https://aws.amazon.com/global-accelerator/pricing/</a></p> 
<h2>Getting Started</h2> 
<p>To get started, follow the <a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/globalaccelerator/installation/" rel="noopener" target="_blank">Installation Guide</a> and explore the examples repository for common deployment patterns. If you encounter issues or have feature requests, <a href="https://github.com/kubernetes-sigs/aws-load-balancer-controller" rel="noopener" target="_blank">open an issue on GitHub</a>. We welcome contributions from the community — visit the <a href="https://github.com/kubernetes-sigs/aws-load-balancer-controller" rel="noopener" target="_blank">AWS Load Balancer Controller GitHub</a> to get involved.</p> 
<h2>Current Limitations</h2> 
<p>There are two current limitations to be aware of. First, IP addresses specified via BYOIP cannot be updated after accelerator creation. To change IP addresses, you must create a new accelerator. Second, auto-discovery currently works only within the same AWS Region as the controller. For cross-region configurations, use manual endpoint registration with ARNs.</p> 
<h2>Conclusion</h2> 
<p>The AWS Global Accelerator Controller brings the full power of AWS’s global network directly into your Kubernetes workflows. By managing Global Accelerator resources declaratively through Kubernetes CRDs, you can improve application performance for global users, simplify operational workflows, maintain configuration consistency, and enable GitOps practices for global traffic management. We encourage you to try the new AWS Global Accelerator Controller and share your feedback with us.</p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="jaiganesh.jpg"><img alt="Jaiganesh Girinathan" class="alignleft size-full wp-image-5363" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2022/04/12/jaiganesh.jpeg" width="125" /></p> 
 <h3 class="lb-h4">Jaiganesh Girinathan</h3> 
 <p style="color: #879196; font-size: 1.2rem;">Jaiganesh Girinathan is a Principal Edge Specialist Solutions Architect focused on content delivery networks and edge computing capabilities with AWS. He has worked with several media customers globally over the last two decades, helping organizations modernize &amp; scale their platforms. He is passionate about building solutions to address key customer needs. Outside of work, you can usually find Jaiganesh star gazing!</p> 
</div> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Shraddha.png"><img alt="Shraddha Bang" class="alignleft size-full wp-image-5363" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/04/02/Shraddha.png" width="125" /></p> 
 <h3 class="lb-h4">Shraddha Bang</h3> 
 <p style="color: #879196; font-size: 1.2rem;">Shraddha Bang is a Software Development Engineer on the AWS Elastic Load Balancing team. She specializes in developing customer-facing surfaces such as the AWS Console, CloudFormation, and Kubernetes controllers to improve cloud-native networking. When she isn’t coding, she likes to travel, explore new cities, and dive into history and architecture.</p> 
</div>
