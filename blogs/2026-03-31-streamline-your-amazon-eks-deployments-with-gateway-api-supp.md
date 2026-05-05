---
title: "Streamline your Amazon EKS deployments with Gateway API support for AWS Load Balancer Controller and Amazon VPC Lattice"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/streamline-your-amazon-eks-deployments-with-gateway-api-support-for-aws-load-balancer-controller-and-amazon-vpc-lattice/"
date: "Tue, 31 Mar 2026 16:31:38 +0000"
author: "Alexandra Huides"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
<p>Building on the recent announcement of <a href="https://aws.amazon.com/blogs/networking-and-content-delivery/aws-load-balancer-controller-adds-general-availability-support-for-kubernetes-gateway-api/">Gateway API support in AWS Load Balancer Controller</a>, in this post we demonstrate a practical architecture that uses both controllers through a single API specification. This approach simplifies operations while maintaining the flexibility to choose the right AWS service for each networking requirement.</p> 
<p>Managing application networking in Kubernetes has traditionally required learning multiple APIs and controller-specific configurations. Teams deploying on <a href="https://aws.amazon.com/eks/">Amazon Elastic Kubernetes Service (Amazon EKS)</a> often use <a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/">AWS Load Balancer Controller</a> for internet-facing traffic and various solutions for service-to-service communication. This fragmentation of configuration APIs creates operational complexity and increases the learning curve.</p> 
<p>Gateway API addresses this challenge by providing a unified, role-oriented API for configuring ingress and service-to-service communication. This post demonstrates how to use Gateway API with both AWS Load Balancer Controller and <a href="https://www.gateway-api-controller.eks.aws.dev/latest/">Amazon VPC Lattice</a>. While we use different AWS services for different networking layers (ALB for internet ingress, VPC Lattice for service-to-service communication), Gateway API provides a consistent configuration interface. Instead of learning Ingress annotations, custom CRDs, and service-specific configurations, you can use a single, standardized API across distinct networking layers.</p> 
<h2>Understanding the Kubernetes Gateway API</h2> 
<p>Gateway API is an official Kubernetes project focused on L4 and L7 routing in Kubernetes. This project represents the next generation of Kubernetes Ingress, Load Balancing, and Service Mesh APIs. It has been designed to be generic, expressive, and role oriented. Gateway API is a collection of Kubernetes Custom Resource Definitions (CRDs) that model service networking. Unlike the traditional Ingress API, Gateway API provides:</p> 
<ul> 
 <li><strong>Role-oriented design</strong>: Separate resources for infrastructure operators (GatewayClass, Gateway) and application developers (HTTPRoute, GRPCRoute), with clear separation of ownership.</li> 
 <li><strong>Expressive routing</strong>: Rich traffic routing capabilities including header-based routing and weighted routing, without requiring controller-specific annotations.</li> 
 <li><strong>Extensibility</strong>: A standardized way to extend functionality through policy attachments and custom resources.</li> 
 <li><strong>Portability</strong>: Consistent API across different implementations, making it easier to switch between or combine multiple controllers.</li> 
</ul> 
<p>The core resources in Gateway API are:</p> 
<ul> 
 <li>GatewayClass: Defines a class of Gateways that can be instantiated, similar to StorageClass for persistent volumes. Each GatewayClass specifies which controller will manage Gateways of that class.</li> 
 <li>Gateway: Represents an instance of a load balancer or proxy. It defines listeners (ports and protocols) and references a GatewayClass to determine the implementation.</li> 
 <li>HTTPRoute: Defines HTTP traffic routing rules, including hostname matching, path-based routing, and backend service selection.</li> 
