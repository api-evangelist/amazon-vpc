---
title: "How LSEG connects the world of finance: Using AWS Cross-Region PrivateLink to transform global market data access"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/how-lseg-connects-the-world-of-finance-using-aws-cross-region-privatelink-to-transform-global-market-data-access/"
date: "Wed, 08 Apr 2026 16:57:51 +0000"
author: "Rohit Singh"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
<p>The London Stock Exchange Group (LSEG) is a leading global financial markets infrastructure and data provider, serving over 25,000 customers across 190 countries. The company operates the London Stock Exchange and provides critical market data, analytics, and trading technology to banks, asset managers, hedge funds, and other financial institutions worldwide. Through its Real-Time Optimized (RTO) platform, LSEG delivers over 8 million price updates each second, supporting high-frequency trading, algorithmic strategies, and real-time risk management across global capital markets.</p> 
<p>Financial institutions need consistent, low-latency access to market data across multiple geographic regions. LSEG’s RTO platform addresses this imperative by providing a cloud-native market data distribution service that delivers low latency to trading applications, risk management systems, and analytics platforms worldwide.</p> 
<p>LSEG’s RTO platform has revolutionized this paradigm through the deployment of Cross-Region <a href="https://aws.amazon.com/privatelink/">AWS PrivateLink</a> technology, expanding low latency market data access from six <a href="https://aws.amazon.com/">Amazon Web Services</a> (AWS) <a href="https://aws.amazon.com/about-aws/global-infrastructure/regions_az/">Regions</a> to a comprehensive 32-Region global footprint. This transformation enables financial institutions to access real-time market data from previously inaccessible AWS Regions while maintaining enterprise-grade security, reducing infrastructure costs, and streamlining network architecture.</p> 
<p>In this post, we explore the technical architecture behind LSEG’s Cross-Region PrivateLink implementation, examine the AWS services that power this solution, and detail the performance and security benefits that make this approach compelling for financial institutions seeking to modernize their market data operations across geographic boundaries.</p> 
<h2>&nbsp;LSEG’s global expansion requirements</h2> 
<p>LSEG’s RTO service sought to expand its global reach to better serve financial institutions worldwide. This required addressing several key requirements:</p> 
<ul> 
 <li><strong>Broader regional coverage</strong>: Extending RTO’s availability beyond the initial six strategic AWS Regions to serve customers in emerging markets and new geographic areas.</li> 
 <li><strong>Simplified deployment model</strong>: Enabling global access without requiring infrastructure deployment in every target AWS Region, reducing operational complexity for both LSEG and customers.</li> 
 <li><strong>Enhanced customer flexibility</strong>: Allowing financial institutions to access RTO from any AWS Region where they operate, no matter where LSEG’s infrastructure is deployed.</li> 
 <li><strong>Streamlined connectivity</strong>: Eliminating the need for precise Availability Zone (AZ) coordination between provider and consumer endpoints to streamline customer onboarding.</li> 
 <li><strong>Predictable cost structure</strong>: Providing transparent, scalable pricing for cross-Region access based upon customer usage rather than infrastructure overhead.</li> 
</ul> 
<p>With over 8 million price updates each second flowing through the RTO platform and data volumes growing 20–40% a year, LSEG sought to expand global access to its market data service. Initially available in six AWS Regions, LSEG recognized an opportunity to expand to more geographic locations without deploying infrastructure in each new AWS Region. Cross-Region PrivateLink enabled this expansion, allowing LSEG to extend RTO’s reach to 32 AWS Regions while maintaining the same low-latency performance and security standards.</p> 
<h3>Existing connectivity limitations</h3> 
<p>Before Cross-Region PrivateLink, financial institutions had three primary connectivity options, each with trade-offs:</p> 
<ol> 
 <li>Internet connectivity: Although widely available, internet-based connections introduce latency variability and raise questions about the security of sensitive market data.</li> 
 <li>In-Region PrivateLink: Provides excellent security and performance but limits customers to AWS Regions where LSEG has deployed RTO infrastructure.</li> 
 <li>Delivery direct: Offers dedicated connectivity but requires physical infrastructure deployment and is limited to specific geographic locations.</li> 
