---
layout: "docs"
page_title: "Secret Backend: AWS"
sidebar_current: "docs-secrets-aws"
description: |-
  The AWS secret backend for Vault generates access keys dynamically based on IAM policies.
---

# AWS Secret Backend

Name: `aws`

The AWS secret backend for Vault generates AWS access credentials dynamically
based on IAM policies. This makes IAM much easier to use: credentials could
be generated on the fly, and are automatically revoked when the Vault
lease is expired.

This page will show a quick start for this backend. For detailed documentation
on every path, use `vault path-help` after mounting the backend.

## Quick Start

The first step to using the aws backend is to mount it.
Unlike the `generic` backend, the `aws` backend is not mounted by default.

```text
$ vault mount aws
Successfully mounted 'aws' at 'aws'!
```

Next, we must configure the root credentials that are used to manage IAM credentials:

```text
$ vault write aws/config/root \
    access_key=AKIAJWVN5Z4FOFT7NLNA \
    secret_key=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 \
    region=us-east-1
```

The following parameters are required:

- `access_key` - the AWS access key that has permission to manage IAM
  credentials.
- `secret_key` - the AWS secret key that has permission to manage IAM
  credentials.
- `region` the AWS region for API calls.

The next step is to configure a role. A role is a logical name that maps
to a policy used to generated those credentials. For example, lets create
a "deploy" role:

```text
$ vault write aws/roles/deploy \
    policy=@policy.json
```

This path will create a named role along with the IAM policy used
to restrict permissions for it. This is used to dynamically create
a new pair of IAM credentials when needed.

The `@` tells Vault to load the policy from the file named `policy.json`. Here
is an example IAM policy to get started:

```javascript
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "iam:*",
    "Resource": "*"
  }
}
```

For more information on IAM policies, please see the
[AWS IAM policy documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/PoliciesOverview.html).

To generate a new set of IAM credentials, we simply read from that role:

```text
$ vault read aws/creds/deploy
Key             Value
lease_id        aws/creds/deploy/7cb8df71-782f-3de1-79dd-251778e49f58
lease_duration  3600
access_key      AKIAIOMYUTSLGJOGLHTQ
secret_key      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
security_token  <nil>
```

If you run the command again, you will get a new set of credentials:

```text
$ vault read aws/creds/deploy
Key             Value
lease_id        aws/creds/deploy/82d89562-ff19-382e-6be9-cb45c8f6a42d
lease_duration  3600
access_key      AKIAJZ5YRPHFH3QHRRRQ
secret_key      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
security_token  <nil>
```

If you want keys with an STS token use the 'sts' endpoint instead of 'creds.'
The aws/sts endpoint will always fetch STS credentials with a 1hr ttl.

```text
$vault read aws/sts/deploy
Key            	Value
lease_id       	aws/sts/deploy/31d771a6-fb39-f46b-fdc5-945109106422
lease_duration 	3600
lease_renewable	true
access_key     	ASIAJYYYY2AA5K4WIXXX
secret_key     	HSs0DYYYYYY9W81DXtI0K7X84H+OVZXK5BXXXX
security_token 	AQoDYXdzEEwasAKwQyZUtZaCjVNDiXXXXXXXXgUgBBVUUbSyujLjsw6jYzboOQ89vUVIehUw/9MreAifXFmfdbjTr3g6zc0me9M+dB95DyhetFItX5QThw0lEsVQWSiIeIotGmg7mjT1//e7CJc4LpxbW707loFX1TYD1ilNnblEsIBKGlRNXZ+QJdguY4VkzXxv2urxIH0Sl14xtqsRPboV7eYruSEZlAuP3FLmqFbmA0AFPCT37cLf/vUHinSbvw49C4c9WQLH7CeFPhDub7/rub/QU/lCjjJ43IqIRo9jYgcEvvdRkQSt70zO8moGCc7pFvmL7XGhISegQpEzudErTE/PdhjlGpAKGR3d5qKrHpPYK/k480wk1Ai/t1dTa/8/3jUYTUeIkaJpNBnupQt7qoaXXXXXXXXXX
```


If you get an error message similar to either of the following, the root credentials that you wrote to `aws/config/root` have insufficient privilege:

```text
$ vault read aws/creds/deploy
* Error creating IAM user: User: arn:aws:iam::000000000000:user/hashicorp is not authorized to perform: iam:CreateUser on resource: arn:aws:iam::000000000000:user/vault-root-1432735386-4059

$ vault revoke aws/creds/deploy/774cfb27-c22d-6e78-0077-254879d1af3c
Revoke error: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/sys/revoke/aws/creds/deploy/774cfb27-c22d-6e78-0077-254879d1af3c
Code: 400. Errors:

* invalid request
```

The root credentials need permission to perform various IAM actions. These are the actions that the AWS secret backend uses to manage IAM credentials. Here is an example IAM policy that would grant these permissions:

```javascript
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:CreateUser",
        "iam:PutUserPolicy",
        "iam:ListGroupsForUser",
        "iam:ListUserPolicies",
        "iam:ListAccessKeys",
        "iam:DeleteAccessKey",
        "iam:DeleteUserPolicy",
        "iam:RemoveUserFromGroup",
        "iam:DeleteUser"
      ],
      "Resource": [
        "arn:aws:iam::ACCOUNT-ID-WITHOUT-HYPHENS:user/vault-*"
      ]
    }
  ]
}
```