</ul> 
<p>These are shown in the following diagram (Figure 1). For details, refer to the Kubernetes Gateway API documentation.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/figure-1-1.png"><img alt="" class="aligncenter size-full wp-image-32542" height="524" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/figure-1-1.png" width="600" /></a></p> 
<p style="text-align: center;">Figure 1: Kubernetes Gateway API specification (<a href="https://gateway-api.sigs.k8s.io/">source</a>)</p> 
<h2>Architecture considerations: Choosing the right controller</h2> 
<p>When designing your EKS networking architecture with Gateway API, it’s important to understand when to use each controller. Both AWS Load Balancer Controller and VPC Lattice Gateway API Controller implement the same Gateway API specification, but they are optimized for different traffic patterns and use cases.</p> 
<p>Use AWS Load Balancer Controller when you need any of the below:</p> 
<ul> 
 <li><strong>Internet ingress</strong>: Your service needs to be accessible from the public internet</li> 
 <li><strong>AWS Web Application Firewall (WAF) integration</strong>: You require AWS WAF for application-layer security</li> 
 <li><strong>Advanced ALB features</strong>: You need features like OIDC integration (<a href="https://aws.amazon.com/cognito/">Amazon Cognito</a>), fixed response rules, or redirect actions; You can also integrate <a href="https://aws.amazon.com/cloudfront/">Amazon CloudFront</a> with your private Application Load Balancers as <a href="https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html">VPC origins</a>.</li> 
</ul> 
<p>Example use cases include public-facing web applications, REST APIs consumed by mobile applications or third-party services, GraphQL endpoints for external developers, or webhook receivers from external systems.</p> 
<p>Use VPC Lattice Gateway API Controller when you need any of the below:</p> 
<ul> 
 <li><strong>Cross-cluster communication</strong>: Services need to communicate across multiple EKS clusters</li> 
 <li><strong>Cross-VPC connectivity</strong>: Services span multiple VPCs and they need to communicate with each other, or services have targets across multiple VPCs.</li> 
 <li><strong>Cross-Account access</strong>: Services in different AWS accounts need to communicate</li> 
 <li><strong>Service-to-service communication features</strong>: You need features like automatic service discovery, traffic management, service-level routing capabilities</li> 
 <li><strong>Authentication and authorization</strong>: You want to enforce service-to-service authentication and authorization policies</li> 
 <li><strong>Simplified networking</strong>: You want to avoid managing network connectivity services such as VPC peering or <a href="https://aws.amazon.com/transit-gateway/">AWS Transit Gateway</a>.</li> 
 <li><strong>Mixed compute options</strong>: You have applications that use <a href="https://aws.amazon.com/lambda/">AWS Lambda</a>, <a href="https://aws.amazon.com/ecs/">Amazon Elastic Container Service (ECS)</a>, <a href="https://aws.amazon.com/fargate/">ECS Fargate</a>, or <a href="https://aws.amazon.com/ec2/">Amazon Elastic Compute Cloud (EC2)</a> instances, and you want simplified connectivity across all compute options.</li> 
</ul> 
<p>Example use cases include microservices architectures with services distributed across clusters, multi-tenant platforms, hybrid architectures with services using multiple compute options such as EKS, Lambda and ECS, internal APIs that should not be exposed to the internet, or services requiring fine-grained access control</p> 
<p>You can also use both controllers together in the same cluster. This is the most flexible pattern: use AWS Load Balancer Controller for internet-facing services and VPC Lattice Gateway API Controller for internal service-to-service communication. Each controller watches only for Gateways that reference its GatewayClass, so they operate independently without conflict. The GatewayClass Name field in your Gateway resource determines which controller manages it.</p> 
<h2>Architecture example: EKS multi-cluster application networking</h2> 
<p>To demonstrate the simplicity of this unified approach, we explore a practical architecture that uses both controllers. Consider a microservices application with:</p> 
<ul> 
 <li>A frontend service named <code>shop</code> that needs to be exposed to clients on the internet</li> 
 <li>Internal services named <code>payments</code>, <code>cart</code> and <code>inventory</code> that need to communicate with each other</li> 
 <li>These services are distributed across multiple EKS clusters for isolation or organizational boundaries</li> 