</ol> 
<h2>How Cross-Region PrivateLink transforms market data delivery</h2> 
<p>Cross-Region PrivateLink technology fundamentally changes how LSEG’s RTO service delivers market data across geographic boundaries. This cloud approach provides private connectivity between VPC endpoint services across AWS Regions without exposing traffic to the public internet.</p> 
<h3>Architecture overview</h3> 
<p>The following diagram (Figure 1) shows the Cross-Region PrivateLink connectivity architecture for RTO:</p> 
<p><img alt="Figure 1: Cross-Region PrivateLink connectivity architecture for RTO" class="alignnone wp-image-32664 size-full" height="864" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/04/08/Figure-1-Cross-Region-PrivateLink-connectivity-architecture-for-RTO-1.png" width="822" /></p> 
<p><strong>Figure 1</strong>: Cross-Region PrivateLink connectivity architecture for RTO</p> 
<p>In this architecture:</p> 
<ol> 
 <li><strong>Provider-side configuration</strong>: LSEG’s RTO service runs in provider VPCs across six strategic AWS Regions: US-EAST-1, US-EAST-2, EU-WEST-1, EU-CENTRAL-1, AP-NORTHEAST-1, and AP-SOUTHEAST-1.</li> 
 <li><strong>Cross-Region endpoint services</strong>: Cross-Region PrivateLink connectivity exposes endpoint services that are accessible from remote AWS Regions.</li> 
 <li><strong>Consumer-side integration</strong>: Financial institutions can create interface VPC endpoints in their local AWS Region that connect securely to LSEG’s endpoint services in distant AWS Regions.</li> 
 <li><strong>AWS Backbone transport</strong>: All cross-Region traffic flows over the AWS private global network backbone, avoiding the public internet entirely.</li> 
</ol> 
<h2>Technical implementation of Cross-Region PrivateLink for RTO</h2> 
<p>LSEG’s implementation of Cross-Region PrivateLink for RTO provides several distinct technical advantages over traditional connectivity models:</p> 
<h3>Flexible multi-AZ deployment</h3> 
<p>Cross-Region PrivateLink builds upon the proven security and reliability of PrivateLink while extending connectivity across AWS Regions. With CRPL, consumers have the flexibility to deploy interface endpoints in any Availability Zone within their AWS Region, and can also choose to deploy across multiple Availability Zones for enhanced zonal resilience:</p> 
<p><code>#Example AWS CLI command to create a Cross-Region VPC Endpoint with multi-AZ resilience</code></p> 
<p><code>aws ec2 create-vpc-endpoint</code><br /> <code>--vpc-id vpc-0123456789abcdefg</code><br /> <code>--vpc-endpoint-type Interface</code><br /> <code>--service-name com.amazonaws.vpce.us-east-1.vpce-svc-0123456789abcdefg</code><br /> <code>--subnet-ids subnet-xxxxxxxxxxxxxxxxx subnet-yyyyyyyyyyyyyyyyyyyyy</code><br /> <code>--Region ap-southeast-2</code><br /> <code>--service-Region us-east-1 </code></p> 
<p>This command creates a VPC endpoint in the AP-SOUTHEAST-2 Region that connects to LSEG’s RTO service in US-EAST-1. The endpoint can be created in single or multiple subnets for flexibility, with AWS automatically handling the cross-Region connectivity.</p> 
<h3>Managed failover architecture</h3> 
<p>Cross-Region PrivateLink offers managed failover across Availability Zones in both the consumer and provider Regions:</p> 
<ul> 
 <li><strong>Consumer-side failover</strong>: When using the Regional DNS name to access a multi-AZ VPC endpoint, traffic is automatically rerouted to healthy Availability Zones in case a zone fails. This automatic failover requires using the Regional DNS endpoint rather than zonal-specific DNS names.</li> 
 <li><strong>Provider-side failover</strong>: If the failure of a zone affects the provider AWS Region, then AWS dynamically shifts traffic away from the affected Availability Zone to other healthy service Availability Zones to which the service is configured.</li> 
