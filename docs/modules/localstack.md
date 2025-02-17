# Localstack Module

Testcontainers module for the Atlassian's [LocalStack](https://github.com/localstack/localstack), 'a fully functional local AWS cloud stack'.

## Usage example

Running LocalStack as a stand-in for AWS S3 during a test:

```java
DockerImageName localstackImage = DockerImageName.parse("localstack/localstack:0.11.3");

@Rule
public LocalStackContainer localstack = new LocalStackContainer(localstackImage)
        .withServices(S3);

@Test
public void someTestMethod() {
    // AWS SDK v1
    AmazonS3 s3 = AmazonS3ClientBuilder
                    .standard()
                    .withEndpointConfiguration(localstack.getEndpointConfiguration(S3))
                    .withCredentials(localstack.getDefaultCredentialsProvider())
                    .build();

            s3.createBucket("foo");
            s3.putObject("foo", "bar", "baz");

    // AWS SDK v2
    S3Client s3 = S3Client
                .builder()
                .endpointOverride(localstack.getEndpointOverride(LocalStackContainer.Service.S3))
                .credentialsProvider(StaticCredentialsProvider.create(AwsBasicCredentials.create(
                    localstack.getAccessKey(), localstack.getSecretKey()
                )))
                .region(Region.of(localstack.getRegion()))
                .build();

            s3.createBucket(b -> b.bucket("foo"));
            s3.putObject(b -> b.bucket("foo").key("bar"), RequestBody.fromBytes("baz".getBytes()));
```

Environment variables listed in [Localstack's README](https://github.com/localstack/localstack#configurations) may be used to customize Localstack's configuration.
Use the `.withEnv(key, value)` method on `LocalStackContainer` to apply configuration settings.

## `HOSTNAME_EXTERNAL` and hostname-sensitive services

Some Localstack APIs, such as SQS, require the container to be aware of the hostname that it is accessible on - for example, for construction of queue URLs in responses.

Testcontainers will inform Localstack of the best hostname automatically, using the `HOSTNAME_EXTERNAL` environment variable:

* when running the Localstack container directly without a custom network defined, it is expected that all calls to the container will be from the test host. As such, the container address will be used (typically localhost or the address where the Docker daemon is running).

    <!--codeinclude-->
    [Localstack container running without a custom network](../../modules/localstack/src/test/java/org/testcontainers/containers/localstack/LocalstackContainerTest.java) inside_block:without_network
    <!--/codeinclude-->

* when running the Localstack container [with a custom network defined](/features/networking/#advanced-networking), it is expected that all calls to the container will be **from other containers on that network**. `HOSTNAME_EXTERNAL` will be set to the *last* network alias that has been configured for the Localstack container.

    <!--codeinclude-->
    [Localstack container running without a custom network](../../modules/localstack/src/test/java/org/testcontainers/containers/localstack/LocalstackContainerTest.java) inside_block:with_network
    <!--/codeinclude-->

* Other usage scenarios, such as where the Localstack container is used from both the test host and containers on a custom network are not automatically supported. If you have this use case, you should set `HOSTNAME_EXTERNAL` manually.

## AWS Lambda and Networking

When a test using Localstack triggers a [AWS Lambda function](https://aws.amazon.com/lambda/), Localstack [starts a new docker container](https://docs.localstack.cloud/localstack/lambda-executors/#docker) to run the Lambda code. This might cause problems if your Lambda function calls other AWS services in your Localstack setup. To fix this we have to make sure these Lambda containers are in the same network as the rest of our containers.

```java
Network shared = Network.newNetwork()
String networkName = ((NetworkImpl)shared).getName();

DockerImageName localstackImage = DockerImageName.parse("localstack/localstack:0.11.3");

@Rule
public LocalStackContainer localstack = new LocalStackContainer(localstackImage)
        .withServices(LAMBDA)
        .withEnv("LAMBDA_DOCKER_NETWORK", networkName)
        .withNetwork(network)
        .withNetworkAliases("localstack")
```
Your Lambda should now be able to connect to other Localstack AWS services with the `http://localstack:4566` url.
The `LAMBDA_DOCKER_NETWORK` environment variable is [described in the Localstack docs](https://github.com/localstack/localstack#lambda-configurations).

## Adding this module to your project dependencies

Add the following dependency to your `pom.xml`/`build.gradle` file:

```groovy tab='Gradle'
testImplementation "org.testcontainers:localstack:{{latest_version}}"
```

```xml tab='Maven'
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>localstack</artifactId>
    <version>{{latest_version}}</version>
    <scope>test</scope>
</dependency>
```
