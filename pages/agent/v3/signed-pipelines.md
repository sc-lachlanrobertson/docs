# Signed pipelines

## What are signed pipelines?

Signed Pipelines is a security feature on the Buildkite Agent that allows you to cryptographically sign steps that are uploaded Buildkite, and then verify that signature on the Buildkite Agent before running the step. This can prevent a malicious actor from modifying the steps that are run on your agents, especially in the (hopefully unlikely) event that the Buildkite backend is compromised.

These signatures mean that while a threat actor could modify jobs in flight, when the Agent runs those steps, when it tries to verify the signature, it will fail, and the step will not be run.

<details>
  <summary>I think I've seen this before...</summary>
  This work is inspired by the older <a href="https://github.com/buildkite/buildkite-signed-pipeline"><code>buildkite-signed-pipeline</code></a> tool, which was a tool you could add to your agent instances. It had a similar idea - signing steps before they're uploaded to Buildkite, then verifying them when they're run. However, it had a number of limitations, including:
  <ul>
    <li>It had to be installed on every agent instance, leading to more configuration</li>
    <li>It only supported symmetric HS256 signatures, meaning that every verifier could also sign uploads</li>
    <li>It couldn't sign matrix steps</li>
  </ul>

  This newer version of pipeline signing is built right into the agent, and addresses all of these limitations. Being built into the agent, it's easier to configure and use.

  Many thanks to Seek.com.au, who we collaborated with on the older version of the tool, and who have been instrumental in the development of this newer version.
</details>

## Enabling signed pipelines on your pipelines

Behind the scenes, Signed Pipelines uses [JSON Web Signing (JWS)](https://datatracker.ietf.org/doc/html/rfc7797) to generate its signatures. This means that you'll need to generate a [JSON Web Key Set (JWKS)](https://datatracker.ietf.org/doc/html/rfc7517) to sign and verify your pipelines with, then configure your agents to use those keys.

### How do I generate a key pair?

Luckily, the Buildkite Agent has you covered! There's a JWKS generation tool built into the agent, which you can use to generate a key pair for you. To run it, you'll need to [install the agent on your machine](/docs/agent/v3/installation), and then run:
```bash
buildkite-agent tool keygen --alg <algorithm> --key-id <key-id>
```

replacing `<algorithm>` and `<key-id>` with the algorithm you want to use, and the key ID you want to use. For example, to generate an RSA-PSS key pair with a key ID of `my-key-id`, you'd run:
```bash
buildkite-agent tool keygen --alg PS256 --key-id my-key-id
```

The agent will then generate two JSON Web Key Sets for you, one public and one private, and output them to files in the current directory. You can then use these keys to sign and verify your pipelines.

If no key id is provided, the agent will generate a random one for you.

Note that the value of `--alg` must be a valid [JSON Web Signing Algorithm](https://datatracker.ietf.org/doc/html/rfc7518#section-3). The agent does not support the `none` algorithm, or any signature algorithms in the `RS` series (any signing algorithms that use RSASSA-PKCS1 v1.5 signatures, `RS256` for example).

<details>
  <summary>Why doesn't the agent support RSASSA-PKCS1 v1.5 signatures?</summary>
  In short, it's because RSASSA-PKCS1 v1.5 signatures are generally regarded to be less secure than the newer RSA-PSS signatures. While RSASSA-PKCS1 v1.5 signatures are still largely known to be relatively secure, we want to encourage our users to use the most secure algorithms possible, so when using RSA keys, we only support RSA-PSS signatures. We also recommend looking into ECDSA and EdDSA signatures, which are generally regarded to be more secure than RSA signatures.
</details>

### What algorithm should I use?

Signed Pipelines supports a number of different signing algorithms, which can largely be broken down into two categories: Symmetric Signing Algorithms (where signing and verification use the same keys), and Asymmetric Signing Algorithms (where signing and verification use different keys).

We recommend using an asymmetric signing algorithm, as it creates a permissions boundary between the agents that upload and sign pipelines, and agents that run other jobs and have the ability to verify signatures. This means that were an attacker able to compromise instances with verification keys, they wouldn't be able to modify the steps that are run on your agents.

With regards to specific algorithm choice, basically any of the asymmetric signing algorithms we support are fine and will be secure. If you're not sure which one to use, `EdDSA` is proven to be secure, has a modern design, wasn't designed by a Nation State Actor, and produces nice short signatures.

Signed pipelines supports the following signing algorithms:

#### Symmetric
- `HS256`
- `HS384`
- `HS512`

#### Asymmetric
- `PS256`
- `PS384`
- `PS512`
- `ES256`
- `ES384`
- `ES512`
- `EdDSA`

### Making your agents sign steps they upload 

Now that your keys are generated, we need to configure your agents to use them. On agents that will be uploading pipelines, add the following to your agent's config file:
```ini
job-signing-jwks-path=<path to signing keys>
signing-key-id=<the key id you generated earlier>
job-verification-jwks-path=<path to verification keys>
```

On instances that will verify jobs, add:
```ini
job-verification-jwks-path=<path to verification keys>
```

No

### Signing the pipeline's stored configuration

[ed: this section feels clunky to me, and i'm not sure the best way to explain it - we need to sign the initial steps of a pipeline as well as the steps that get uploaded by an agent, but it's difficult to concisely convey all the concepts involved]

In Buildkite, pipelines have two YAML definitions - one that's stored Buildkite, which you can change on the settings page for a given pipeline, and other(s) that are uploaded by agents. In this guide, we'll refer to the former as the "stored" configuration, and the latter as "uploaded" configuration. The stored configuration generally contains the initial steps needed to run a pipeline, and almost always contains a step that uploads a pipeline to Buildkite.

> Some older Buildkite pipelines don't use YAML for their stored pipeline configuration - these aren't supported by signed pipelines, and will have to be migrated to use the new format.

You may have noticed that when we configured the signing instances, we gave them the capability to both sign and verify jobs. For maximum security, the stored pipeline will need to be

The solution here is to add static signatures to the steps in your pipeline's stored configuration.

[ed: explain how to do this - buildkite-agent pipeline upload --jwks-file-path /path/to/keys --signing-key-id blah --dry-run and copy and paste into buildkite's UI]

### Hard mode: static signatures

[ed: if you're really paranoid, you can sign a pipeline once, and hardcode the signatures into your pipeline yaml. this means that if anyone changes anything on the signed steps, your jobs will fail]

## Rotating signing keys

[ed: generate new keys, and add them to the existing key sets you've already generated. switch over the --signing-key-id and bob's your uncle]

## General guidance for running signed pipelines

[ed: something about using different queues for pipeline uploads vs everything else, then give uploaders the singing key and runners the verification key]
