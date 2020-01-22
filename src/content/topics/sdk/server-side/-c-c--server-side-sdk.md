---
title: "C/C++ (server-side) SDK reference"
excerpt: ""
---
This reference guide documents basic usage of our C server-side SDK, and explains in detail how its functions work. If you want to dig even deeper, our SDKs are open source-- head to our [C SDK GitHub repository](https://github.com/launchdarkly/c-server-sdk) to look under the hood. Additionally you can clone and run a [sample application](https://github.com/launchdarkly/hello-c-server) using this SDK.
<Callout intent="alert">
  <Callout.Title>For use in server applications</Callout.Title>
   <Callout.Description>LaunchDarkly provides both a client-side and a server-side C/C++ SDK. If you are embedding LaunchDarkly in a client-side application please use the [client-side SDK](https://docs.launchdarkly.com/docs/c-sdk-reference). Learn more about our [client-side and server-side SDKs](./client-side-and-server-side).</Callout.Description>
</Callout>

## Getting started
Building on top of our [Quickstart](./getting-started) guide, the following steps will get you started with using the LaunchDarkly SDK in your C application.

Unlike other LaunchDarkly SDKs, the C SDK has no installation steps. To get started, clone [this repository](https://github.com/launchdarkly/c-server-sdk) or download a release archive from the [GitHub Releases](https://github.com/launchdarkly/c-server-sdk/releases) page. You can use `CMakeLists.txt` in this repository as a starting point for integrating this SDK into your application.

Once ready, your first step should be to include the LaunchDarkly SDK headers:
[block:code]
{
  "codes": [
    {
      "code": "#include \"launchdarkly/api.h\"",
      "language": "c"
    }
  ]
}
[/block]
Configure logging, and call the global initialization function. These functions must be called before any other operations are performed. LaunchDarkly provides a predefined convenience logger.
[block:code]
{
  "codes": [
    {
      "code": "LDConfigureGlobalLogger(LD_LOG_INFO, LDBasicLogger);\nLDGlobalInit();",
      "language": "c"
    }
  ]
}
[/block]
Once the SDK is installed and imported, you'll want to create a single, shared instance of `LDClient`. You should specify your SDK key here so that your application will be authorized to connect to LaunchDarkly and for your application and environment.

Calling `LDClientInit` will initiate a remote call to the LaunchDarkly service to fetch feature flags. This call will block up to the time defined by `maxwaitmilliseconds`. If you request a feature flag before the client has completed initialization, you will receive the default flag value.
[block:code]
{
  "codes": [
    {
      "code": "unsigned int maxwaitmilliseconds = 
1. * 1000;\nstruct LDConfig *config = LDConfigNew(\"YOUR_SDK_KEY\");\nstruct LDUser *user = LDUserNew(\"YOUR_USER_KEY\");\nstruct LDClient *client = LDClientInit(config, maxwaitmilliseconds);",
      "language": "c"
    }
  ]
}
[/block]

<Callout intent="alert">
   <Callout.Description>It's important to make this a singleton-- internally, the client instance maintains internal state that allows us to serve feature flags without making any remote requests. **Be sure that you're not instantiating a new client with every request.**",
  <Callout.Title>LDClient must be a singleton
  </Callout.Description>
</Callout>

Using `client`, you can check which variation a particular user should receive for a given feature flag.
[block:code]
{
  "codes": [
    {
      "code": "show_feature = LDBoolVariation(client, user, \"your.flag.key\", false);\nif (show_feature) {\n    // application code to show the feature\n} else {\n    // the code to run if the feature is off\n}",
      "language": "c"
    }
  ]
}
[/block]
If it is possible for your flag evaluation to be executed before `client` initializes, you should wrap your call in `LDClientIsInitialized(client)`:
[block:code]
{
  "codes": [
    {
      "code": "if (LDClientIsInitialized(client)) {\n  // flag evaluation goes here\n}",
      "language": "c"
    }
  ]
}
[/block]
Lastly, when your application is about to terminate, shut down `client`. This ensures that the client releases any resources it is using, and that any pending analytics events are delivered to LaunchDarkly. If your application quits without this shutdown step, you may not see your requests and users on the dashboard, because they are derived from analytics events. **This is something you only need to do once**.
[block:code]
{
  "codes": [
    {
      "code": "LDClientClose(client);",
      "language": "c"
    }
  ]
}
[/block]

## Customizing your client
You can also pass other custom parameters to the client via the configuration object:

[block:code]
{
  "codes": [
    {
      "code": "struct LDConfig *config = LDConfigNew(\"YOUR_SDK_KEY\");\nLDConfigSetEventsCapacity(config, 1000);\nLDConfigSetEventsFlushInterval(config, 30000);",
      "language": "c"
    }
  ]
}
[/block]
Here, we've customized the event queue capacity and flush interval parameters. Note that you should finish setting up your configuration object before you call `LDClientInit`.
## Users
Feature flag targeting and rollouts are all determined by the user you pass to your `variation` calls.
[block:code]
{
  "codes": [
    {
      "code": "struct LDUser *user = LDUserNew(\"aa0ceb\");\nLDUserSetFirstName(user, \"Ernestina\");\nLDUserSetLastName(user, \"Evans\");\nLDUserSetEmail(user, \"ernestina@example.com\");
struct LDJSON *tmp;\nstruct LDJSON *custom = LDNewObject();\nstruct LDJSON *groups = LDNewArray();\ntmp = LDNewText(\"Google\");\nLDArrayPush(groups, tmp);\ntmp = LDNewText(\"Microsoft\");\nLDArrayPush(groups, tmp);\nLDObjectSetKey(custom, \"groups\", groups);
LDUserSetCustom(user, custom);",
      "language": "c"
    }
  ]
}
[/block]
Let's walk through this snippet. The most important attribute is the user key-- in this case we've used the hash "aa0ceb". *The user key is the only mandatory user attribute.* The key should also uniquely identify each user. You can use a primary key, an e-mail address, or a hash, as long as the same user always has the same key. We recommend using a hash if possible.

All of the other attributes (like `firstName`, `email`, and the `custom` attributes) are optional. The attributes you specify will automatically appear on our dashboard, meaning that you can start segmenting and targeting users with these attributes.

In addition to built-in attributes like names and e-mail addresses, you can pass us any of your own user data by passing `custom` attributes, like the `groups` attribute in the example above.

Custom attributes are one of the most powerful features of LaunchDarkly. They let you target users according to any data that you want to send to us-- organizations, groups, account plans-- anything you pass to us becomes available instantly on our dashboard.
[block:code]
{
  "codes": [
    {
      "code": "LDUserFree(user);",
      "language": "c"
    }
  ]
}
[/block]
When you are done with an `LDUser` ensure that you free the structure.
## Private user attributes
You can optionally configure the C SDK to treat some or all user attributes as [private user attributes](https://docs.launchdarkly.com/v2.0/docs/private-user-attributes). Private user attributes can be used for targeting purposes, but are removed from the user data sent back to LaunchDarkly.

In the C SDK there are three ways to define private attributes for the LaunchDarkly client:

* When creating the `LDConfig` object, you can use `LDConfigSetAllAttributesPrivate`. When you do this, all user attributes (except the key) for the user are removed before the user is sent to LaunchDarkly.

[block:code]
{
  "codes": [
    {
      "code": "LDConfigSetAllAttributesPrivate(config, true);",
      "language": "c"
    }
  ]
}
[/block]
* When creating the `LDConfig` object, you can list specific private attributes with `LDConfigAddPrivateAttribute`. If any user has a custom or built-in attribute named in this list, it will be removed before the user is sent to LaunchDarkly.
[block:code]
{
  "codes": [
    {
      "code": "LDConfigAddPrivateAttribute(config, \"email\");",
      "language": "c"
    }
  ]
}
[/block]
* You can also define private attribute names on a per-user basis. For example:
[block:code]
{
  "codes": [
    {
      "code": "LDUserAddPrivateAttribute(user, \"email\");",
      "language": "c"
    }
  ]
}
[/block]

## Variations
The `variation` family of functions determine whether a flag is enabled or not for a specific user. In C, there is a `variation` function for each type (e.g. `LDBoolVariation`, `LDStringVariation`, etc).
[block:code]
{
  "codes": [
    {
      "code": "bool value = LDBoolVariation(client, user, \"your.feature.key\", false, NULL);",
      "language": "c"
    }
  ]
}
[/block]
The functions take an `LDClient`, `LDUser`, feature flag key, default value, and optional `LDDetails` struct for an evaluation explanation (see below).

The default value will only be returned if an error is encountered-- for example, if the feature flag key doesn't exist or the user doesn't have a key specified. 
## Evaluation details
By passing an `LDDetails` struct to a variation call you can programmatically inspect the reason for a particular evaluation. For more information about the nature of the "reason" data, see the reference guide on [Evaluation reasons](./evaluation-reasons).
[block:code]
{
  "codes": [
    {
      "code": "struct LDDetails details;\nbool value = LDBoolVariation(client, user, \"your.feature.key\", false, &details);
/* inspect details here */\nif (details.reason == LD_FLAG_NOT_FOUND) {\n\t/* ... */\n}
LDDetailsClear(&details);",
      "language": "c"
    }
  ]
}
[/block]
For particular information on the `LDDetails` structure please inspect `ldvariations.h`.
## All flags

<Callout intent="alert">
  <Callout.Title>Creating users</Callout.Title>
   <Callout.Description>Note that unlike variation and identify calls, `LDAllFlags` does not send events to LaunchDarkly. Thus, users are not created or updated in the LaunchDarkly dashboard.
</Callout.Description>
</Callout>
The `LDAllFlags` function captures the state of all feature flag keys with regard to a specific user. This includes their values, as well as other metadata.

This method can be useful for passing feature flags to your front-end. In particular, it can be used to provide bootstrap flag settings for our [JavaScript SDK](./js-sdk-reference).
[block:code]
{
  "codes": [
    {
      "code": "struct LDJSON *allFlags = LDAllFlags(client, user);",
      "language": "c"
    }
  ]
}
[/block]

## Anonymous users
You can also distinguish logged-in users from anonymous users in the SDK, as follows:
[block:code]
{
  "codes": [
    {
      "code": "LDUserSetAnonymous(user, true);",
      "language": "c"
    }
  ]
}
[/block]
You will still need to generate a unique key for anonymous users-- session IDs or UUIDs work best for this.

Anonymous users work just like regular users, except that they won't appear on your Users page in LaunchDarkly. You also can't search for anonymous users on your Features page, and you can't search or autocomplete by anonymous user keys. This is actually a good thing-- it keeps anonymous users from polluting your Users page!
## Offline mode
In some situations, you might want to stop making remote calls to LaunchDarkly and fall back to default values for your feature flags. For example, if your software is both cloud-hosted and distributed to customers to run on premise, it might make sense to fall back to defaults when running on premise.
[block:code]
{
  "codes": [
    {
      "code": "LDConfigSetOffline(config, true);",
      "language": "c"
    }
  ]
}
[/block]

## Flush
Internally, the LaunchDarkly SDK keeps an event buffer for analytics events. These are flushed periodically in a background thread. In some situations (for example, if you're testing out the SDK in a simulator), you may want to manually call flush to process events immediately.
[block:code]
{
  "codes": [
    {
      "code": "LDClientFlush(client);",
      "language": "c"
    }
  ]
}
[/block]
This function will not block, but instead initiate a flush operation in the background. Note that the flush interval is configurable-- if you need to change the interval, you can do so via the configuration.
## Track
The `LDClientTrack` function allows you to record actions your users take on your site. This lets you record events that take place on your server. In LaunchDarkly, you can tie these events to goals in A/B tests. Here's a simple example:
[block:code]
{
  "codes": [
    {
      "code": "LDClientTrack(client, \"your-goal-key\", user, NULL);",
      "language": "c"
    }
  ]
}
[/block]
You can also attach a JSON object containing arbitrary data to your event:
[block:code]
{
  "codes": [
    {
      "code": "struct LDJSON *data = LDNewObject();\nstruct LDJSON *price = LDNewNumber(320);\nLDObjectSetKey(data, \"price\", price);
LDClientTrack(client, \"your-goal-key\", user, data);
LDJsonFree(data);",
      "language": "c"
    }
  ]
}
[/block]

## Identify
The `LDClientIdentify` function creates or updates users on LaunchDarkly, making them available for targeting and autocomplete on the dashboard. In most cases, you won't need to call `LDClientIdentify`-- the `variation` call will automatically create users on the dashboard for you. `LDClientIdentify` can be useful if you want to pre-populate your dashboard before launching any features. 
[block:code]
{
  "codes": [
    {
      "code": "LDClientIdentify(client, user);",
      "language": "c"
    }
  ]
}
[/block]

## Logging
By default, there is no log output. We provide a default logger that you can enable, with the option to set how verbose it should be. For instance, setting it to `LD_LOG_TRACE` would produce the most verbose output, which may be useful in troubleshooting.
[block:code]
{
  "codes": [
    {
      "code": "LDConfigureGlobalLogger(LD_LOG_TRACE, LDBasicLogger);",
      "language": "text"
    }
  ]
}
[/block]
You can also use your own custom log function:
[block:code]
{
  "codes": [
    {
      "code": "static void\nmyCustomLogger(const LDLogLevel level, const char *const text)\n{\n    printf(\"[%s] %s\\n\", LDLogLevelToString(level), text);\n}
LDConfigureGlobalLogger(LD_LOG_TRACE, myCustomLogger);\n",
      "language": "c"
    }
  ]
}
[/block]
Note that the SDK does not lock on any logging. Ensure that your implementation is thread safe.

Whether using a custom logger or the default one, `LDConfigureGlobalLogger` must  be called before the client is initialized; you cannot modify logging while the client is running.
## Shutting down
To fully uninitialize the C SDK resources you must use `LDClientClose`. The operation will block until all resources have been freed. It is not safe to use any API methods after this process is initiated.
[block:code]
{
  "codes": [
    {
      "code": "LDClientClose(client);",
      "language": "c"
    }
  ]
}
[/block]