</ul> 
<p>The dual-layer failover capability provides superior resilience when compared to traditional connectivity methods.</p> 
<h2>Security model</h2> 
<p>The Cross-Region PrivateLink security model maintains the same principles as in-Region PrivateLink:</p> 
<p>Bidirectional authorization: Service providers can configure their endpoint services to require acceptance of endpoint connections, and consumers must create endpoints to approved services.</p> 
<p>Private DNS integration: Consumers can also enable private DNS for their endpoints, allowing applications to use the same DNS names across AWS Regions.</p> 
<p>Network-level security: All traffic between the consumer VPC endpoint and LSEG’s service flows over the AWS private network backbone, never crossing the public internet.</p> 
<h2>Integration with RTO authentication and discovery</h2> 
<p>The Cross-Region PrivateLink implementation integrates seamlessly with LSEG’s existing authentication and service discovery mechanisms:</p> 
<h3>Authentication flow</h3> 
<p>RTO supports both V1 (machine account) and V2 (service account) authentication mechanisms over Cross-Region PrivateLink connections. When accessing the service through a cross-Region endpoint, the authentication flow remains consistent:</p> 
<ol> 
 <li>Authenticate through Delivery Platform Token Service API to obtain a security token.</li> 
 <li>Use the security token to discover available service endpoints through the Service Discovery API.</li> 
 <li>Establish a connection to a streaming service through the Cross-Region PrivateLink endpoint.</li> 
 <li>Send a Login Request over the active connection using the security token.</li> 
 <li>Request the desired permissioned content.</li> 
