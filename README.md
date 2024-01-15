# OAuth2.0_OnAWS

# 1. Access cloud resouces from public
There are so many methoads to access own cloud resouces from public in general. Writing them down with pros and cons.
<br>
I believe that the **Signed URL in AWS CloudFront** is one of the solutions for access from plublic, but I will take up it on another occation. <br>

|  | Method | Pros | Cons | Use Case |
| --- | --- | --- | --- | --- |
| #1 | User's Access Key | - Easy to use. | - Likely to be leaked if it is used in the source code. <br> - If leaked, it can be used by anyone who obtains it, which can potentially compromise your cloud resources and Account itself. | - Shouldn't use because Access key is used as direct access to public API of every cloud resouce which you allow to be controlled. <br> - If you want to use it by some reasons, use it thru environment variables or [opaque type](https://github.com/developer-onizuka/mvc_CosmosDB#3-create-a-secret-resouce-in-kubernetes-cluster). But don't trust it completely. ([Multiple malicious Python packages available on the PyPI repository](https://www.bleepingcomputer.com/news/security/pypi-python-packages-caught-sending-stolen-aws-keys-to-unsecured-sites/)) <br> - You should use web app scanner such as OWASP ZAP during your CI/CD pipeline. |
| #2 | Connection String | - Easy to use. <br> - Safer than User's Access Key. <br> - Define some limitations such as expiration date and IP Address. | - Likely to be leaked if it is used in the source code. <br> - If leaked, it can be used by anyone who obtains it, which can potentially compromise your cloud resources. But it is safer than Access key because it is a connection string for accessing the resources with constraints such as expiration date and accessible IP address. Best practices for SAS is [here](https://docs.microsoft.com/en-us/learn/modules/configure-storage-security/7-apply-best-practices). | - You may use it for some trustworthy person or for temporary. <br> - Internal environment access from onprem server to its organization's own cloud resources. <br> - Use it thru environment variables or opaque type. But don't trust it completely. <br> - You should use web app scanner such as OWASP ZAP during your CI/CD pipeline. <br> - SAS in Azure Storage Account <br> - Pre-Signed URL in AWS S3 |
| #3 | Access Token<br>(OAuth2.0) | - Doesn't need to use secrets such as Access Keys or connection strings in code. | - Difficult to understand how it works. <br> - You need to write a code with some SDKs to get a token from the authorization server's token endpoint. <br> - Default token lifetime ranges between 60-90 minutes. Needs logics toward expiration (But it is not necessary on Managed ID). | - Zero Trust <br> - SaaS subscripution for an unspecified number of customers in general. <br> - [Service principal](https://github.com/developer-onizuka/OAuth2.0_AzureAD#oauth20_azuread) in Azure App registration <br> - Cognito User Pools / Cognito User Identity Pools in AWS|


# 2. How OAuth2.0 works
# **(1) Goals with OAuth2.0** 
- App should get a Token from the Token Endpoint of OAuth2.0's Authorization server to access some specific resouces in the cloud.<br>
- In order to get a Token, App has to send the ClientID (and its Client Secret) to the Token Endpoint of OAuth2.0's Authorization server.<br>
- No ClientIDs should be written in the App because malicious hacker can get it easily and the resouces might have unexpected access from someone.<br>

By the way, there are four types of grant in OAuth2.0. 
```
- Authorization code
- Implicit
- Resource owner password credentials 
- Client credentials
```
But I take **Client credentials** as an example because it is easy for me to explain.

# **(2) Between cloud resources** <br>
You can use AWS Metadata Service for IAM role while the access is between cloud resouces.<br>
AWS's IAM role uses OAuth2.0 Client credentials technology as far as I know. The ClientID in metadata service is already registered by system administrator of the AWS Account. No ClientIDs is written in the App as you can see below: <br>

