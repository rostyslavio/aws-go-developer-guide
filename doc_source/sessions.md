# Using Sessions to Configure Service Clients in the AWS SDK for Go<a name="sessions"></a>

In the AWS SDK for Go, a session is an object that contains configuration information for [service clients](making-requests.md)\. which you use to interact with AWS services\. For example, sessions can include information about the region where requests will be sent, which credentials to use, or additional request handlers\. Whenever you create a service client, you must specify a session\. For more information about sessions, see the [session](https://docs.aws.amazon.com/sdk-for-go/api/aws/session/) package in the AWS SDK for Go API Reference\.

Sessions can be shared across all service clients that share the same base configuration\. The session is built from the SDK’s default configuration and request handlers\.

You should cache sessions when possible\. This is because creating a new session loads all configuration values from the environment and configuration files each time the session is created\. Sharing the session value across all of your service clients ensures the configuration is loaded the fewest number of times\.

## Concurrency<a name="concurrency"></a>

Sessions are safe to use concurrently as long as the session isn’t being modified\. The SDK doesn’t modify the session once the session is created\. Creating service clients concurrently from a shared session is safe\.

## Sessions with a Shared Configuration File<a name="sessions-with-shared-config"></a>

Using the previous method, you can create sessions that load the additional configuration file only if the `AWS_SDK_LOAD_CONFIG` environment variable is set\. Alternatively you can explicitly create a session with a shared configuration enabled\. To do this, you can use `NewSessionWithOptions` to configure how the session is created\. Using the `NewSessionWithOptions` with `SharedConfigState` set to `SharedConfigEnable` will create the session as if the `AWS_SDK_LOAD_CONFIG` environment variable was set\.

## Creating Sessions<a name="creating-sessions"></a>

When you create a `session`, you can pass in optional `aws.Config` values that override the default or that override the current configuration values\. This allows you to provide additional or case\-based configuration as needed\.

By default `NewSession` only loads credentials from the shared credentials file \(`~/.aws/credentials`\)\. If the `AWS_SDK_LOAD_CONFIG` environment variable is set to a truthy value, the session is created from the configuration values from the shared configuration \(`~/.aws/config`\) and shared credentials \(`~/.aws/credentials`\) files\. See [Sessions with a Shared Configuration File](#sessions-with-shared-config) for more information\.

Create a session with the default configuration and request handlers\. The following example creates a session with credentials, region, and profile values from either the environment variables or the shared credentials file\. It requires that the `AWS_PROFILE` is set, or `default` is used\.

```
sess, err := session.NewSession()
```

The SDK provides a [default configuration](https://docs.aws.amazon.com/sdk-for-go/api/aws/defaults/) that all sessions use, unless you override a field\. For example, you can specify an AWS Region when you create a session by using the `aws.Config` struct\. For more information about the fields you can specify, see the [aws\.Config](https://docs.aws.amazon.com/sdk-for-go/api/aws/#Config) in the AWS SDK for Go API Reference\.

```
sess, err := session.NewSession(&aws.Config{
    Region: aws.String("us-east-2")},
)
```

Create an Amazon S3 client instance from a session:

```
sess, err := session.NewSession()
if err != nil {
    // Handle Session creation error
}
svc := s3.New(sess)
```

## Create Sessions with Option Overrides<a name="create-session-with-option-overrides"></a>

In addition to `NewSession`, you can create sessions using `NewSessionWithOptions`\. This function allows you to control and override how the session will be created through code, instead of being driven by environment variables only\.

Use [NewSessionWithOptions](https://docs.aws.amazon.com/sdk-for-go/api/aws/session/#NewSessionWithOptions) when you want to provide the config profile, or override the shared credentials state \(`AWS_SDK_LOAD_CONFIG`\)\.

```
// Equivalent to session.New
sess, err := session.NewSessionWithOptions(session.Options{})

// Specify profile to load for the session's config
sess, err := session.NewSessionWithOptions(session.Options{
    Profile: "profile_name",
})

// Specify profile for config and region for requests
sess, err := session.NewSessionWithOptions(session.Options{
    Config: aws.Config{Region: aws.String("us-east-2")},
    Profile: "profile_name",
})

// Force enable Shared Config support
sess, err := session.NewSessionWithOptions(session.Options{
    SharedConfigState: SharedConfigEnable,
})

// Assume an IAM role with MFA prompting for token code on stdin
sess := session.Must(session.NewSessionWithOptions(session.Options{
    AssumeRoleTokenProvider: stscreds.StdinTokenProvider,
    SharedConfigState: SharedConfigEnable,
}))
```

### Deprecated `New`<a name="deprecated-new"></a>

The `New` function has been deprecated because it doesn’t provide a good way to return errors that occur when loading the configuration files and values\. Because of this, `NewSession` was created so errors can be retrieved when creating a session fails\.

### Shared Configuration Fields<a name="shared-configuration-fields"></a>

By default, the SDK loads credentials from the shared credentials file `~/.aws/credentials`\. Any other configuration values are provided by the environment variables, SDK defaults, and user\-provided `aws.config` values\.

If the `AWS_SDK_LOAD_CONFIG` environment variable is set, or the **SharedConfigEnable** option is used to create the session \(as shown in the following example\), additional configuration information is also loaded from the shared configuration file \(`~/.aws/config`\), if it exists\. If any configuration setting value differs between the two files, the value from the shared credentials file \(`~/.aws/credentials`\) takes precedence\.

```
sess := session.Must(session.NewSessionWithOptions(session.Options{
    SharedConfigState: session.SharedConfigEnable,
}))
```

See the [session package’s documentation](https://docs.aws.amazon.com/sdk-for-go/api/aws/session/) for more information on shared credentials setup\.

## Environment Variables<a name="environment-variables"></a>

When a session is created, you can set several environment variables to adjust how the SDK functions, and what configuration data it loads when creating sessions\. Environment values are optional\. For credentials, you must set both an access key and a secret access key\. Otherwise, Go ignores the one you’ve set\. All environment variable values are strings unless otherwise noted\.

See the [session package’s documentation](https://docs.aws.amazon.com/sdk-for-go/api/aws/session/) for more information on environment variable setup\.

## Adding Request Handlers<a name="adding-request-handlers"></a>

You can add handlers to a session for processing HTTP requests\. All service clients that use the session inherit the handlers\. For example, the following handler logs every request and its payload made by a service client\.

```
// Create a session, and add additional handlers for all service
// clients created with the Session to inherit. Adds logging handler.
sess, err := session.NewSession()
sess.Handlers.Send.PushFront(func(r *request.Request) {
    // Log every request made and its payload
    logger.Println("Request: %s/%s, Payload: %s",
        r.ClientInfo.ServiceName, r.Operation, r.Params)
})
```

## Copying a Session<a name="copying-a-session"></a>

You can use the [Copy](https://docs.aws.amazon.com/sdk-for-go/api/aws/session/#Session.Copy) method to create copies of sessions\. Copying sessions is useful when you want to create multiple sessions that have similar settings\. Each time you copy a session, you can specify different values for any field\. For example, the following snippet copies the `sess` session while overriding the `Region` field to `us-east-2`:

```
usEast2Sess := sess.Copy(&aws.Config{Region: aws.String("us-east-2")})
```