---
title: "Mounting Virtual Media Using Redfish on iDRAC 8"
layout: post
date: 2021-12-19
image: /assets/images/2021-12-19-mounting-virtual-media-drac8/redfish.jpg
headerImage: true
tag:
- dell
- drac
- virtual media
category: blog
blog: true
author: jamesdenton
description: "Mounting Virtual Media Using Redfish on iDRAC 8"
---

Using HP iLO 4 for the last few years, you could say I've been a bit *spoiled* with some of the conveniences provided within.

So, imagine my surprised when firing up my recently-acquired Dell R630 for the first time, only to find that HTTP-based virtual media was not an option in the UI!
<!--more-->
Some time later I came to find out that mounting virtual media requires the use of the API. No big deal, except that I had not found an obvious guide to using the included API (which I later found out was Redfish v1). It took some time to find a good, working example [here](https://github.com/dell/iDRAC-Redfish-Scripting/issues/24)

And now, I'll save you some time and trouble by demonstrating a mount and eject operation via curl.

## Mounting Virtual Media via HTTP/S

To attach virtual media, one must use the following format:


```
POST
URI: https://<drac_ip_address>/redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.InsertMedia
BODY:
{
"Image": "http://<web_server>/<image_name>.iso"
}
```

The following example will ***mount*** an ISO using curl:

```
curl -v -k -X POST https://172.19.0.25/redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.InsertMedia \
-u root \
-H 'Content-Type: application/json' \
-d '{"Image": "http://172.22.0.5/VMware-VMvisor-Installer-7.0U2-17630552.x86_64.iso"}'
```


A successful operation will result in an HTTP `204` status code:

```
> POST /redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.InsertMedia HTTP/1.1
> Host: 172.19.0.25
> Authorization: Basic cm9vdDpjYWx2aW5jYWx2aW4=
> User-Agent: curl/7.77.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 81
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 204 No Content
< Strict-Transport-Security: max-age=63072000
< Vary: Accept-Encoding
< Keep-Alive: timeout=60, max=199
< X-Frame-Options: SAMEORIGIN
< Content-Type: application/json; charset=utf-8
< Server: iDRAC/8
< Date: Mon, 20 Dec 2021 07:24:06 GMT
< Cache-Control: no-cache
< Content-Length: 0
< Connection: Keep-Alive
< Accept-Ranges: bytes
<
* Connection #0 to host 172.19.0.25 left intact
```

Attempting to mount an ISO with something already attached will result in a `500` error:

```
> POST /redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.InsertMedia HTTP/1.1
> Host: 172.19.0.25
> Authorization: Basic cm9vdDpjYWx2aW5jYWx2aW4=
> User-Agent: curl/7.77.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 81
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 500 Internal Server Error
< Strict-Transport-Security: max-age=63072000
< OData-Version: 4.0
< Vary: Accept-Encoding
< Keep-Alive: timeout=60, max=199
< X-Frame-Options: SAMEORIGIN
< Content-Type: application/json;odata.metadata=minimal;charset=utf-8
< Server: iDRAC/8
< Date: Mon, 20 Dec 2021 07:22:56 GMT
< Cache-Control: no-cache
< Content-Length: 424
< Connection: Keep-Alive
< Access-Control-Allow-Origin: *
< Accept-Ranges: bytes
<
{"error":{"@Message.ExtendedInfo":[{"Message":"The Virtual Media image server is already connected.","MessageArgs":[],"MessageArgs@odata.count":0,"MessageId":"IDRAC.1.6.VRM0012","RelatedProperties":[],"RelatedProperties@odata.count":0,"Resolution":"No response action is required.","Severity":"Informational"}],"code":"Base.1.2.GeneralError","message":"A general error has occurred. See ExtendedInfo for more information"}}
```

## Ejecting Virtual Media

To eject virtual media, one must use the following format:

```
Method: POST
URI: https://<idrac_ip_address>/redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.EjectMedia
BODY:
{}
```

The following example will ***eject*** an ISO using curl:

curl -v -k -X POST https://drac_ip_address>/redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.EjectMedia \
-u root \
-H 'Content-Type: application/json' \
-d '{}'

And, yes, the payload is required (and empty) on an eject operation.


A successful operation will result in an HTTP `204` status code:

```
> POST /redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.EjectMedia HTTP/1.1
> Host: 172.19.0.25
> Authorization: Basic cm9vdDpjYWx2aW5jYWx2aW4=
> User-Agent: curl/7.77.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 2
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 204 No Content
< Strict-Transport-Security: max-age=63072000
< Vary: Accept-Encoding
< Keep-Alive: timeout=60, max=199
< X-Frame-Options: SAMEORIGIN
< Content-Type: application/json; charset=utf-8
< Server: iDRAC/8
< Date: Mon, 20 Dec 2021 07:23:29 GMT
< Cache-Control: no-cache
< Connection: Keep-Alive
< Transfer-Encoding: chunked
< Accept-Ranges: bytes
<
* Excess found: excess = 5 url = /redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.EjectMedia (zero-length body)
```

Attempting to eject an ISO that is not attached will result in a `500` error:

```
> POST /redfish/v1/Managers/iDRAC.Embedded.1/VirtualMedia/CD/Actions/VirtualMedia.EjectMedia HTTP/1.1
> Host: 172.19.0.25
> Authorization: Basic cm9vdDpjYWx2aW5jYWx2aW4=
> User-Agent: curl/7.77.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 2
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 500 Internal Server Error
< Strict-Transport-Security: max-age=63072000
< OData-Version: 4.0
< Vary: Accept-Encoding
< Keep-Alive: timeout=60, max=199
< X-Frame-Options: SAMEORIGIN
< Content-Type: application/json;odata.metadata=minimal;charset=utf-8
< Server: iDRAC/8
< Date: Mon, 20 Dec 2021 07:20:35 GMT
< Cache-Control: no-cache
< Content-Length: 774
< Connection: Keep-Alive
< Access-Control-Allow-Origin: *
< Accept-Ranges: bytes
<
{"error":{"@Message.ExtendedInfo":[{"Message":"No Virtual Media devices are currently connected.","MessageArgs":[],"MessageArgs@odata.count":0,"MessageId":"IDRAC.1.6.VRM0009","RelatedProperties":[],"RelatedProperties@odata.count":0,"Resolution":"No response action is required.","Severity":"Critical"},{"Message":"The request failed due to an internal service error.  The service is still operational.","MessageArgs":[],"MessageArgs@odata.count":0,"MessageId":"Base.1.2.InternalError","RelatedProperties":[],"RelatedProperties@odata.count":0,"Resolution":"Resubmit the request.  If the problem persists, consider resetting the service.","Severity":"Critical"}],"code":"Base.1.2.GeneralError","message":"A general error has occurred. See ExtendedInfo for more information"}}
```

#
## Summary

Now that I know what to do, using the API can be faster (and more flexible) that the UI. However, getting there was a bit of a challenge. Hopefully this helps you on your journey.

---
If you have some thoughts or comments on this article, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.