</ul> 
<p>In the architecture shown in figure 2, the <strong>Edge cluster </strong>uses both controllers simultaneously for complementary use cases: AWS Load Balancer Controller manages the <code>shop.api.example.com</code> service exposed to the internet via ALB, and VPC Lattice Gateway API Controller manages the <code>payments.api.example.com</code> service for internal consumption via VPC Lattice. This demonstrates that both controllers coexist in the same cluster, each managing their respective GatewayClass.</p> 
<p><strong>Apps-A cluster</strong> uses VPC Lattice Gateway API Controller to expose the <code>cart.api.example.com</code> service, and Apps-B cluster uses VPC Lattice Gateway API Controller to expose the <code>inventory.api.example.com</code> service. All internal services communicate with each other through VPC Lattice, which provides automatic service discovery and routing across clusters without requiring any additional networking configuration.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/figure-2_lbc-and-vpc-lattice-setup.png"><img alt="" class="aligncenter size-full wp-image-32543" height="2261" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/figure-2_lbc-and-vpc-lattice-setup.png" width="2466" /></a></p> 
<p style="text-align: center;">Figure 2: Architecture diagram with the 3 EKS clusters and the 4 apps</p> 
<p>The following diagram (Figure 3) shows the Edge EKS cluster in more detail. Both controllers are deployed as pods in the cluster, each watching for Gateway resources that reference their GatewayClass:</p> 
<ul> 
 <li>The AWS Load Balancer Controller watches for Gateways referencing <code>aws-alb-gateway-class</code> and provisions ALBs accordingly.</li> 
 <li>The VPC Lattice Gateway API Controller watches for Gateways referencing <code>amazon-vpc-lattice</code> and creates VPC Lattice services in the associated service network.</li> 
</ul> 
<p>When an HTTPRoute is created, the corresponding controller picks it up based on which Gateway it references, and configures the routing rules on the appropriate AWS resource.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-3.png"><img alt="" class="aligncenter size-full wp-image-32544" height="2059" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-3.png" width="3133" /></a></p> 
<p style="text-align: center;">Figure 3: Edge EKS cluster running both controllers</p> 
<h2>Traffic flows</h2> 
<p>Understanding how traffic flows through this architecture helps clarify which controller handles each networking layer. The following diagrams show the complete request path from internet client to backend services.</p> 
<h3>Flow 1: Internet client to shop service (ALB)</h3> 
<p>Figure 4 details the internet ingress traffic flow. When an external client sends a request to <code>shop.api.example.com</code>, the Application Load Balancer receives the request and terminates TLS using the certificate provisioned through <a href="https://aws.amazon.com/certificate-manager/">AWS Certificate Manager (ACM)</a>. The ALB then forwards the traffic to the shop pods running in the Edge EKS cluster, using the target group that the AWS Load Balancer Controller automatically configures and keeps in sync as pods scale up or down. The entire flow is managed by the AWS Load Balancer Controller, which watches for changes to the Gateway and HTTPRoute resources and updates the ALB configuration accordingly.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-4.png"><img alt="" class="aligncenter size-full wp-image-32545" height="2743" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-4.png" width="3144" /></a></p> 
<p style="text-align: center;">Figure 4: Traffic flow 1 – internet clients to shop ALB</p> 
<h3>Flow 2: Cross-cluster communication (VPC Lattice)</h3> 
<p>Figure 5 details the cross-cluster traffic flow. When the shop pod calls cart.api.example.com, the request is resolved by VPC Lattice DNS to a VPC Lattice endpoint. VPC Lattice routes the request to the cart service running in Apps-A cluster, without requiring VPC peering, Transit Gateway, or any additional networking configuration. The VPC Lattice Gateway API Controller in each cluster is responsible for registering its services with the shared service network and keeping target groups up to date as pods scale.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-5.png"><img alt="" class="aligncenter size-full wp-image-32546" height="3195" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-5.png" width="2454" /></a></p> 
<p style="text-align: center;">Figure 5: Traffic flow 2 – shop calls cart through VPC Lattice</p> 
<h3>End-to-end flow</h3> 
<p>The end-to-end flow combines both controllers. An external client sends a request to shop.api.example.com, which the ALB receives and routes to the shop pods in the Edge cluster. The shop application then calls internal services such as <code>cart.api.example.com</code> and <code>inventory.api.example.com</code>. These DNS names resolve to VPC Lattice endpoints, which route the requests to the corresponding pods in Apps-A and Apps-B clusters. The shop pod can also call&nbsp;<code>payments.api.example.com</code>, which VPC Lattice routes to the payments pods in the same Edge cluster. VPC Lattice handles the cross-cluster routing, target health checking, and load balancing transparently.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-6.png"><img alt="" class="aligncenter size-full wp-image-32547" height="4917" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-6.png" width="2454" /></a></p> 
<p style="text-align: center;">Figure 6: End-to-end flow</p> 
<h2>Key configuration highlights</h2> 
<p>The detailed installation instructions and complete code examples are in our <a href="https://github.com/aws-samples/sample-eks-kubernetes-gateway-api">GitHub repository</a>. AWS provides two controllers that implement the Gateway API specification, each optimized for different use cases.</p> 
<h3>AWS Load Balancer Controller with support for Gateway API</h3> 
<p>The AWS Load Balancer Controller manages Elastic Load Balancing resources for Kubernetes clusters. With Gateway API support, it creates Application Load Balancers (ALBs) or Network Load Balancers (NLBs) based on Gateway resources. The following diagram (figure 7) shows the mapping between Gateway API specification and the AWS Load Balancer Controller components:</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-7.png"><img alt="" class="aligncenter size-full wp-image-32548" height="2106" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-7.png" width="3398" /></a></p> 
<p style="text-align: center;">Figure 7: Gateway API specification mapping to AWS LBC components</p> 
<p>Below are definition examples for:</p> 
<ul> 
 <li>GatewayClass for AWS Load Balancer Controller:</li> 
