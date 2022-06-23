
# Example Permissions Boundaries

This repository contains a sample IAM permissions boundary as a starting point for creating your own permissions boundary to meet the security needs of your organization. These IAM permissions boundary samples, when attached to an IAM role, allow it to perform all expected workload tasks without being able to modify the security of its environment.

This policy sample is just an example, and may allow for more access than you intend your application roles to have. You should remove any unnecessary entitlements for your applications in the spirit of least-privileged access.

## About Permissions Boundaries

Permissions boundaries are policies that can be attached to IAM users and roles that define their maximum entitlements. IAM permissions boundaries can only deny entitlements with either an implicit (entitlement is not present in the permissions boundary) or explicit (there is a deny statement in the permissions boundary) deny, and cannot be used to grant an entitlement.

To learn more about permissions boundaries, refer to the AWS documentation link below.

## The use case for Permissions Boundaries

Organizations want to empower developers to take self service actions in AWS to build with more speed and agility. However, granting the ability to create and modify IAM roles and policies may lead to a developer raising their level of privilege for themselves or their applications beyond what was intended.

IAM permissions boundaries can be enforced when creating and modifying IAM roles through IAM conditions to ensure that the role cannot be modified with a level of entitlement beyond what is specified in the permissions boundary. Typically, the permissions boundary contains actions that an application may take, like s3:GetObject, but not operations that would allow a role to modify the security of its own environment such as ec2:AuthorizeSecurityGroupEgress. 

## Using Permissions Boundaries

Permissions boundaries should primarily be used to enable builders self-service access to create, modify, and update IAM principals and policies without escalating their level of entitlements in an AWS environment beyond what is intended.

This is typically enforced with a service control policy or identity policy targeting roles used by builders or CI/CD pipelines for applications. For example, the following policy restricts only creating, modifying, and updating of roles with a specified policy attached as a permissions boundary, such as:


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

Builders subject to the above Identity or Service Control Policy will be forced to only create or modify roles that have the policy "permissionboundarypolicy" set as the permissions boundary, and will not be able to modify the policy used as a permissions boundary or their role's own policies.

Wildcarding the account number with '*' in the "Resource" element of an IAM policy is safe for actions in the "iam:" namespace, as IAM resources can only ever be modified within the same AWS account. This means the above policy examples can be used without updating the ARNs for a specific account ID. Wildcarding account numbers in other namespaces/resources may grant entitlement to resources in other AWS accounts if they are setup for cross account access.

## Using paths to constrain access

As a best practice, roles used by builders to take actions in self service should only be entitled to create or modify IAM resources with specific paths as a means of ensuring they do not modify resources that are not used by their workloads, such as those belonging to security or infrastructure teams. The following allow statements only permit for builders to create/modify policies in the 'applicationpolicies' path, create/modify/pass roles in the 'applicationroles' path, and create/modify instance profiles in the applicationinstanceprofiles path.

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

Using the above examples to ensure that builders are always using permissions boundaries when modifying IAM roles, and only ever modifying IAM resources in specific paths can be used to safely delegate IAM tasks to builders in an AWS account.

## This Repository

This policy contains two samples policies, and a cloud formation template to deploy each:

1. VerbosePermissionsBoundary . This example contains a full list of actions with no wildcards present, and the policy is separated out into different statements based on the different usecases
2. MinifiedPermissionsBoundary . This example has all the same entitlements, however wildcarding has been liberally applied and all the statements are condensed into one, optimizing for character space.

The policies policy documents can be found in the policies/ folder in their respectively named .json file

The Cloud Formation template for each can be found in the /cloudformation folder in a their respectively named .json file . The Cloud Formation template will create policies named "VerbosePermissionsBoundary" and "MinifiedPermissionsBoundary".
## Documentation Links

https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html
