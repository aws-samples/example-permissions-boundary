# Example Permissions Boundaries

This repository contains two example IAM permissions boundary policies as a starting point for creating your own permissions boundary to meet the security needs of your organization. The IAM permissions boundary sample, when attached to an IAM principal, allow it to perform the tasks typically associated with an application role without being able to modify the security of its environment. This read me also contains permissions boundary best practices, and guidance on how to use service control policies to help enforce permissions boundaries are used.

These permissions boundary policies are just an example, and may allow for more access than you intend your application roles to have. You should remove any unnecessary entitlements for your applications in the spirit of least-privileged access.

## About Permissions Boundaries

Permissions boundaries are IAM policies that can be attached to IAM principals (users and roles), and limit the maximum permissions of the permissions. IAM permissions boundaries can only deny entitlements with either an implicit (entitlement is not present in the permissions boundary) or explicit (there is a deny statement in the permissions boundary) deny, and cannot be used to grant an entitlement. 

Permissions boundaries are an IAM policy defined in the just same way as other IAM policies, however when used as a permission boundary the policy will apply constraints to the principal to which they are attached.

Permissions boundaries simplify the journey to least privilege by defining what is the maximum possible privilege.

To learn more about permissions boundaries, refer to the AWS documentation at the end of this README.

## The use case for Permissions Boundaries

Companies want to empower their builders to take self service actions in AWS to build with more speed and agility. However, granting the ability to create and modify IAM roles and policies can lead to a builder gaining more privilege for themselves or their applications beyond what was originally intended.

For example when a builder wants to deploy an AWS Lambda function, a corresponding IAM role is required to be defined as the Lambda function’s execution role. If a developer has permissions to create a new IAM role and policy for that role, they could specify any permission in it's IAM policies. However, if the developer is forced to use a permissions boundary, the permissions of the Lambda function's role cannot exceed that of the permissions boundary. Using the sample permissions boundary provided in this repo as a boundary on that Lambda function's role, the lambda could take actions like read data from s3, but not modify security groups or attach internet gateways.

Permissions boundaries provide a safe and scalable mechanism for an organization to delegate IAM self service actions to their builders, and to help ensure that the created IAM roles adhere to their defined security requirements.

Permissions boundaries are the IAM feature for providing builders self-service access to create, modify, and update IAM principals and policies without elevating their privilege beyond what is defined in the permission boundary. This allows companies to delegate some level of IAM self service to developers, and expedite the IAM role and policy creation process as the persons who are building the application have the ability to define it's IAM role.

## Permissions Boundaries Best Practices


#### Do not put resources in permissions boundaries policies

Permissions boundaries are best for coarse grained access control, limiting what IAM actions can be performed by the IAM role they’re attached to. Least privileged access to specific AWS resources, such as S3 buckets of KMS keys should be managed in resource or IAM policy. Having a wildcard (“*”) in the resource element of a permissions boundary policy does not grant access to all resources, or any resource. This only allows permissions to be granted by other policy types that are capable of granting access (IAM, and resource policies).

#### Only use allow statements in permissions boundaries

Use only allow statements in your permissions boundary policies. Any IAM action that is not in the allow statements of your permissions boundary will be implicitly denied, and you can deny many more actions using the allow plus implicit deny approach rather than explicitly denying actions while granting access to others in the permissions boundary.

#### Avoid using conditions in permissions boundaries

Access conditions, such as those based on source IP address, or Virtual Private Cloud (VPC) ID of the request are best placed in other policy types. If you have a common set of access you want to allow, or deny based on a condition across multiple IAM roles, we recommend using Service Control Policy or Resource Policy to apply that restriction.

#### Avoid using a unique permissions boundary per IAM role

Permissions boundaries are ideally reusable across many roles within an AWS account. Managing a unique permissions boundary per IAM Role makes permissions boundary enforcement challenging to scale with service control policy, and introduces additional IAM complexity without a clear benefit. 

If you want to limit the maximum permissions further than just using one permissions boundary, you may consider creating multiple permissions boundary policies that align with different types of applications you may run and have your builders choose the one that is best suited for their application. An example of this would be different permissions boundaries for the different tiers of a 3 tier webapp, and the permissions boundary associated with the IAM role of the EC2 instances of the front end/presentation tier do not allow access to data services, but do allow operational utilities like cloudwatch logging and systems manager. The backend permissions boundary would allow for the possibility of access to be granted to data storage services, such as RDS or S3.


## Using permissions boundaries

Typically a permissions boundary policy contains actions that an created role may perform, like s3:GetObject, but not operations that would allow a role to modify the security of its own environment such as ec2:AuthorizeSecurityGroupEgress. 

When using permissions boundaries, it is helpful to think in terms of three typical personas and their who participate in the role and policy creation process:

