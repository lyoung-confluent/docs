# Setup
## commonfate/aws-sso@v2
:::info
When setting up a provider for your deployment, we recommend using the [interactive setup workflow](../../../interactive-setup.md) which is available from the Providers tab of your admin dashboard.
:::
## Example granted_deployment.yml
```yaml
version: 2
deployment:
  stackName: example
  account: "12345678912"
  region: ap-southeast-2
  release: v0.12.0
  parameters:
    CognitoDomainPrefix: example
    AdministratorGroupID: granted_administrators
    ProviderConfiguration:
      aws-sso-v2:
        uses: commonfate/aws-sso@v2
        with:
          identityStoreId: ""
          instanceArn: ""
          region: ""
          ssoRoleArn: ""
          ssoSubdomain: ""

```
## Find the AWS SSO instance details
### Configuration Fields
This step will guide you through collecting the values for these fields required to setup your provider.

| Field | Description |
| ----------- | ----------- |
| identityStoreId | the AWS SSO Identity Store ID |
| instanceArn | the AWS SSO Instance ARN |
| region | the region the AWS SSO instance is deployed to |
| ssoSubdomain | the custom SSO subdomain configured (if applicable) |
### Using the AWS CLI

If you have the AWS CLI installed and can access the account that your AWS SSO instance is deployed to, run the following command to retrieve details about the instance:

```bash
❯ aws sso-admin list-instances
{
    "Instances": [
        {
            "InstanceArn": "arn:aws:sso:::instance/ssoins-1234567890",
            "IdentityStoreId": "d-1234567890"
        }
    ]
}
```

The **InstanceArn** value in the CLI output should be provided as the **instanceArn** parameter when configuring the provider.

The **IdentityStoreId** field in the CLI output should be provided as the **identityStoreId** parameter when configuring the provider.

If your AWS SSO instance is deployed in a separate region to the region that Common Fate is running in, set the **region** parameter to be the region of your AWS SSO instance (e.g. 'us-east-1').

### Using the AWS Console

Open the AWS console in the account that your AWS SSO instance is deployed to. If your company is using AWS Control Tower, this will be the root account in your AWS organisation.

Visit the **Settings** tab. The information about your SSO instance will be shown here, including the Instance ARN (as the “ARN” field) and the Identity Store ID.

