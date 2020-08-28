# Using aws cli with multi-factor authentication (MFA)
AWS has a great console where it is easy to use multi-factor authentication (MFA) after it has been configured for the user. With this tutorial you can also use MFA with aws commandline tools. To do this AWS uses Secure Token Service, which allows assume-role with MFA. This process helps create a much more secure way to use Access Keys on a remote machine. Even if the key is compromised, it is almost impossible to use without the MFA device and the knowledge of the role that the IAM user is allowed to access using the access key. To take advantage of this process follow these steps:

### Creating User
This section is based on the [Policies for Delegating Access](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_policy-examples.html) from AWS.

### Create IAM Policy 
Create an IAM policy and give it the desired access permissions. In this example I have given the user access to write to a s3 bucket. 

- Policy to access to one of your buckets, examplebucket, and allow the user to add, update, and delete objects. Amazon has some great [sample policies](https://docs.aws.amazon.com/AmazonS3/latest/dev/example-policies-s3.html) to check out.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::examplebucket",
                "arn:aws:s3:::examplebucket/*"
            ]
        }
    ]
}
```
### Create IAM Role 
Create an IAM Role and give it the desired access. In this example I have attached the policy created in the previous step - write to a s3 bucket.

After we create the IAM User, which we will use to assume the role created above, we have to update the Trust Relationship for the role to allow the IAM User to use sts:AssumeRole.

#### Trust Relationship Policy for the role:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MYACCOUNTID:user/myIAMUser"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
### Create IAM User
Create an IAM user with access to assume a role and assign policies to allow it to assume a role with MFA. You can create the user to not have console access. Change the MFA policy accodingly. This user will need Programmactic Access. Make sure to save the Access Key and SecretAccessKey as these will be needed to assume role from the cli. Refer [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) on how to create IAM Users.

1. Policy to Assume Role
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Resource": "arn:aws:iam::MYACCOUNTID:role/role-to-assume"
        }
    ]
}
```
2. Policy to enforce using MFA for the IAM user
 Refer [AWS Tutorial](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_users-self-manage-mfa-and-creds.html). The tutorial has a very wide policy, but I prefer to use a more restrictive policy that allows the user to only make changes to their account. This can be more restrictive for IAM users that do not have console access.
 ```json
 {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAllUsersToListAccounts",
            "Effect": "Allow",
            "Action": [
                "iam:ListAccountAliases",
                "iam:ListUsers"
            ],
            "Resource": [
                "arn:aws:iam::MYACCOUNTID:user/*"
            ]
        },
        {
            "Sid": "AllowIndividualUserToSeeTheirAccountInformation",
            "Effect": "Allow",
            "Action": [
                "iam:ChangePassword",
                "iam:CreateLoginProfile",
                "iam:DeleteLoginProfile",
                "iam:GetAccountPasswordPolicy",
                "iam:GetAccountSummary",
                "iam:GetLoginProfile",
                "iam:UpdateLoginProfile"
            ],
            "Resource": [
                "arn:aws:iam::MYACCOUNTID:user/${aws:username}"
            ]
        },
        {
            "Sid": "AllowIndividualUserToListTheirMFA",
            "Effect": "Allow",
            "Action": [
                "iam:ListVirtualMFADevices",
                "iam:ListMFADevices"
            ],
            "Resource": [
                "arn:aws:iam::MYACCOUNTID:mfa/*",
                "arn:aws:iam::MYACCOUNTID:user/${aws:username}"
            ]
        },
        {
            "Sid": "AllowIndividualUserToManageThierMFA",
            "Effect": "Allow",
            "Action": [
                "iam:CreateVirtualMFADevice",
                "iam:DeactivateMFADevice",
                "iam:DeleteVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:ResyncMFADevice"
            ],
            "Resource": [
                "arn:aws:iam::MYACCOUNTID:mfa/${aws:username}",
                "arn:aws:iam::MYACCOUNTID:user/${aws:username}"
            ]
        },
        {
            "Sid": "DoNotAllowAnythingOtherThanAboveUnlessMFAd",
            "Effect": "Deny",
            "NotAction": "iam:*",
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:MultiFactorAuthAge": "true"
                }
            }
        }
    ]
}
 ```