</ol> 
<h2>LSEG Service Discovery API with Cross-Region PrivateLink</h2> 
<p>LSEG’s Service Discovery API has been enhanced to support Cross-Region PrivateLink connections through other parameters:</p> 
<p>Consumers can query the LSEG’s Service Discovery API for detailed information about available endpoints, including the Cross-Region PrivateLink service names required for establishing connections:</p> 
<p><code> {</code><br /> <code>&nbsp; "services": [</code><br /> <code>&nbsp;&nbsp;&nbsp; {</code><br /> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "port": 14002,</code><br /> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "location": ["us-east-1a", "us-east-1b"],</code><br /> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "transport": "tcp",</code><br /> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "provider": "aws",</code><br /> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "endpoint": "us-east-1-aws-3-sm.optimized-pricing-api.refinitiv.net",</code><br /> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "dataFormat": ["rwf"],</code><br /> <code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "privatelink": "com.amazonaws.vpce.us-east-1.vpce-svc-0123456789abcdefg"</code><br /> <code>&nbsp;&nbsp;&nbsp; },</code><br /> <code>&nbsp;&nbsp;&nbsp; // Additional endpoints...</code><br /> <code>&nbsp; ]</code><br /> <code>}</code></p> 
<h2>&nbsp;Connectivity options: After Cross-Region PrivateLink</h2> 
<p>Cross-Region PrivateLink addresses the limitations of existing connectivity methods by combining the security and reliability of PrivateLink with global reach. Unlike internet connectivity, it provides consistent low-latency performance over the AWS private network. Unlike in-Region PrivateLink, customers can now access the RTO from any of the 32 AWS Regions without LSEG deploying infrastructure in each location. Furthermore, unlike Delivery Direct, it requires no physical infrastructure deployment, which provides rapid customer onboarding.</p> 
<p>The following diagram (Figure 2) demonstrates the connectivity options available with Cross-Region PrivateLink:</p> 
<p><img alt="Figure 2: RTO connectivity options" class="alignnone wp-image-32665 size-full" height="747" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/04/08/Figure-2-RTO-connectivity-options-1.png" width="1141" /></p> 
<p><strong>Figure 2</strong>: RTO connectivity options</p> 
<p><strong> Supported protocol</strong>: Cross-Region PrivateLink supports the same encrypted and unencrypted protocols as other RTO connectivity methods, including RSSL and WebSocket connections.</p> 
<p><strong>Region support matrix</strong>: LSEG can now provide RTO services across 32 AWS Regions globally, including both default AWS Regions and opt-in AWS Regions. This significantly expands the geographic coverage when compared to the original six-Region deployment without adding overhead or deploying additional resources in each AWS Region.</p> 
<h2>Technical consideration and lessons learned</h2> 
<p>During LSEG’s Cross-Region PrivateLink implementation, several key technical considerations emerged:</p> 
<ul> 
 <li><strong>DNS resolution</strong>: Providing consistent service discovery across AWS Regions required careful coordination between regional DNS configurations and the centralized Service Discovery API.</li> 
 <li><strong>Connection pooling</strong>: High-frequency trading applications needed optimization for connection reuse across cross-Region endpoints to maintain performance.</li> 
 <li><strong>Monitoring and observability</strong>: Cross-Region connectivity required enhanced monitoring to track latency and availability across multiple AWS Regions simultaneously.</li> 
</ul> 
<h2>Performance and network characteristics</h2> 
<p>Cross-Region PrivateLink routes traffic over the AWS private backbone network rather than the public internet.</p> 
<h3>Latency measurements</h3> 
<p>Testing between AWS Regions shows the following network transit times:</p> 
<table border="1"> 
 <thead> 
  <tr> 
   <th>Region Pair</th> 
   <th>Average Latency (ms)</th> 
   <th>95th Percentile (ms)</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>US East <img alt="↔" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2194.png" style="height: 1em;" /> US West</td> 
   <td>70</td> 
   <td>85</td> 
  </tr> 
  <tr> 
   <td>US <img alt="↔" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2194.png" style="height: 1em;" /> EU</td> 
   <td>90</td> 
   <td>110</td> 
  </tr> 
  <tr> 
   <td>US <img alt="↔" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2194.png" style="height: 1em;" /> APAC</td> 
   <td>160</td> 
   <td>190</td> 
  </tr> 
  <tr> 
   <td>EU <img alt="↔" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2194.png" style="height: 1em;" /> APAC</td> 
   <td>170</td> 
   <td>200</td> 
  </tr> 
 </tbody> 
</table> 
<p>These measurements only represent network transit. Total latency includes RTO service processing time and client application overhead.</p> 
<h3>Comparison to public internet connectivity</h3> 
<p>Prior to Cross-Region PrivateLink availability, customers outside of the LSEG deployment Regions connected through public internet. The following table shows the key differences:</p> 
<table border="1"> 
 <thead> 
  <tr> 
   <th>Characteristic</th> 
   <th>Public Internet</th> 
   <th>Cross-Region PrivateLink</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>Routing</td> 
   <td>Variable paths through ISP networks</td> 
   <td>Fixed paths over AWS backbone</td> 
  </tr> 
  <tr> 
   <td>Traffic exposure</td> 
   <td>Crosses public networks</td> 
   <td>Remains on AWS private network</td> 
  </tr> 
  <tr> 
   <td>Configuration</td> 
   <td>Requires VPN or Direct Connect for privacy</td> 
   <td>Direct VPC endpoint connection</td> 
  </tr> 
  <tr> 
   <td>Failover</td> 
   <td>Manual or third-party solutions</td> 
   <td>Automatic across Availability Zones</td> 
  </tr> 
 </tbody> 
</table> 
<p>Characteristic Public internet Cross-Region PrivateLink Routing Variable paths through ISP networks Fixed paths over AWS backbone Traffic exposure Crosses public networks Remains on AWS private network Configuration Requires VPN or Direct Connect for privacy Direct VPC endpoint connection Failover Manual or third-party solutions Automatic across Availability Zones</p> 
<h3>Bandwidth scaling</h3> 
<p>The Cross-Region PrivateLink implementation maintains the same scaling characteristics as in-Region PrivateLink:</p> 
<ul> 
 <li>Bandwidth scales automatically with demand, starting at 10 Gbps per Elastic Network Interface (ENI) and increasing as needed</li> 
 <li>Automatic scaling of bandwidth as traffic increases without manual intervention</li> 
 <li>Support for thousands of concurrent connections per endpoint</li> 
</ul> 
<h3>Data transfer pricing</h3> 
<p>Cross-Region PrivateLink uses standard AWS inter-Region data transfer rates:</p> 
<table border="1"> 
 <thead> 
  <tr> 
   <th>Region Pair</th> 
   <th>Cost per GB</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>Within same Region</td> 
   <td>$0.01</td> 
  </tr> 
  <tr> 
   <td>US East <img alt="↔" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2194.png" style="height: 1em;" /> US West</td> 
   <td>$0.02</td> 
  </tr> 
  <tr> 
   <td>US <img alt="↔" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2194.png" style="height: 1em;" /> EU</td> 
   <td>$0.05</td> 
  </tr> 
  <tr> 
   <td>US <img alt="↔" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2194.png" style="height: 1em;" /> APAC</td> 
   <td>$0.09</td> 
  </tr> 
 </tbody> 
</table> 
<p>Other charges apply: hourly fee per endpoint and per connection hour. Go to the <a href="https://aws.amazon.com/privatelink/pricing/">AWS PrivateLink pricing</a>.</p> 
<h2>Customer success stories</h2> 
<p>Several financial institutions have already implemented Cross-Region PrivateLink for RTO access:</p> 
<p><strong> Global investment bank:</strong> Reduced market data latency by 45% for their APAC trading desk by accessing RTO through Cross-Region PrivateLink from AP-SOUTHEAST-2 to US-EAST-1. This allowed the firm to consolidate infrastructure while maintaining performance requirements for real-time trading applications.</p> 
<p><strong>Hedge fund</strong>: Established a new trading operation in the ME-SOUTH-1 Region without deploying more infrastructure in RTO-enabled AWS Regions. They used the Cross-Region PrivateLink implementation to launch in weeks rather than months, with latency characteristics that met their algorithmic trading requirements.</p> 
<h2>Conclusion</h2> 
<p>LSEG’s implementation of Cross-Region PrivateLink for RTO represents a significant step forward in global market data delivery. Financial institutions can use the AWS global network backbone and PrivateLink technology to access real-time market data from 32 AWS Regions worldwide without compromising security, reliability, or performance. The solution addresses a fundamental challenge in global financial markets: providing consistent, low-latency access to critical market data regardless of geographic location, with institutions able to establish new trading operations in weeks rather than months while maintaining enterprise-grade security and performance standards.</p> 
<p>As financial markets continue to evolve, technologies such as Cross-Region PrivateLink will play a crucial role in enabling institutions to build truly worldwide market data strategies. This transformation allows organizations to focus on competitive differentiation and business growth rather than being constrained by infrastructure limitations and connectivity challenges.</p> 
<h2>About the authors</h2> 
<p>
 <!-- First Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="MikeGranoski.jpg"><img alt="Mike Granoski" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/04/08/MikeGranoski.jpg" width="125" /></p> 
 <h3 class="lb-h4">Mike Granoski (Guest)</h3> 
 <p style="color: #879196; font-size: 1rem;">Mike Granoski is the Director of Real-Time Architecture at LSEG, where he leads the modernization of cloud‑native, low‑latency pricing distribution systems. As Principal Architect for Real-Time Optimized (RTO), he drives innovation in high‑performance market data delivery, focusing on resiliency, throughput, and exceptional client experience.</p> 
</div> 
<p>
 <!-- Second Author --></p> 
<div class="blog-author-box" style="border: 1px solid #d5dbdb; padding: 15px;"> 
 <p class="RohitSingh.png"><img alt="Rohit Singh" class="alignleft wp-image-1288 size-thumbnail" height="125" src="https://d2908q01vomqb2.cloudfront.net/5b384ce32d8cdef02bc3a139d4cac0a22bb029e8/2026/04/08/RohitSingh.png" width="125" /></p> 
 <h3 class="lb-h4">Rohit Singh</h3> 
 <p style="color: #879196; font-size: 1rem;">Rohit Singh is a Principal Solutions Architect at AWS with over 9 years of experience helping enterprise customers design and build cloud-native solutions. He specializes in financial services and capital markets, working with exchanges and trading firms on low-latency architectures, data platforms, and generative AI. Rohit is based in the UK and is passionate about helping organizations modernize their technology to drive innovation and business outcomes.</p> 
</div>