![](https://static.commonfate.io/providers/aws/sso/console-instance-arn-setup.png)

### Custom SSO Subdomain
This optional field allows to specify a [custom AWS access portal URL](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtochangeURL.html).

This configuration can be left blank and will default to using AWS's default subdomain for your IAM Identity Store instance.
## Create an IAM role
### Configuration Fields
This step will guide you through collecting the values for these fields required to setup your provider.

| Field | Description |
| ----------- | ----------- |
| ssoRoleArn | The ARN of the AWS IAM Role with permission to administer SSO |
The AWS SSO provider requires permissions to manage your SSO instance.

The following instructions will help you to setup the required IAM Role with a trust relationship that allows only the Common Fate Access Handler to assume the role.

This role should be created in your AWS management account. This is the account where AWS SSO is configured and your AWS Organization is managed.

Copy the following YAML and save it as 'common-fate-access-handler-sso-role.yml'.

We recommend saving this alongside your common-fate-deployment.yml file in source control.

```yaml
Resources:
  CommonFateAccessHandlerSSORole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          - Action: sts:AssumeRole
            Effect: Allow
            Principal:
              AWS: "{{ Access Handler Execution Role ARN }}"
        Version: "2012-10-17"
      Description: This role grants management access to AWS SSO for the Common Fate Access Handler.
      Policies:
        - PolicyName: AccessHandlerSSOPolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: ReadSSO
                Action:
                  - iam:GetRole
                  - iam:GetSAMLProvider
                  - iam:ListAttachedRolePolicies
                  - iam:ListRolePolicies
                  - identitystore:ListUsers
                  - organizations:DescribeAccount
                  - organizations:DescribeOrganization
                  - organizations:ListAccounts
                  - organizations:ListAccountsForParent
                  - organizations:ListOrganizationalUnitsForParent
                  - organizations:ListRoots
                  - organizations:ListTagsForResource
                  - sso:DescribeAccountAssignmentCreationStatus
                  - sso:DescribeAccountAssignmentDeletionStatus
                  - sso:DescribePermissionSet
                  - sso:ListAccountAssignments
                  - sso:ListPermissionSets
                  - sso:ListTagsForResource
                Effect: Allow
                Resource: "*"
              - Sid: AssignSSO
                Action:
                  - iam:UpdateSAMLProvider
                  - sso:CreateAccountAssignment
                  - sso:DeleteAccountAssignment
                Effect: Allow
                Resource: "*"
Outputs:
  RoleARN:
    Value:
      Fn::GetAtt:
        - CommonFateAccessHandlerSSORole
        - Arn
```

### Using the AWS CLI

If you have the AWS CLI installed and can deploy cloudformation you can run the following commands to deploy this stack.
Ensure you have credentials for the same account that Common Fate is deployed to and that AWS_REGION environment variable is set correctly, we recommend deploying this role to the same region as your Common Fate stack.

```bash
aws cloudformation deploy --template-file common-fate-access-handler-sso-role.yml --stack-name Common-Fate-Access-Handler-SSO-Role --capabilities CAPABILITY_IAM
```

Once the stack is deployed, you can retrieve the role ARN by running the following command.

```bash
aws cloudformation describe-stacks --stack-name Common-Fate-Access-Handler-SSO-Role --query "Stacks[0].Outputs[0].OutputValue"
```

### Using the AWS Console

Open the AWS Console in your organisation's management account and click **Create stack** then select **with new resources (standard)** from the menu.

![](https://static.commonfate.io/providers/aws/sso/create-stack.png)

Upload the template file

![](https://static.commonfate.io/providers/aws/sso/create-stack-with-template.png)

Name the stack 'Common-Fate-Access-Handler-SSO-Role'

![](https://static.commonfate.io/providers/aws/sso/specify-stack-details.png)

Click **Next**

Click **Next**

Acknowledge the IAM role creation check box and click **Create Stack**

![](https://static.commonfate.io/providers/aws/sso/accept-iam-prompt.png)

Copy the **RoleARN** output from the stack and paste it in the **ssoRoleArn** config value on the right.

![](https://static.commonfate.io/providers/aws/sso/role-output.png)

### Restricting access to particular Permission Sets

The CloudFormation template above will give the Common Fate Access Handler access to all Permission Sets in your AWS SSO instance. If you wish to further restrict this, replace the `AssignSSO` policy statement in the CloudFormation YAML with the following:

```
- Sid: AssignSSO
  Action:
    - sso:CreateAccountAssignment
    - sso:DeleteAccountAssignment
    - sso:DescribePermissionSet
    - organizations:DescribeAccount
  Effect: Allow
  Resource:
    - arn:aws:sso:::account/*
    - <SSO_INSTANCE_ARN>
    - <PERMISSION_SET_ARN_1>
    - <PERMISSION_SET_ARN_2>
```

Where:

- `<SSO_INSTANCE_ARN>` is the ARN of your AWS SSO instance from Step 1.
- `<PERMISSION_SET_ARN_1>`, `<PERMISSION_SET_ARN_2>` and so forth are the ARN of the Permission Sets to give Common Fate access to.

You can further restrict Common Fate's access to only provision permission sets in particular accounts. To do so, replace `arn:aws:sso:::account/*` with the ARNs of the specific account IDs you'd like Common Fate to access.

### Permitting Access to Assign Permission Sets for the Management Account

The CloudFormation template above will give the Common Fate Access Handler access to assign users to all accounts in your organization except for the management account.
When the SSO provider create an account assignment, [AWS Service Linked Role](https://docs.aws.amazon.com/singlesignon/latest/userguide/using-service-linked-roles.html) has permission to create a role for the permission set in the account which the user is able to assume. However, this service role does not have permission to create roles in the management account.

If you want to manage access to the management account through Common Fate, add the following statement to the IAM policy cloudformation template:

```
- Sid: AssignManagementAccountSSO
  Effect: Allow
  Action:
    - iam:CreateRole
    - iam:AttachRolePolicy
  Resource: arn:aws:iam::*:role/aws-reserved/sso.amazonaws.com/*
  Condition:
    StringEquals:
      aws:PrincipalOrgMasterAccountId: "${aws:PrincipalAccount}"
```

This policy is restricted to creating SSO roles in the management account only and is adapted from the [AWS Service Linked Role](https://docs.aws.amazon.com/singlesignon/latest/userguide/using-service-linked-roles.html) policy.