</ul> 
<pre><code class="lang-yaml"># alb-gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
  kind: GatewayClass
  metadata:
    name: aws-alb-gateway-class
  spec:
    controllerName: gateway.k8s.aws/alb
</code></pre> 
<ul> 
 <li>The corresponding Gateway instantiation of the GatewayClass that triggers ALB provisioning:</li> 
</ul> 
<pre><code class="lang-yaml"># my-alb-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
  kind: Gateway
  metadata:
    name: my-alb-gateway
  namespace: eks-gateway-demo
  spec:
    gatewayClassName: aws-alb-gateway-class
    infrastructure:
      parametersRef:
        kind: LoadBalancerConfiguration
        name: lbconfig-gateway
        group: gateway.k8s.aws
     listeners:
       - name: https
          hostname: "ahop.api.example.com"
          protocol: HTTPS
          port: 443
          allowedRoutes:
            namespaces:
              from: Same</code></pre> 
<ul> 
 <li>LoadBalancerConfiguration that specifies custom configuration parameters for your load balancer, such as the ACM certificate:</li> 
</ul> 
<pre><code class="lang-yaml">apiVersion: gateway.k8s.aws/v1beta1
kind: LoadBalancerConfiguration
metadata:
  name: shop-alb-config
  namespace: eks-gateway-demo
spec:
  listenerConfigurations:
    - protocolPort: HTTPS:443
      defaultCertificate: arn:aws:acm:REGION:ACCOUNT_ID:certificate/CERT_ID</code></pre> 
<ul> 
 <li>And the HTTPRoute referring to the Kubernetes service for shop:</li> 
</ul> 
<pre><code class="lang-yaml">apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-route
  namespace: eks-gateway-demo
spec:
  parentRefs:
    - name: shop-gateway
      sectionName: https
  hostnames:
    - shop.api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: shop
          port: 80</code></pre> 
<h3>Amazon VPC Lattice Gateway API Controller</h3> 
<p>The VPC Lattice Gateway API Controller creates Amazon VPC Lattice services and associates them automatically with the service network. VPC Lattice is a fully managed application networking service that simplifies service-to-service communication across VPCs and accounts.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-8.png"><img alt="" class="aligncenter size-full wp-image-32549" height="2106" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/28/fig-8.png" width="3398" /></a></p> 
<p style="text-align: center;">Figure 8: Gateway API specification mapping to Amazon VPC Lattice components</p> 
<p>Below are definition examples for:</p> 
<ul> 
 <li>GatewayClass for VPC Lattice:</li> 
</ul> 
<pre><code class="lang-yaml">apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: amazon-vpc-lattice
spec:
  controllerName: application-networking.k8s.aws/gateway-api-controller
</code></pre> 
<ul> 
 <li>The corresponding Gateway instantiation that uses the VPC Lattice service network:</li> 
</ul> 
<pre><code class="lang-yaml">apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: core-apis-service-network
  namespace: eks-gateway-demo
