---
templateKey: blog-post
title: Managing Secrets and Parameters on AWS
date: 2020-08-19T22:47:50.691Z
description: >-
    What services we used to manage our secrets and good practices we've learned through the process.  Written by Ben Limpanukorn.
featuredpost: true
featuredimage: /img/aws.png
tags:
  - DevOps
---
If you’re building a web application nowadays, chances are you’ll need to interface with external services.  Whether you’re dealing with API keys or database credentials, at some point you’ll need to think about how to manage these secrets in different environments from development to production.

Individuals and small teams working on private codebases might be tempted to simply embed these secrets in their source code reasoning that only trusted team members would have access.  However, this can be a liability if a developer leaves or if snippets of the code are leaked accidentally.  There's also the possibility of unintentionally modifying production resources while testing or developing new features.

The simplest solution to this would be to simply encrypt these secrets in your source code and decrypt them in production with a secret key managed by a service like AWS KMS.  With KMS you can control exactly which IAM users and roles can use the key to decrypt secrets.  You can even use multiple keys to control access to different environments.

If encrypting your secrets is enough to keep them secure, what's the appeal of managed services like AWS Parameter Store or Secrets Manager?  Besides centralizing management and permissions, one of the main benefits---and the reason why we chose this solution---is that these services separate the process of making code changes from the process of updating or rotating parameters.

# Parameter Store, Secrets Manager, and S3

![Managing Secrets with KMS, S3, and Parameter Store](/img/aws-managing-secrets-diagram.svg)

AWS offers several options when it comes to managing parameters and secrets.  There is significant overlap between each service and a multitude of articles comparing their feature sets so instead we'll focus on which services we chose to use and how we used them.

The main features we were looking for in each service was

1. Encryption/Access Control using AWS KMS
2. Hierarchical organization by environment/application
3. Small and Large Parameters
3. Minimal cost

The first two features are supported by all three of the mentioned options.  S3's server-side encryption can be paired with KMS to control access (in addition to a bucket's ACL) and objects can be organized into subfolders.  SSM Parameter Store supports secure parameters encrypted with KMS and parameters can be named hierarchically (e.g. "prod/app/parameter") and retrieved by prefixes.  The Secrets Manager also supports hierarchical naming and encryption with KMS by default.

Since all three services covered our use case, the main deciding factor ended up being the cost of each service.  Excluding the cost of the KMS key which would be common among all three services, the AWS Secrets Manager costs 0.40$ per secret per month while the Parameter Store was free for standard parameters (under 4096 bytes).  Since we didn't need the extra features that the Secrets Manager provides like integrated secrets rotation, it made more sense for us to stick with the Parameter Store.  However, some of our parameters were larger than 4KiB and would have needed to be stored as Advanced Parameters for 0.05$ per month each had we used the Parameter store.  For these we instead opted to store them encrypted with KMS in an S3 bucket which costs 0.023$ per GiB per month.

# Parameter Store Tips

If you’ve decided to use the parameter store to manage your secrets, you have the option of interfacing with it using an SDK such as Boto3 for Python or the AWS CLI.

Unlike the Secrets Manager where all the values are by nature encrypted, the Parameter Store provides the option for plaintext parameters. This means that when you retrieve one or more parameters by calling get-parameter or get-parameters, by default it will return the value as-is, even if it was a SecureString parameter in which case it would return the ciphertext. You need to specify "with decryption" in order for the values to be returned in cleartext.  This can be easy to miss since get-parameter(s) will not throw an error if you attempt to get an encrypted parameter without decrypting it.  Note that the "with decryption" flag will be ignored for String (non encrypted) parameters so you can safely include the flag even if you have a mix of SecureString and String parameters.

Additionally if you use the AWS CLI to retrieve parameters, you can use the "--query" flag to skip parsing through the returned JSON like so:


```
$ aws ssm get-parameter \
    --name "prod/app/secureparam" \
    --with-decryption
# without --query this will return the following JSON:
{
    "Parameter": {
        "Name": "prod/app/secureparam",
        "Type": "SecureString",
        "Value": "MySecretValue",
        "Version": 1,
        "LastModifiedDate": 111222333.444,
        "ARN": "arn:aws:ssm:us-west-2:111222333444:parameter/prod/app/secureparam"
    }
}
```
```
$ aws ssm get-parameter \
    --name "prod/app/secureparam" \
    --with-decryption \
    --query Parameter.Value
# this will return the following:
MySecretValue
```

# Conclusion

Every application has different needs, and your choices may be influenced by other factors such as AWS API call limits or the Secrets Manager's more fine grained access control options.  Whether you decide to use AWS's SSM Parameter Store, Secrets Manager, or S3, ensuring your production credentials and keys are safe and secure is a critical part of any deployment strategy.

