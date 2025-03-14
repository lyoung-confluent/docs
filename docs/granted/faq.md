---
sidebar_position: 5
---

# FAQ

## How does Granted work?

Granted is a command line interface (CLI) written in Go which uses the [AWS Go SDK v2](<(https://github.com/aws/aws-sdk-go-v2)>) to interact with AWS cloud services, including AWS SSO. Granted uses the [AWS Federated sign-in endpoint](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html) in order to open the AWS web console.

## Who was Granted created by?

Granted is created and maintained by [Common Fate](https://commonfate.io/).

## Is Granted free?

Yes! Granted is released as open source with an MIT licence. You are free to remix Granted and use it as part of your own internal CLI tools. You can find the source code to Granted on GitHub [here](https://github.com/common-fate/granted). Issues and Pull Requests are most welcome.

Common Fate is building tooling on top of Granted for identity workflows such as breakglass access and user access review evidence generation. This tooling is available as [Common Fate Cloud](https://granted.dev/cfcloud) under a commercial licence. The Granted core CLI will always remain free and open source.

## What cloud providers does Granted work with?

Granted currently works with AWS, but we're planning on extending support to other cloud providers. If you'd like to see support for another cloud provider please let us know by [opening an issue on GitHub](https://github.com/common-fate/granted/issues)!
