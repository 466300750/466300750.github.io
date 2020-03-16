---
layout: post
title: AWS_Well-Architected_Framework
date: 2019-04-20 19:42:31
categories: aws
share: y
excerpt_separator: <!--more-->
---


<!--more-->

## 引言
AWS 解决方案架构师在不同的业务垂直领域和应用场景都有多年的架构方案经验，帮助设计和review过AWS上成千上万个客户的架构。因此有一些最佳实践。

AWS Well-Architected Framework 五大支柱

Pillar Name|Description
-----------|-----------Operational Excellence|能够运行并监控交付业务价值的系统，并且持续提升对 processes and procedures的支持.Security|能够在交付业务价值的时候通过风险评估和缓解策略来包含信息、系统和资源.Reliability|系统能够从基础设施或服务中断中恢复，动态获取计算资源来满足需求，并且缓解中断，比如由错误配置或短暂的网络问题.Performance Efficiency|能够高效利用计算资源满足系统要求，并且能随着需求改变和技术演进维持这种高效.Cost Optimization|能够低成本运行系统来交付业务价值.

### Operational Excellence

1. **AWS CloudFormation**
2. 最佳实践：
	- **Prepare**:AWS Config and AWS Config rules
	- **Operate**:Amazon CloudWatch
	- **Evolve**:Amazon Elasticsearch Service(AmazonES)
	
### Security
1. **AWS Identity and Access Management (IAM)**
2. 最佳实践：
	- **Identity and Access Management**： 能够安全控制接入 AWS service 和resource; MFA 在user access 添加了额外一层保护. AWS Organizations 让你对多个 AWS 账号进行集中管理和强制策略.
	- **Detective Controls**: AWS CloudTrail 记录 AWS API calls, AWSConfig 提供了你的 AWS 资源和配置的详细目录. Amazon GuardDuty 是一个 managed threat detection 服务，它持续监测恶意的或者未授权的行为. Amazon CloudWatch 是一个对 AWS 资源进行监测的服务，它可以出发 CloudWatch Events to automate security responses.
	- **Infrastructure Protection**: Amazon Virtual Private Cloud(AmazonVPC) 能够让你在你定义的虚拟网络里启动 AWS 资源. Amazon CloudFront 是一个全球的 content delivery network that securely delivers data, videos, applications, and APIs to your viewers，它集成了 AWS Shield for DDoS mitigation. AWS WAF 网页应用防火墙，它部署在Amazon CloudFront 或者 Application Load Balancer 帮助应用不受常用的web漏洞影响.
	- **Data Protection**: 像 ELB, Amazon Elastic Block Store(AmazonEBS), Amazon S3, and Amazon 关系型数据库服务 (Amazon RDS) 包含加密功能. Amazon Macie 自动发现、分类并保护敏感数据, 同时 AWS Key Management Service (AWS KMS) 让你更加容易地创建和控制加密用的密钥.
	- **Incident Response**: IAM 应该被用于授予合适的授权 incident response teams and response tools. AWS CloudFormation 可以用于创建被信任的环境或者为 conducting investigations 清理空间. Amazon CloudWatch Events 允许你创建规则待触发automated responses including AWS Lambda.

### Reliablity
1. **Amazon CloudWatch**
2. 最佳实践：
	- **Foundations**: AWS IAM、Amazon VPC、AWS Trusted Advisor(它提供服务内部 limits 的可见性)、AWS Shield(它是个 managed 分布式拒绝服务DDoS 保护服务，安全保卫应用跑在 AWS 上).
	- **Change Management**: AWS CloudTrail 记录了你的 account 上的 AWS API 调用，并且传输日志文件给你审计. AWS Config 提供了你的 AWS 资源和配置的详细目录,并且持续记录配置文件的修改. Amazon Auto Scaling 是一个会提供自动化需求管理的服务. Amazon CloudWatch 能够基于metrics 报警，而且有个日志特性，可用于收集日志文件.
	- **Failure Management**: AWS CloudFormation 提供资源创建的模板. Amazon S3 提供了长时间备份服务. Amazon Glacier 提供了长时间的归档服务. AWS KMS 提供了可信的密钥管理系统.

### Performance Efficiency
1. **Amazon CloudWatch**
2. 最佳实践
	- **Selection**
		- **Compute**: Auto Scaling 是确保你有足够的实例来满足需求并且维持响应.
		- **Storage**: Amazon EBS 提供了许多的存储选项(如 SSD 和 provisioned input/output operations per second (PIOPS)) 以优化你的业务场景. Amazon S3 提供 serverless content delivery, 并且 Amazon S3 传输加速使它在远距离传输也快速、简单并且安全.
		- **Database**: Amazon RDS 提供了许多数据库特性(如 PIOPS 和 read replicas) 优化你的业务场景. Amazon DynamoDB 提供了在任何 scale 下秒级别的时延.
		- **Network**: Amazon Route53 提供基于时延的路由.Amazon VPC endpoints and AWS Direct Connect 可以减轻网络距离或者抖动.
	- **Review**: AWS Blog 和 AWS 网页上的 What's New 部分是学习新 feature 和 服务的地方.
	- **Monitoring**: Amazon CloudWatch 提供metrics、报警和通知，而且可以合 AWS Lambda 一起使用触发一些行为.
	- **Trade offs**: Amazon ElastiCache, Amazon CloudFront, AWS Snowball 可以用于提高性能. Amazon RDS 读副本也可以扩展 read-heavy workloads.
	
### Cost Optimization
1. **Cost Explorer**
2. 最佳实践
	- **Expenditure Awareness**: AWS Cost Explorer 可以查看并记录你的使用详情. AWS Budgets 会在你的 usage 或者 spend 超过了实际或者预算金额的时候通知你.
	- **Cost-Effective Resources**: 可以使用 Cost Explorer 的预订实例推荐功能. Amazon CloudWatch 和 Trusted Advisor 来正确 size 你的资源. 可以在 RDS 上使用 Amazon Aurorato 来消除数据库证书花费. AWS Direct Connect 和 Amazon CloudFront 可用于优化数据传输.
	- **Matching supply and demand**: Auto Scaling 避免浪费.
	- **Optimizing Over Time**: AWS News Blog 和 官网上 What's News 部分的查阅. AWS Trusted Advisor 监督你的 AWS 环境并且找机会消除无用的或者空闲的资源或者提交预订资源大小来节约开销.















