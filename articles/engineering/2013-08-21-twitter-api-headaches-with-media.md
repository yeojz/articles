---
title: "Twitter API: OAuth and Update_with_media"
date: 2013-08-21
first_writte: 2013-08-21
note: ""
---

In the past week, I had a project which required me to implement a server-to-api upload of images. Being client-side oriented, I didn't think much of the issue, until I hit a snag — actually creating the multipart request.

The python _urllib2_ and _poster_ libraries helped alot, but somehow, I was still getting 401, 400, and sometimes 215 twitter error codes. Weeks in and multiple searches on the internet didn't lead to anything much.

But finally, I hit the solution… a solution so simple that I could tear, and a solution which I tried on a whim:

> **Removal of the "status" field before OAuth signing request is made.**

I was already **removing the "media[]" field**. But it didn't occur to me that "status" needs to be removed as well, nor did I suspect it to be the cause of my problems.

This requirement was also not mentioned in the Twitter API docs. It may be in the OAuth protocol, but I didn't look into that, so I don't really know.

Anyway, here's some sample code. Hopefully it will save someone from unnecessary code-wrestling.

```python

import httplib
import oauth
from poster.encode import multipart_encode
from poster.streaminghttp import register_openers


# Registering poster's methods
register_openers()


# Defining OAuth method
signature_method = oauth.OAuthSignatureMethod_HMAC_SHA1()


# Some API Settings
domain = "http://api.twitter.com"
url = domain + "/1.1/statuses/update_with_media.json"
consumer = oauth.OAuthConsumer(KEY, SECRET)


# Creating the signing request
# Note the parameter field is set to None. {} works too
req = oauth.OAuthRequest.from_consumer_and_token(
		consumer,
		token=TOKEN,
		http_url=url,
		parameters=None,
		http_method="POST")

req.sign_request(signature_method, consumer, TOKEN)

# I'm using httplib here. urllib2 should be possible as well
conn = httplib.HTTPSConnection(domain)

# Set the actual content
content = {
'status': "YOUR STATUS MESSAGE",
# Can be open(media, 'rb') or maybe urllib2.urlopen(media).read()
'media[]': "YOUR MEDIA"
}

# Generate multipart data and headers from content
datagen, headers = multipart_encode(content)

# Append OAuth Authorization headers to generated headers
headers.update(req.to_header())

# Body to string
body = "".join(datagen)

# Make your request
conn.request("POST", req.get_url_location, body, headers)

# Get Result
response = conn.getResponse()
print response.read()
```
