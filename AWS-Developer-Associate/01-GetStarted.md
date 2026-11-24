
# AWS Global Infrastructure

## 1. How to Choose an AWS Region

An AWS Region is a geographical location where AWS operates its infrastructure.

When launching a new application, consider four main factors:

### 1.1 Compliance
- Follow data governance and legal requirements.
- Data does not leave an AWS Region without explicit permission.
- Choose a Region that meets your organization's regulatory requirements.

### 1.2 Proximity to Customers
- Choose a Region close to your customers.
- Shorter distances generally reduce network latency.
- Lower latency improves application response times.

### 1.3 Available Services
- Not every AWS service is available in every Region.
- New services and features may launch in certain Regions first.
- Verify that your required services are supported before choosing a Region.

### 1.4 Pricing
- AWS pricing varies between Regions.
- Compare costs using AWS service pricing pages.
- Choosing a less expensive Region can reduce infrastructure costs.

**Key Takeaway:**
Choose an AWS Region based on compliance, customer proximity, service availability, and pricing.

---

## 2. AWS Availability Zones (AZs)

An Availability Zone is one or more discrete data centers within an AWS Region.

### Key Concepts
- Each AWS Region contains multiple Availability Zones.
- Regions usually have 3 Availability Zones.
- The slides describe a minimum of 3 and a maximum of 6.
- Each AZ has redundant:
  - Power
  - Networking
  - Connectivity

### Example: Asia Pacific (Sydney)

**Region:** `ap-southeast-2`

**Availability Zones:**
- `ap-southeast-2a`
- `ap-southeast-2b`
- `ap-southeast-2c`

### Why Availability Zones Matter
- Provide infrastructure redundancy.
- Improve application availability.
- Allow applications to run across multiple locations.
- Reduce the risk of an application becoming unavailable due to a single data center failure.

**Key Takeaway:**
A Region is a geographical area containing multiple Availability Zones. Each AZ consists of one or more data centers with redundant infrastructure.

---

## 3. AWS Global Services vs. Regional Services

AWS services can operate globally or within a specific Region.

### 3.1 Global Services

These services operate at a global scope rather than being tied to a single AWS Region.

| Service | Description |
|---------|-------------|
| IAM | Identity and Access Management; manages users, roles, and permissions. |
| Route 53 | DNS service that routes users to applications. |
| CloudFront | Content Delivery Network (CDN) that delivers content to users. |
| AWS WAF | Web Application Firewall that protects web applications. |

### 3.2 Regional Services

Most AWS services are Region-scoped.

| Service | Description |
|---------|-------------|
| Amazon EC2 | Infrastructure as a Service (IaaS); provides virtual servers. |
| Elastic Beanstalk | Platform as a Service (PaaS); simplifies application deployment. |
| AWS Lambda | Function as a Service (FaaS); runs code without managing servers. |
| Amazon Rekognition | Software as a Service (SaaS); provides image and video analysis capabilities. |

### Key Differences

| Global Services | Regional Services |
|-----------------|-------------------|
| Operate globally | Operate within a selected Region |
| Examples: IAM, Route 53 | Examples: EC2, Lambda |
| Not tied to a single Region for their primary scope | Resources are generally created within a specific Region |

**Key Takeaway:**
IAM, Route 53, CloudFront, and WAF are presented as global services. Most AWS services, including EC2 and Lambda, operate at a regional scope.

---

## 4. AWS Certification Review

### Important Concepts to Remember

1. **AWS Region:** A geographical location containing multiple Availability Zones.
2. **Availability Zone:** One or more discrete data centers with redundant power, networking, and connectivity.
3. **Region Selection:** Consider compliance, proximity, available services, and pricing.
4. **Global Services:** IAM, Route 53, CloudFront, and WAF.
5. **Regional Services:** EC2, Elastic Beanstalk, Lambda, and Rekognition.

### Practice Questions

**Q1: What are the four main factors when choosing an AWS Region?**

Answer: Compliance, proximity to customers, available services, and pricing.

**Q2: What is an AWS Availability Zone?**

Answer: One or more discrete data centers with redundant power, networking, and connectivity.

**Q3: Why should an application use multiple Availability Zones?**

Answer: To improve availability, fault tolerance, and infrastructure redundancy.

**Q4: Is Amazon EC2 a global or regional service?**

Answer: Regional.

**Q5: Name three AWS global services.**

Answer: IAM, Route 53, and CloudFront.

**Q6: What is the main benefit of choosing an AWS Region close to customers?**

Answer: Reduced network latency.

**Q7: Are all AWS services available in every Region?**

Answer: No. Service and feature availability varies by Region.