### Using STS AssumeRole
There are multiple ways to use sts:AssumeRrole option. You have to enter the MFA serial-number the first time when you enter the cli with your command. All the following commands do not require the MFA key as allowed in the Role configuration duration. For the rest of the session you will not be required for the serial-number. Refer [documentation](https://docs.aws.amazon.com/cli/latest/reference/sts/get-session-token.html) for details.

#### Using role-arn in credentials file
Configure the ~/.aws/credentials file with the following key-value pairs. Post configuration call the commands allowed for the role in the policy. With this option, you do not have to run a command to get the temporary acccess key and save it some place every time you want to start using the cli.
>Note: This is my preferred method to work in the cli.

*Open ~/.aws/credentials file in your preffered text editor and add the following lines to it.*
``` ini
[iamuser]
aws_access_key_id = THEACCESSKEYIDFROMAWS
aws_secret_access_key = TheSecretAccessKeyYouGetfromAWSWhenSettingUPTheIAMUser

[default]
role_arn = arn:aws:iam::MYACCOUNTID:role/my-iam-role
source_profile = iamuser
mfa_serial = arn:aws:iam::MYACCOUNTID:mfa/myIAMUser 
```
The ***iamuser*** profile contains the access key for the myIAMUser configured in AWS console earlier. This access key only provides the user to execute the sts AssumeRole command and perform the tasks specified in the policy attached to the role.

Execute the allowed aws command with the profile that assumes the role. You will be prompted for the MFA key.

__Command Examples__:
```bash
aws s3 ls examplebucket 
aws s3 cp ./helloworld.txt s3://examplebucket/
```


#### Use IAM Role in the cli with the role_arn
There are times when you want to save the token in your credentials file. Most of this is done when you want to use the Access Key with a different software that can read credential file or with scripts that you wrote in boto3 etc. Use the following command to assume the role You can specify the duration you want for the access key to be valid.

*Open ~/.aws/credentials file in your preffered text editor and add the following lines to it.*
``` ini
[iamuser]
aws_access_key_id = THEACCESSKEYIDFROMAWS
aws_secret_access_key = TheSecretAccessKeyYouGetfromAWSWhenSettingUPTheIAMUser
```

 __Command Format__:
```bash
aws sts assume-role
  --role-arn <arn of the iam role to assume> 
  --role-session-name <identifier value>
  --serial-number <arn of the mfa device> 
  --token-code <code from mfa token>
  [--duration-seconds <value>]
  --profile <profile of the role_arn from credentials file>
```
 __Command Example__:
```bash
aws sts assume-role --role-arn arn:aws:iam::MYACCOUNTID:role/my-iam-role --role-session-name "mySession1" --serial-number arn:aws:iam::MYACCOUNTID:mfa/myIAMUser --duration-seconds 3600 --token-code 123456 --profile iamuser
```

Executing the above command will return you credentials for the duration requested. Note the Expiration. Default is 1 hour for IAM Roles, which can be set in the console. If you set the duration to be larger than the duration set in the role then you will get an error.
```json 
{
    "Credentials": {
        "AccessKeyId": "ASIAS4ZCF3BG6MVAIHDC",
        "SecretAccessKey": "RFclxB2Q23aBSKGf8fusfZ99V7Y+Cs83bnfR4pat",
        "SessionToken": "IQoJb3JpZ2luX...<remainder of token>",
        "Expiration": "2020-04-26T08:30:59+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAS4ZCF3BGRDXR7CEDN:mySession1",
        "Arn": "arn:aws:sts::MYACCOUNTID:assumed-role/my-iam-role/mySession1"
    }
}
```
Save it in the credentials file located in ~/.aws/ directory. 

Refer: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html
```ini
[default]
output = json
region = us-east-1
aws_access_key_id = ASIAS4ZCF3BGWAMFF2PF
aws_secret_access_key = RFclxB2Q23aBSKGf8fusfZ99V7Y+Cs83bnfR4pat
aws_session_token = IQoJb3JpZ2luX...<remainder of token>
```
After setting up the credentials file as above the permitted commands can be executed from cli.

__Command Examples__:
```bash
aws s3 ls examplebucket 
aws s3 cp ./helloworld.txt s3://examplebucket/
```
