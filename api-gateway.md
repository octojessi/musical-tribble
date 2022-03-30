## Getting started

### Scope
This document is intended to be a quick-start guide to creating an API gateway from scratch and enabling custom rate limits for GitHub Enterprise as a proof of concept. To that end, a basic configuration is detailed for functionality. This document is not intended to be a comprehensive, advanced configuration guide for Gravitee or API gateways in general.

### Why use an API gateway?
The use of an API gateway can help with a number of challenges, as well as unify management of your various APIs within your infrastructure. The purpose of this gateway demonstration is to show how we can rate-limit users, while leaving CI processes unlimited. This has specific context within GitHub Enterprise.

### Components
There are many API gateways to choose from, some commercial and some open source. For the purposes of this demonstration, [Gravitee](https://gravitee.io) was chosen due to its being open source and not feature limited. Before going to production, proper testing should be done and architecture should be considered.

This _Docker Compose_ file builds the following services:

- MongoDB
- ElasticSearch
- Management API
- Management UI
- API Gateway

## How to run
This can be run using Docker Compose, or Kubernetes Kompose. The following command is for running this as a daemon with Docker Compose.

```bash
docker-compose up -d
```

If you would like to tear it down, simply run:

```bash
docker-compose down
```

And finally, to clean up your test environment, run:

```bash
docker-compose rm
```

## How to use

| Service | URL |
| --- | --- |
| Management API | `http://[HOSTNAME]:8005` |
| API Gateway | `http://[HOSTNAME]:8000` |
| Management UI | `http://[HOSTNAME]:8002` |

Once you've started your services, login to the management UI and create your API endpoints.

![gravitee-01](https://user-images.githubusercontent.com/865381/50696892-b2fca880-100e-11e9-8b6d-ee1707645b4b.png)

The default administrative username and password is `admin`/`admin`.

![gravitee-02](https://user-images.githubusercontent.com/865381/50696893-b2fca880-100e-11e9-87ad-6c4fedccda8b.png)

Once logged in, navigate to the _Administration_ section to create your APIs.

![gravitee-03](https://user-images.githubusercontent.com/865381/50696895-b2fca880-100e-11e9-826e-4d11817d3e3d.png)

Click the blue `+` icon

![gravitee-04](https://user-images.githubusercontent.com/865381/50698619-65366f00-1013-11e9-8a35-46a2ace2abb1.png)

If you are configuring this for the first time and don't have a backup configuration file, click the blue `->` arrow to create and API from scratch.

![gravitee-05](https://user-images.githubusercontent.com/865381/50698620-65cf0580-1013-11e9-882f-60611f553dfe.png)

Next, fill in the information for this _new_ API that you are creating. The details here will be what Gravitee serves to the user.

| Field | Description |
| --- | --- |
| Name | The name of this API endpoint |
| Version | Semantic versioning for this endpoint |
| Description | User-friendly description |
| Context-path | The URL path that will follow the hostname (i.e. /api/v3) |

These details are what the user will see, not what Gravitee points to. In this example we are using `/api/v3` as the context-path, but it can be anything.

![gravitee-06](https://user-images.githubusercontent.com/865381/50698621-65cf0580-1013-11e9-9c33-f44949e37458.png)

Next, provide the target endpoint. In our case, `https://github.example.com/api/v3` is our internal GitHub Enterprise instance, so that will be our endpoint.

![gravitee-07](https://user-images.githubusercontent.com/865381/50698623-65cf0580-1013-11e9-9070-a59199b45c2f.png)

Next we'll create a _Plan_, which will be used to determine our rate limiting, security methods, quotas, etc.

**The _Name_ of this plan matters. If you re-use the name, the metrics will cross into other APIs, so you will find yourself hitting rate limits much faster!**

| Field | Description |
| --- | --- |
| Name | Unique identifier for this plan (do not re-use across APIs) |
| Security Type | OAuth, Token, Keyless (public), etc |
| Description | User-friendly description of this plan |
| `Rate Limit` Max Requests | Maximum number of requests for this plan |
| `Rate Limit` Period Time | The period of time before the max requests is reset |
| `Rate Limit` Time Unit | Minutes or seconds for _Period Time_ |
| `Quota` Max Requests | Maximum number of requests allowed |
| `Quota` Period Time | The period of time before the quota resets |
| `Quota` Time Unit | Hours, Days, Weeks or Months |

Using a `keyless` security type will pass-through your API traffic to the endpoint without requiring intermediary authorization. This might be useful in some cases, like when using Azure Pipelines, but may not be ideal when you want to prevent users from targeting endpoints they should not have access to. Proper planning should be considered before going into production.

![gravitee-08](https://user-images.githubusercontent.com/865381/50698624-65cf0580-1013-11e9-850f-5f541665fd87.png)

If you have documentation for this API you may provide it here. Otherwise, click _Skip_.

![gravitee-09](https://user-images.githubusercontent.com/865381/50698626-65cf0580-1013-11e9-95b7-1cc9e147eb78.png)

Review the information, the click _Create and start the API_.

![gravitee-10](https://user-images.githubusercontent.com/865381/50698627-65cf0580-1013-11e9-9354-ce3391aac7f1.png)

Confirm the creation.

![gravitee-11](https://user-images.githubusercontent.com/865381/50698628-66679c00-1013-11e9-82e8-9a1ee61522b7.png)

Now your API is available when you click on **_APIs_**
![gravitee-12](https://user-images.githubusercontent.com/865381/50698629-66679c00-1013-11e9-8eb8-99e49f52b2b4.png)
![gravitee-13](https://user-images.githubusercontent.com/865381/50698630-66679c00-1013-11e9-9d5c-356cb4660acc.png)

Repeat this step for each endpoint that you want custom rules for. 

In this example, our rate limit was set to 1 request per minute in the screenshot. This will result in the following output:

```bash
$ curl localhost:8000/api/v3
{
  "message": "Must authenticate to access this API.",
  "documentation_url": "https://developer.github.com/enterprise/2.15/v3"
}
$ curl localhost:8000/api/v3
{
  "message" : "Rate limit exceeded ! You reach the limit of 1 requests per 1 minutes",
  "http_status_code" : 429
}
```
In setting up another endpoint, also pointing to GitHub, create one that has no rate limit and a path-context of `/ci` and you can see that users hitting the `/ci` context are not rate limited, while users hitting the `/api/v3` context are. 

```bash
$ curl localhost:8000/ci
{
  "message": "Must authenticate to access this API.",
  "documentation_url": "https://developer.github.com/enterprise/2.15/v3"
}
$ curl localhost:8000/ci
{
  "message": "Must authenticate to access this API.",
  "documentation_url": "https://developer.github.com/enterprise/2.15/v3"
}
$ curl localhost:8000/ci
{
  "message": "Must authenticate to access this API.",
  "documentation_url": "https://developer.github.com/enterprise/2.15/v3"
}
$ curl localhost:8000/ci
{
  "message": "Must authenticate to access this API.",
  "documentation_url": "https://developer.github.com/enterprise/2.15/v3"
}
$ curl localhost:8000/ci
{
  "message": "Must authenticate to access this API.",
  "documentation_url": "https://developer.github.com/enterprise/2.15/v3"
}
$ curl localhost:8000/api/v3
{
  "message" : "Rate limit exceeded ! You reach the limit of 1 requests per 1 minutes",
  "http_status_code" : 429
}
```

There are several ways to achieve this same outcome, so you should familiarize yourself with the API Gateway you choose and find the right configuration for your goals.