spec:
  gatewayClassName: amazon-vpc-lattice
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      allowedRoutes:
        namespaces:
          from: Same
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same</code></pre> 
<ul> 
 <li>And the HTTPRoute for the <code>payments</code> VPC Lattice service:</li> 
</ul> 
<pre><code class="lang-yaml">apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: payments-route
  namespace: eks-gateway-demo
spec:
  parentRefs:
    - name: core-apis-service-network
      sectionName: https
  hostnames:
    - payments.api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: payments
          port: 80</code></pre> 
<p>The Gateway and HTTPRoute resources use the same CRDs regardless of which controller you are using. The only difference between the two is the gatewayClassName field, which determines whether you configure an ALB or a VPC Lattice service. This architecture gives you flexibility to standardize on using Gateway API as the developer-facing contract, then select the appropriate data plane for each networking layer.</p> 
<h2>Validating the architecture</h2> 
<p>Once deployed, you can validate that traffic flows correctly through both controllers. The complete test suite with automated validation scripts is in the <a href="https://github.com/aws-samples/sample-eks-kubernetes-gateway-api">GitHub repository</a>. Also check our application deployment for a visual test of the end-to-end flow.</p> 
<h3>Validation flow</h3> 
<p><strong>Test 1: Internet → ALB → Shop Service</strong></p> 
<p>curl https://shop.api.example.com</p> 
<p>✓ Validates: AWS Load Balancer Controller<br /> ✓ Validates: ALB provisioning and routing<br /> ✓ Validates: TLS termination with ACM</p> 
<p><strong>Test 2: Shop → Inventory (Cross-Cluster via VPC Lattice)</strong></p> 
<p>From shop pod:<br /> curl http://inventory.api.example.com</p> 
<p>✓ Validates: Cross-cluster VPC Lattice routing<br /> ✓ Validates: Service network associations<br /> ✓ Validates: Multi-cluster service discovery</p> 
<h2>Migration strategy: AWS Load Balancer Controller with Ingress and Gateway API support</h2> 
<p>The AWS Load Balancer Controller supports both the Ingress API and Gateway API simultaneously in the same cluster. This means you can:</p> 
<ol> 
 <li>Keep existing Ingress resources running. Your current production traffic continues to flow through existing ALBs managed by Ingress resources.</li> 
 <li>Deploy new services using Gateway API. New applications can use Gateway API from day one.</li> 
 <li>Gradually migrate existing services from Ingress to Gateway API at your own pace.</li> 
</ol> 
<p>The AWS Load Balancer Controller watches for both resource types:</p> 
<pre><code class="lang-yaml"># Your existing Ingress resources continue to work
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: legacy-app
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - host: legacy.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: legacy-service
                port:
                  number: 80

---
# New Gateway API resources work alongside Ingress
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: new-app-gateway
spec:
  gatewayClassName: aws-alb-gateway-class
  listeners:
    - name: https
      protocol: HTTPS
      port: 443

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: new-app-route
spec:
  parentRefs:
    - name: new-app-gateway
  hostnames:
    - newapp.example.com
  rules:
    - backendRefs:
        - name: new-service
          port: 80