- The administrator or cloud operator: Defines the permissions boundary policies and the policies attached to the builder's roles
- The builder: Creates subsequent principals for their applications to use. The builder is required to attach permissions boundary policies to all roles they create.
- The application: Uses roles created by the builder. Application principals have the permissions boundary policy attached by the builder.

 **_NOTE:_**  The role used by the builder persona may be used by the builders directly, or may be a role used by the builders CI/CD process.


![A diagram explaining permissions boundaries](permissions_boundary.png)

To summarize:

- The administrator or cloud operator persona defines a permissions boundary policy
- The builder persona creates roles, and attaches the permissions boundary policy to application roles they create
- The application persona is restricted by the contents of the permissions boundary policy

The following sections will refer to these personas for simplicity.

### Ensuring that the permissions boundary policies are used

IAM permissions boundaries, created by the administrator persona, can be enforced during the creation and modification of IAM roles with IAM conditions in policy. 

When attached to the builder persona, the following IAM policy helps ensure that the builder can create, modify, and update roles, *but only* when a specified permissions boundary policy is attached to the new role.


```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceActionsHaveBoundary",
      "Effect": "Deny",
      "Action": [
        "iam:AttachRolePolicy",
        "iam:CreateRole",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "iam:PermissionsBoundary": "arn:aws:iam::*:policy/permissionboundarypolicy"
        }
      }
    },
    {
      "Sid": "DenyChangesToBoundaryPolicy",
      "Effect": "Deny",
      "Action": [
        "iam:DeletePolicy",
        "iam:CreatePolicyVersion",
        "iam:CreatePolicy",
        "iam:DeletePolicyVersion",
        "iam:SetDefaultPolicyVersion"
      ],
      "Resource": "arn:aws:iam::*:policy/permissionboundarypolicy"
    }
  ]
}
```
With this policy attached, the builder persona is required to attach arn:aws:iam::*:policy/permissionboundarypolicy whenever they create roles or specify policies to attach to a role for use by their application. The builder is also unable to make changes to arn:aws:iam::*:policy/permissionboundarypolicy, the permissions boundary policy itself.

Replacing the account number with a wildcard (“*”) in the Resource element of an IAM policy is safe for actions in the iam: namespace because IAM resources can be modified only in the same AWS account. This means the preceding policy example can be used without updating the ARNs for a specific account ID. Using wildcards in place of account numbers in other namespaces or resources might grant entitlements to resources in other AWS accounts if they are configured for cross account-access.

### Using IAM paths to constrain access

As a best practice, roles used by builders to take self-service actions should only be allowed to create or modify IAM resources with specific IAM paths. This approach helps to ensure these actions do not modify resources that are not used by their workloads, such as those belonging to security or infrastructure teams. The following Allow statements permit builders only to create and modify policies in the applicationpolicies path; create, modify, and pass roles in the applicationroles path; and create and modify instance profiles in the applicationinstanceprofiles path.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OnlyPoliciesInTheirPath",
      "Effect": "Allow",
      "Action": [
        "iam:CreatePolicy",
        "iam:CreatePolicyVersion",
        "iam:DeletePolicy",
        "iam:SetDefaultPolicyVersion"
      ],
      "Resource": "arn:aws:iam::*:policy/applicationpolicies/*"
    },
    {
      "Sid": "AllowRolesOnlyInPath",
      "Effect": "Allow",
      "Action": [
        "iam:AttachRolePolicy",
        "iam:CreateRole",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:PutRolePermissionsBoundary",
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::*:role/applicationroles/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:DeleteInstanceProfile"
      ],
      "Resource": [
        "arn:aws:iam::*:role/applicationroles/*",
        "arn:aws:iam::*:instance-profile/applicationinstanceprofiles/*"
      ]
    }
  ]
}
```

For more information about IAM paths, see the AWS documentation links at the end of this README.

### Policy summary

The two example policies attached to our builder persona ensure that builders:

Are always required to use predefined permissions boundary policies when creating or modifying IAM roles used by an application.
Are able to modify IAM resources only in specific paths.

The combination of these two achieves the goal of safely delegating IAM tasks to builders in an AWS account.


## This repository

This repository contains two example policies and an AWS CloudFormation template to deploy each policy:

1. VerbosePermissionsBoundary . This example policy includes a full list of actions with no wildcards present. The policy is separated into different statements based on different use cases.
2. MinifiedPermissionsBoundary . This example policy includes all the same entitlements as the first example policy; however, wildcards have been used liberally, and all the statements are condensed into one, optimizing for character space.

You can find the raw IAM policy documents in the policies/ folder in the respectively named .json file.

You can find an AWS CloudFormation template for each policy can be found in the /cloudformation folder in a their respectively named .json file . The AWS CloudFormation template will create policies named "VerbosePermissionsBoundary" and "MinifiedPermissionsBoundary".

## Documentation links

https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html

https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html#identifiers-arns
