---
title: "Stop using static credentials"
date: 2022-11-07T22:34:34+01:00
draft: false
categories:
    - Devops
tags:
    - Security
    - CI
    - Gitlab
    - Github
    - AWS
---

We are reading on the news almost every other month now, of companies getting pwned left and right. Phishing attacks are increasingly more common, usually targeting employees with access to repositories containing static credentials and secrets, which the attackers can then leverage to gain access to the cloud accounts, and from there it is game over, since pretty much all of the data that companies store and process, is stored in the cloud.

Storing secrets and access keys in plain text in repositories, poses a huge problem, and is often the weakest link when it comes to gaining access to internal systems. Even though the keys are seemingly stored in a secure environment (Github), and the engineers all use strong passwords, along with 2FA, they still are susceptible to the increasingly sophisticated phishing attacks that the attackers use.

Usually when engineers create those keys, they scope them with very lax permissions (because honestly who has time to go back and forth with guessing which exact permissions are needed in order for the CI workflow to pass), so they slap in 'Administrator' access and call it a day. This is enough for the attacker to gain access to the whole kingdom and wreak havoc.

But there are better ways of handling authorization for cloud resources, without the need to use static credentials.

## credentials in a CI environment

In a typical CI workflow, the last steps usually involve pushing some files to a blob store, and running some commands to deploy an application in the cloud. These actions naturally require authenticating to the cloud provider, before accessing those resources and running those commands.

Using static access keys for this scenario has several downsides:

- We would need to rotate the keys on a regular basis.
- Some CI services don't offer an easy way to store and use secrets in a secure manner. You would need an [additional solution](https://docs.gitlab.com/ee/ci/secrets/) like Vault to encrypt and use env variables in the CI workflows.
- And the most obvious, these credentials are long-lived and could easily get leaked. Sometimes devs would push these credentials accidentally to public repositories, which is a huge security liability. Github now even [scans](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) repositories for credentials that have been leaked publicly, to protect users from malicious actors.

Fortunately there's a better way of authenticating to cloud providers, that doesn't involve the usage of static credentials. Instead you could make your CI workflow request temporary credentials dynamically, when needed, using web identity federation (OpenID Connect).

It works like this:

- you configure Github (or your code platform of choice) as an identity provider in AWS (or other major cloud provider, they all support it)
- configure a role to be used in your build workflow, that has the appropriate permissions
- assume the role in the build workflow. This involves receiving an authentication token and then exchanging that token to request temporary security credentials.

I know, i know, it's a bit of a hassle to request tokens and exchange them for temporary creds, but fortunately there are ready-made solutions that simplify this whole workflow.

Let me show you an example with Github and AWS.

### Example

Let's say, for the sake of this example, that we have some React application in a Github repository, that we want to deploy on S3 and Cloudfront. Naturally, we want to do the building and deployment process in an automated way, so we opt in to using Github Actions.

The workflow would look something like this:

```yaml
name: Build and Deploy

on:
  push:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    # These permissions are needed to interact with GitHub's OIDC Token endpoint.
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3

      - run: npm install
      - run: npm build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::1234561234:role/my-gh-actions-role
          aws-region: us-east-1

      - name: Deploy
        run: |
          aws s3 cp ./build s3://my-s3-bucket --recursive --include "*"
          aws cloudfront create-invalidation --distribution-id <DISTRIBUTION-ID> --paths "/*"
```

As you can see, I'm using the handy [Github Action](https://github.com/aws-actions/configure-aws-credentials) for configuring the AWS credentials. There's only two parameters that we need to set up in the action, before we can use the `aws` command in the consequent steps:
- `role-to-assume` - the role ARN of the role that we want to assume. This role should have all the necessary permissions needed to interact with the AWS services.
- `aws-region` - the AWS region.

Now, before we can add the role ARN, we obviously need to create it in IAM first. But even before that, we'll need to add Github as an **Identity Provider** in IAM:

1. We open up the AWS console, and go to the [IAM](https://console.aws.amazon.com/iam/) section
2. Open Identity Providers
3. Click on the Add Provider button
4. Choose OpenID Connect as the provider type
5. Use `https://token.actions.githubusercontent.com` as the provider URL
6. For "Audience" use **sts.amazonaws.com**

Next up, we'll need to create the role and prepare it to be used with Web Identity federation:
1. Open the [IAM](https://console.aws.amazon.com/iam/) console
2. We go to Roles and choose Create Role
3. Choose the **Web Identity** role type
4. For Identity Provider, we choose **token.actions.githubusercontent.com** and for Audience **sts.amazonaws.com**
5. We select the necessary permissions (don't use Administrator permissions!). In this case we need permissions to write to our S3 bucket and create invalidation for our Cloudfront distribution

We can also edit the trust policy, to add the **sub** field to the validation condition. This will restrict the assume role action only to be used for a particular repo. For example:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::123456123456:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:*"
                },
                "ForAllValues:StringEquals": {
                    "token.actions.githubusercontent.com:iss": "https://token.actions.githubusercontent.com",
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

After we've created the role, we can add the role ARN to the configure AWS creds action, and we're good to go.

Note that for the workflow to work, it requires a **permission** setting with **id-token: write**. This setting allows the token to be received from Github's identity provider. You can add the permissions globally in the workflow, or per job, like so:

```yaml
# .github/workflows/build.yaml

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
# the rest of the workflow...
```

Now we can safely authenticate to AWS and use the `aws` command with no static keys in sight.

You could essentially achieve the same setup in all major code platforms, since they all support OIDC.

Here are some links with guides:

Github:
- https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform
- https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure

Gitlab:
- https://docs.gitlab.com/ee/ci/cloud_services/aws/
- https://docs.gitlab.com/ee/ci/cloud_services/azure/
- https://docs.gitlab.com/ee/ci/cloud_services/google_cloud/
- https://docs.gitlab.com/ee/ci/examples/authenticating-with-hashicorp-vault/

Bitbucket:
- https://support.atlassian.com/bitbucket-cloud/docs/deploy-on-aws-using-bitbucket-pipelines-openid-connect/
- https://bitbucket.org/blog/bitbucket-pipelines-and-openid-connect-no-more-secret-management

Stay vigilant and happy coding!
