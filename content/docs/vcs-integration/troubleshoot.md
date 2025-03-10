---
title: Troubleshoot
---

_If you need further assistance setting up GitOps, feel free to [reach us](/docs/faq/#how-to-reach-us)._

## Committed migration file does not trigger issue creation

When a migration file is committed to the VCS, VCS will send a webhook event to Bytebase. There are two error categories:

- Bytebase has received webhook events

- Bytebase has not received any webhook event

### Bytebase has received webhook events

In this case, if you visit your project activity page, you should find an activity event suggesting Bytebase has received the webhook event. However, the committed file doesn't match the configured path and the event is ignored.

![activity-warning](/content/docs/vcs-integration/troubleshoot/activity-warning.webp)

You should check the committed file conforms exactly to the [naming convention](/docs/vcs-integration/name-and-organize-schema-files) and the directory structure conforms to the [layout](/docs/vcs-integration/name-and-organize-schema-files#file-organization). Bytebase file name match is **case-sensitive**. Below screenshot shows some common mistakes:

![commit](/content/docs/vcs-integration/troubleshoot/commits.webp)

![activity](/content/docs/vcs-integration/troubleshoot/activities.webp)

1. ❌ File name mismatches.
1. ❌ File name case mismatches.
1. ❌ Environment id mismatches. Note we should match `Environment ID` instead of `Enviornment Name`.
   ![environment-id](/content/docs/vcs-integration/troubleshoot/environment-id.webp)
1. ✅ Match.

### Bytebase has not received any webhook event

In this case, you should visit the your VCS provider's webbook page and check the webhook event history.

1. Make sure Bytebase has configured a proper [External URL](/docs/get-started/install/external-url).

1. Make sure that configured URL is network accessible from VCS.

You can [turn on debug mode](/docs/faq/#toggle-debug-mode-at-runtime). Then on the GitOps settings page, visit
the linked VCS webhook page to check the details.

![webhook](/content/docs/vcs-integration/troubleshoot/webhook.webp)

## Failed to create webhook xxx, status code: 422 for GitLab

If you configure [External URL](/docs/get-started/install/external-url) with the private IP such as `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, you will need to enable [Allow requests to the local network from webhooks and integrations](https://docs.gitlab.com/ee/security/webhooks.html#allow-outbound-requests-to-certain-ip-addresses-and-domains) first.

![activity-warning](/content/docs/vcs-integration/troubleshoot/gitlab-allow-internal-request.webp)

## OAuth CORS error with old GitLab version

When using old GitLab version (e.g. 9.4.0) to setup VCS integration, you may encounter OAuth error like [this one](https://github.com/bytebase/bytebase/issues/467):

![oauth-failed](/content/docs/vcs-integration/troubleshoot/oauth-failed.webp)

This is a common problem in the old GitLab verison:

- https://gitlab.com/gitlab-org/gitlab-foss/-/issues/19470
- https://gitlab.com/gitlab-org/gitlab/-/issues/300077

### Verify the problem

Open your browser devtool with `F12`, check the `Network` section. If the latest token request with **`CORS error`** status, we can be certain that it's the `/oauth/token` api CORS error inside GitLab.

![cors-error](/content/docs/vcs-integration/troubleshoot/cors-error.webp)

### Potential solution

We cannot change GitLab source code to add the `Access-Control-Allow-Origin: *` to `/oauth/token` response header, but can use Nginx as a reverse proxy for GitLab (the other proxy service works the similar way).

**CORS solution with Nginx**

Add `add_header` codes directive to the base path location block of your Nginx GitLab configuration file.

```nginx
server {
  ...
  location / {
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Headers' 'Authorization,Accept,Origin,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range';
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE,PATCH';

    if ($request_method = 'OPTIONS') {
      add_header 'Access-Control-Allow-Origin' $http_origin;
      add_header 'Access-Control-Max-Age' 1728000;
      add_header 'Content-Type' 'text/plain charset=UTF-8';
      add_header 'Content-Length' 0;
      return 204;
    }

    if ($request_method != GET) {
      add_header 'Access-Control-Allow-Origin' '*';
    }
    ...
  }
  ...
}
```

Run the following command to reload your updated config file.

```bash
sudo nginx -s reload
```

---

Afterwards, try the GitLab setup again.