![metadata_service_AWS.png](https://github.com/developer-onizuka/Diagrams/blob/main/OAuth2.0_Authorization/metadata_service_AWS.drawio.png)

# **(ex) Laundry service in dormitory** <br>
One day, a lazy student in a dormitory uses a housekeeping service to have them wash his dirty clothes. As you can imagine, the service guy does not have any permissions to use washing machine in the dormitory. So the service guy needs to ask dormitory manager a token.<br>
But dormitory manager does not have any tokens in person, neither. Please also note the dormitory manager does not have any knowledges about washing machines. So the dormitory manager asks it for the token manager who manages and maintains washing machines in the dormitory.<br>
Finally, the service guy gets a token and can put it into a washing machine so that he can wash his costomer's dirty clothes. But please note the service guy can not do anything besides using washing machines. This is because the service guy does not have any tokens to use dormitorie's property such as kitchen or bathroom.

![dormitory_AWS.drawio.png](https://github.com/developer-onizuka/Diagrams/blob/main/OAuth2.0_Authorization/dormitory_AWS.drawio.png)


# **(3) From public to cloud resources via AWS Cognito** <br>
You can utilize Cognito User Pools and Identity Pools instead of using Custom Identity Broker Application above.<br> 
Cognito Identity Pools issues a token for each user as a temporary credential. There are two types of managed identities: Authenticated users and Unauthenticated users (guest users). <br>
The Authenticated and unauthenticated user would be given temporary credential (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY and AWS_SESSION_TOKEN) by STS through **AssumeRoleWithWebIdentity** API request with Role ARN, so that the Federated user can get the Role A and an UserID which does not have any roles (and the unauthenticated user can get Role B and an UserID which does not have any roles). <br>
In [3. Temporary security credentials in IAM](https://github.com/developer-onizuka/OAuth2.0_Authorization/blob/main/README.md#3-temporary-security-credentials-in-iam), you can understand the difference between STS APIs which can provide with temporary security credentials such as GetSessionToken and AssumeRoleWithWebIdentity etc.<br>

![AWS_Cognito.drawio.png](https://github.com/developer-onizuka/OAuth2.0_Authorization/blob/main/AWS_Cognito.drawio.png)

> https://aws.amazon.com/jp/blogs/compute/secure-api-access-with-amazon-cognito-federated-identities-amazon-cognito-user-pools-and-amazon-api-gateway/
<br>

![AWS_Cognito_unauthenticated.drawio.png](https://github.com/developer-onizuka/OAuth2.0_Authorization/blob/main/AWS_Cognito_unauthenticated.drawio.png)

# **(4) From onprem with to cloud resources via AWS Directory Service**
AD Connector is designed to give you an easy way to establish a trusted relationship between your Active Directory and AWS. When AD Connector is configured, the trust allows you to:<br>

- Sign in to AWS applications such as Amazon WorkSpaces, Amazon WorkDocs, and Amazon WorkMail by using your Active Directory credentials.
- Seamlessly join Windows instances to your Active Directory domain either through the Amazon EC2 launch wizard or programmatically through the EC2 Simple System Manager (SSM) API.
- Provide federated sign-in to the AWS Management Console by mapping Active Directory identities to AWS Identity and Access Management (IAM) roles.

AD Connector cannot be used with your custom applications, as it is only used for secure AWS integration for the three use-cases mentioned above. **Custom applications relying on your on-premises Active Directory should communicate with your domain controllers directly or utilize AWS Managed Microsoft AD rather than integrating with AD Connector.** <br>

> https://aws.amazon.com/jp/blogs/security/how-to-connect-your-on-premises-active-directory-to-aws-using-ad-connector/

![AWSDirectoryService.png](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img3.png)

Armed with information above, you can create a trust between your Active Directory and AWS. In addition, you now have a quick and simple way to enable single sign-on without needing to replicate identities or deploy additional infrastructure on premises.<br>

# **(5) From Onprem with Hashi-Corp Vault to cloud resources via public IdP's Authentication** <br>
Metadata service is one of dedicated services in AWS EC2 which you can not use in on-premises environment. However, you can easily create a kind of solutions like Metadata service even in on-premises, by using OAuth2.0 with the Hashi-Corp Vault.<br>
In addition, you can use Intel SGX to protect the Key in memory to prevent from compromising caused by some OS vulnerability issues.

![AWS_Cognito_with_Vault.drawio.png](https://github.com/developer-onizuka/Diagrams/blob/main/OAuth2.0_Authorization/AWS_Cognito_with_Vault.drawio.png)

# 3. Temporary security credentials in IAM
You can use the AWS Security Token Service (AWS STS) to create and provide trusted users with temporary security credentials that can control access to your AWS resources. Temporary security credentials work almost identically to long-term access key credentials, with the following differences:

- Temporary security credentials are short-term, as the name implies. They can be configured to last for anywhere from a few minutes to several hours. After the credentials expire, AWS no longer recognizes them or allows any kind of access from API requests made with them.
- Temporary security credentials are not stored with the user but are generated dynamically and provided to the user when requested. When (or even before) the temporary security credentials expire, the user can request new credentials, as long as the user requesting them still has permissions to do so.

As a result, temporary credentials have the following advantages over long-term credentials:

- You do not have to distribute or embed long-term AWS security credentials with an application.
- You can provide access to your AWS resources to users without having to define an AWS identity for them. Temporary credentials are the basis for roles and identity federation.
- The temporary security credentials have a limited lifetime, so you do not have to rotate them or explicitly revoke them when they're no longer needed. After temporary security credentials expire, they cannot be reused. You can specify how long the credentials are valid, up to a maximum limit.

|  | STS API | Goal & Use case | How to work |
| --- | --- | --- | --- |
| #1 | GetSessionToken | IAM User's temporary security credentials <br> - Access with IAM User from untrusted environment with MFA. | 1. Already exists a long-term IAM user (=User-X) in AWS account. <br>2. GetSessionToken Request with MFA token and the identification number of the MFA device associated with the User-X who is making the GetSessionToken call. <br>3. Check if the MFA token is valid. <br>4. Provide User-X's temporary security credentials (ID/Key/Token). |
| #2 | AssumeRole | IAM Role's temporary security credentials <br> - Access with policy not granted to IAM users in different AWS account. | 1. A custom role (=Role-Y) exists in the different AWS account and it is trusted by User-X. <br>2. AssumeRole Request with the User-X's ID as **[SourceID](https://dev.classmethod.jp/articles/update-about-aws-iam-source-identity-attribute/)** and **the ARN of Role-Y**. (But of course, you should login to AWS with your User-X account, before requesting AssumeRole. So it means that **User-X Access Key** had been used implicitly. SourceID is used for **[the confused deputy problem](https://github.com/developer-onizuka/What_is_AzureAD)**.) <br>3. Check if the User-X has a trust relationship. <br>4. Provide Role-Y's temporary security credentials (ID/Key/Token). |
| #3 | GetFederationToken | IAM User's temporary security credentials for Federated Users <br> - Facebook Users can access AWS resources as if Facebook user were the IAM User.| 1. Already exists a long-term IAM user (=User-X) in AWS account. <br>2. GetFederationToken Request with **User-X's Access Key** and IdP's JWT. <br>3. Check if the JWT is valid. <br>4. Provide User-X's temporary security credentials (ID/Key/Token). |
| #4 | AssumeRoleWithWebIdentity | IAM Role's temporary security credentials for Federated Users <br> - Facebook Users can access AWS resources as if Facebook user had the IAM Role. | 1. A custom role (=Role-Y) exists in AWS account. <br>2. AssumeRoleWithWebIdentity Request with IdP's JWT and **the ARN of Role-Y**. <br>3. Check if the JWT is valid. <br>4. Provide Role-Y's temporary security credentials (ID/Key/Token). <br><br> *Instead of directly calling AssumeRoleWithWebIdentity, AWS recommends that you use Amazon Cognito and the Amazon Cognito credentials provider with the AWS SDKs for mobile development.* |

> https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_credentials_temp_request.html <br>
> https://blog.serverworks.co.jp/summary-of-getting-security-credentials-from-sts <br>
> https://www.youtube.com/watch?v=QEGo6ZoN-ao <br>


# 4. Summary
- The point is how we should manage a ClientID which is a kind of secrets to get a Token of cloud resouces thru OAuth2.0. <br>
- AWS IAM role is one of implementation using OAuth2.0. <br>
- As it turns out, AWS IAM role is a kind of system to manage the ClientID securely thru REST API, I believe. <br>
- In the on-prem enviornment, the ClientID would be managed in some reliable Database systems getting along with your App instead of AWS IAM role in cloud. <br>

|  | to On-prem's resource | to Cloud's resource |
| --- | --- | --- |
| **from on-premises** <br> (within the organization)| Connection String | Connection String |
| **from the resources in same cloud subscription** <br> (within the organization)| N/A | OAuth2.0 (Cloud Service Providers provide you with services below) <br> - **AWS IAM role** <br> - Azure ManagedID |
| **from different cloud subscriptions** <br> (within the organization)| Connection String | - [Cross Account Access between AWS accounts](https://n2ws.com/blog/aws-cloud/managing-aws-accounts-cross-account-iam-roles) (but shoud be trusted between AWS accounts) <br> - [Access with Managed ID between Azure subscriptions which each belongs to the same Azure AD tenant.](https://stackoverflow.com/questions/59069065/can-managed-identity-of-a-azure-function-have-access-across-multiple-subscriptio) (But [Managed IDs don't currently support cross-tenant scenarios.](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identities-faq#can-i-use-a-managed-identity-to-access-a-resource-in-a-different-directorytenant)) <br> - [Azure AD External Identities](https://www.youtube.com/watch?v=3Pi_-pHIT_4) for B2B connection. Cross-tenant access setting manages Inbound and Outband traffics for user principals (not for Managed ID principals).<br> - [Authentication flow for external Azure AD users](https://learn.microsoft.com/en-us/azure/active-directory/external-identities/authentication-conditional-access): <br> The most important step is "to see if the resource tenant trusts MFA and device claims (device compliance, hybrid Azure AD joined status) from external tenant's users." <br> If it is not, then "B2B collaboration users are prompted for MFA, which they need to satisfy in the resource tenant."|
| **from anonymous public resources thru RESTful API / GraphQL API** <br> (outside the organization)| OAuth2.0 (Must make the scheme on your own) | OAuth2.0 (Cloud Service Providers provide you services below) <br> - **AWS Cognito User Pools / AWS Cognito Identity Pools** <br> - Azure Service Principal |