Note that this policy example is unrelated to the policy you wrote to `aws/roles/deploy`. This policy example should be applied to the IAM user (or role) associated with the root credentials that you wrote to `aws/config/root`. You have to apply it yourself in IAM. The policy you wrote to `aws/roles/deploy` is the policy you want the AWS secret backend to apply to the temporary credentials it returns from `aws/creds/deploy`.

If you get stuck at any time, simply run `vault path-help aws` or with a subpath for
interactive help output.

## A Note on STS Permissions

Vault generates STS tokens using the IAM credentials passed to aws/config.

Those credentials must have two properties:

- They must have permissions to call sts:GetFederatedToken.
- The capabilities of those credentials have to be at least as permissive as those requested
by policies attached to the STS creds.

If either of those conditions are not met, a "403 not-authorized" error will be returned.

See http://docs.aws.amazon.com/STS/latest/APIReference/API_GetFederationToken.html for more details.

## A Note on Consistency

Unfortunately, IAM credentials are eventually consistent with respect to other
Amazon services. If you are planning on using these credential in a pipeline,
you may need to add a delay of 5-10 seconds (or more) after fetching
credentials before they can be used successfully.

If you want to be able to use credentials without the wait, consider using the STS
method of fetching keys. IAM credentials supported by an STS token are available for use
as soon as they are generated.

## API

### /aws/config/root
#### POST

<dl class="api">
  <dt>Description</dt>
  <dd>
    Configures the root IAM credentials used.
    This is a root protected endpoint.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>`/aws/config/root`</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">access_key</span>
        <span class="param-flags">required</span>
        The AWS Access Key
      </li>
      <li>
        <span class="param">secret_key</span>
        <span class="param-flags">required</span>
        The AWS Secret Key
      </li>
      <li>
        <span class="param">region</span>
        <span class="param-flags">required</span>
        The AWS region for API calls
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>
    A `204` response code.
  </dd>
</dl>

### /aws/config/lease
#### POST

<dl class="api">
  <dt>Description</dt>
  <dd>
    Configures the lease settings for generated credentials.
    This is a root protected endpoint.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>`/aws/config/lease`</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">lease</span>
        <span class="param-flags">required</span>
        The lease value provided as a string duration
        with time suffix. Hour is the largest suffix.
      </li>
      <li>
        <span class="param">lease_max</span>
        <span class="param-flags">required</span>
        The maximum lease value provided as a string duration
        with time suffix. Hour is the largest suffix.
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>
    A `204` response code.
  </dd>
</dl>

### /aws/roles/
#### POST

<dl class="api">
  <dt>Description</dt>
  <dd>
    Creates or updates a named role.
  </dd>

  <dt>Method</dt>
  <dd>POST</dd>

  <dt>URL</dt>
  <dd>`/aws/roles/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
    <ul>
      <li>
        <span class="param">policy</span>
        <span class="param-flags">required</span>
        The IAM policy in JSON format.
      </li>
    </ul>
  </dd>

  <dt>Returns</dt>
  <dd>
    A `204` response code.
  </dd>
</dl>

#### GET

<dl class="api">
  <dt>Description</dt>
  <dd>
    Queries a named role.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>`/aws/roles/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
    None
  </dd>

  <dt>Returns</dt>
  <dd>

    ```javascript
    {
      "data": {
        "policy": "..."
      }
    }
    ```

  </dd>
</dl>

#### DELETE

<dl class="api">
  <dt>Description</dt>
  <dd>
    Deletes a named role.
  </dd>

  <dt>Method</dt>
  <dd>DELETE</dd>

  <dt>URL</dt>
  <dd>`/aws/roles/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
    None
  </dd>

  <dt>Returns</dt>
  <dd>
    A `204` response code.
  </dd>
</dl>


### /aws/creds/
#### GET

<dl class="api">
  <dt>Description</dt>
  <dd>
    Generates a dynamic IAM credential based on the named role.
  </dd>

  <dt>Method</dt>
  <dd>GET</dd>

  <dt>URL</dt>
  <dd>`/aws/creds/<name>`</dd>

  <dt>Parameters</dt>
  <dd>
    None
  </dd>

  <dt>Returns</dt>
  <dd>

    ```javascript
    {
      "data": {
        "access_key": "...",
        "secret_key": "...",
        "secret_token": null
      }
    }
    ```

  </dd>
</dl>


### /aws/sts/
#### GET

<dl class="api">
    <dt>Description</dt>
    <dd>
        Generates a dynamic IAM credential with an STS token based on the named role.
    </dd>

    <dt>Method</dt>
    <dd>GET</dd>

    <dt>URL</dt>
    <dd>`/aws/sts/<name>`</dd>

    <dt>Parameters</dt>
    <dd>
        None
    </dd>

    <dt>Returns</dt>
    <dd>

    ```javascript
    {
        "data": {
            "access_key": "...",
            "secret_key": "...",
            "secret_token": "..."
        }
    }
    ```
    </dd>
</dl>