</code></pre> 
<h3>Migration approach</h3> 
<p>When migrating from Ingress to Gateway API, the key advantage is that both APIs can coexist in the same cluster. To migrate, start by updating your AWS Load Balancer Controller to a version that supports Gateway API (v2.14.0+) and installing the Gateway API CRDs. Your existing Ingress resources continue to work without any changes. For new services, you can start using Gateway API resources. For existing services, you can migrate them one at a time by creating equivalent Gateway and HTTPRoute resources alongside the existing Ingress resources. Once you validate that traffic flows correctly through the new Gateway API resources, you can delete the old Ingress resources.</p> 
<p>This gradual approach allows you to validate each migration step, maintain rollback options, and avoid service disruptions. There is no requirement to migrate all services. You can run both APIs long-term if needed, choosing Gateway API for new services while keeping stable production services on Ingress until you are ready to migrate them.</p> 
<h2>Considerations</h2> 
<p>When implementing this architecture in production, consider the following:</p> 
<ul> 
 <li><strong>Security</strong>: You can use HTTPS for all services, not just internet-facing ones. VPC Lattice supports TLS termination and can integrate with ACM for public certificate management, and with ACM’s Private Certificate Authority (PCA). Implement IAM-based authentication for service-to-service communication using VPC Lattice auth policies.</li> 
 <li><strong>Observability</strong>: Enable access logging for both ALB and VPC Lattice services. VPC Lattice can send logs to Amazon S3, Amazon CloudWatch Logs, or Amazon Kinesis Data Firehose.</li> 
 <li><strong>High availability</strong>: Deploy Gateway resources in multiple Availability Zones by ensuring your EKS node groups span multiple AZs. Both ALB and VPC Lattice automatically distribute traffic across healthy targets.</li> 
 <li><strong>Cluster lifecycle</strong>: When deleting clusters, ensure Gateway resources are deleted first to allow controllers to clean up AWS resources properly. The cleanup scripts in our <a href="https://github.com/aws-samples/sample-eks-kubernetes-gateway-api">GitHub repository</a> demonstrate the correct deletion order.</li> 
</ul> 
<h2>Conclusion</h2> 
<p>Kubernetes Gateway API provides a unified interface for configuring application networking. By using Gateway API with both AWS Load Balancer Controller and Amazon VPC Lattice, you can create multi-cluster architectures while maintaining a consistent operational experience.</p> 
<p>The architecture example we explored demonstrates how to combine internet ingress with Application Load Balancer and service-to-service communication with VPC Lattice, all through a single API. Both controllers coexist in the same cluster, each managing their respective GatewayClass. This allows you to use the right tool for each networking layer without learning multiple APIs.</p> 
<h2>Get started now</h2> 
<p>To get started with this architecture, visit our <a href="https://github.com/aws-samples/sample-eks-kubernetes-gateway-api">GitHub repository</a> for complete code examples, detailed tutorials, and troubleshooting guides. The repository includes:</p> 
<ul> 
 <li>Complete Kubernetes manifests for all three clusters</li> 
 <li>Automated setup scripts for quick deployment</li> 
 <li>Step-by-step tutorials in a beginner-friendly format</li> 
 <li>Troubleshooting guide with common issues and solutions</li> 
</ul> 
<p>For more information about Gateway API support in AWS services, see:</p> 
<ul> 
 <li><a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/gateway/">AWS Load Balancer Controller Gateway API documentation</a></li> 
 <li><a href="https://www.gateway-api-controller.eks.aws.dev/latest/guides/deploy/">Amazon VPC Lattice Gateway API Controller documentation</a></li> 
 <li><a href="https://gateway-api.sigs.k8s.io/">Kubernetes Gateway API documentation</a></li> 
</ul> 
<h2>About the authors</h2> 
<p>
 <!-- First Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="alex_huides.jpg"><img alt="AlexHuides.jpg" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2024/08/08/alex-blog-bio-resized.png" width="125" /></p> 
 <h3 class="lb-h4">Alexandra Huides</h3> 
 <p style="color: #879196; font-size: 1rem;">Alexandra Huides is a Principal Networking Specialist Solutions Architect in the AWS Networking Services product team at Amazon Web Services. She focuses on helping customers build and develop networking architectures for highly scalable and resilient AWS environments. Alex is also a public speaker for AWS, and is helping customers adopt IPv6. Outside work, she loves sailing, especially catamarans, traveling, discovering new cultures, running and reading.</p> 
</div> 
<p>
 <!-- Second Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="Pablo"><img alt="Pablo" class="alignleft wp-image-1288 size-thumbnail" height="125net" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/03/31/pablo_unicorn-small.png" width="125" /></p> 
 <h3 class="lb-h4">Pablo Sánchez Carmona</h3> 
 <p style="color: #879196; font-size: 1rem;">Pablo is a Senior Network Specialist Solutions Architect at AWS, where he helps customers to design secure, resilient and cost-effective networks. When not talking about Networking, Pablo can be found playing basketball or video-games. He holds a MSc in Electrical Engineering from the Royal Institute of Technology (KTH), and a Master’s degree in Telecommunications Engineering from the Polytechnic University of Catalonia (UPC).</p> 
</div>
