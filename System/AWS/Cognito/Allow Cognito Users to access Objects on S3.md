To allow users in your Cognito User Pool to access objects in an S3 Bucket (without allowing public access on the Bucket that is), users must be granted an IAM role that gives them the permission to do so.
In AWS this is done via Cognito Federated Identities. Users from whitelisted identity providers can be granted access via a *federated Sign-In*, controlled by a Trust Policy. This works roughly as follows:
1. Users authenticate against their identity provider and are given a **token**.
2. Users take this token to Cognito Federated Identities and request access to AWS with it, via an *Identity Pool*.
3. Cognito will check the configured IAM Role inside the Identity Pool, specifically it's *Trust Policy*, against the given token.
4. If the Trust Policy allows access, the user is then granted temporary credentials to access AWS resources, based on the IAM Role's Policies.

An example Trust Policy would look like this:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "cognito-identity.amazonaws.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "cognito-identity.amazonaws.com:aud": "<aws region>:<uuid>"
                },
                "ForAnyValue:StringLike": {
                    "cognito-identity.amazonaws.com:amr": "authenticated"
                }
            }
        }
    ]
}
```
There are a few things going on here:
- `Principal.Federated` is set to `cognito-identity.amazonaws.com` to specify Cognito as a service that can assume this role.
- `Action` needs to be `sts:AssumeRoleWithWebIdentity`, which basically states that a user can assume this IAM Role.
- `Condition.StringEquals.cognito-identity.amazonaws.com:aud` matches the `aud` key in the token against your identity pool - assuring that only users from the given pool can assume this role
- `Condition.ForAnyValue:StringLike.cognito-identity.amazonaws.com:amr` matches `authenticated` to ensure users requesting access have authenticated with an identity provider.

# Fine-Grained Access to Services
Granting Access to general Services on your AWS Account might be fine for your use case, but can be limiting if you want to give access only to specific resources.
However, Cognito has just the thing for you: It can map attributes from the token's payload as IAM variables for resolution. This allows you to define Policies that include the user's principal id or username, for example.
The documentation on AWS is [here](https://docs.aws.amazon.com/cognito/latest/developerguide/attributes-for-access-control.html), but here's a short rundown:
- Attribute mapping needs to be enabled in the Identity Pool's settings. You can either use the documented default mappings or use a custom mapping scheme.
- The mapped attributes exist as `${aws:PrincipalTag/<attribute>}`

It's not specified in the documentation, but the mapping uses the IdToken's payload. This is important, because Cognito User Pools set their attributes with a `cognito:` prefix, so if you want to map, for example, the username, it's called `cognito:username`.
Additionally, this means you can inject additional parameters with a `Pre Token Generation`-triggered lambda function.

# Granting Users Access to Personal S3 Resources
Finally, we have everything we need to grant granular access for our users to an S3 bucket. I'll just jump right into the Policy and explain afterwards:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListYourObjects",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": [
                "arn:aws:s3:::my-bucket"
            ]
        },
        {
            "Sid": "ReadEverything",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket/*"
            ]
        }
        {
            "Sid": "WriteDeleteYourObjects",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket/users/${aws:PrincipalTag/username}",
                "arn:aws:s3:::my-bucket/users/${aws:PrincipalTag/username}/*"
            ]
        }
    ]
}
```
With this Policy a user can:
- list and read all objects in the bucket
- write and delete only under his own prefix.

## Uniqueness
In my example, I'm using the username as the prefix for clarity's sake. I can do this because I'm not expecting more identity providers than my own Cognito User Pool, so the username is guaranteed to be unique.
If your Identity Pool combines multiple identity providers, you cannot assume the sub or username will be unique across all of them, so you can't use them for identification. In that case you can use `cognito-identity.amazonaws.com:sub` as the principal identifier, which represents the `subject` as known by the identity pool. It's guaranteed to be unique inside the identity pool and will refer always to the same user, but it's not easily traced back to the original authenticated user.

## Username
If all you need is the username from a Cognito user, this can also be achieved by using `aws:username`. The [documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html#policy-vars-infotouse) states it's not filled for federated users, but at least for Cognito users it seems to be set to the `username` field as intended.[^1]
As this is undocumented behavior I have no idea how stable this is though.

[^1]: https://www.chaosgears.com/post/enabling-amazon-cognito-identity-pools-and-aws-iam-to-perform-attribute-based-access-control