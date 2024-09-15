---
title: Rapid analyzing Sonar HTTP datasets
date: 2018-05-04
tags:
  - threatintel
  - recon
  - sonar
  - scans
url: /blog/sonargrep
ShowReadingTime: true
---


Sometimes you need to gather threat intelligence data as quickly as possible,
and Rapid7's Project Sonar Opendata can provide great insights.

However, there's a challenge: you can't easily grep the HTTP response body with
the lovely [jq](https://stedolan.github.io/jq/) tool because the data field in
the resulting JSON is base64 encoded:
```json
{
  "data": "SFRUUC8xLjAgNDAwIEJhZCBSZXF1ZXN0DQpTZXJ2ZXI6IEFrYW1haUdIb3N0DQpNaW1lLVZlcnNpb246IDEuMA0KQ29udGVudC1UeXBlOiB0ZXh0L2h0bWwNCkNvbnRlbnQtTGVuZ3RoOiAyMDgNCkV4cGlyZXM6IE1vbiwgMjMgQXByIDIwMTggMDc6NDA6MjkgR01UDQpEYXRlOiBNb24sIDIzIEFwciAyMDE4IDA3OjQwOjI5IEdNVA0KQ29ubmVjdGlvbjogY2xvc2UNCg0KPEhUTUw+PEhFQUQ+CjxUSVRMRT5JbnZhbGlkIFVSTDwvVElUTEU+CjwvSEVBRD48Qk9EWT4KPEgxPkludmFsaWQgVVJMPC9IMT4KVGhlIHJlcXVlc3RlZCBVUkwgIiYjOTE7bm8mIzMyO1VSTCYjOTM7IiwgaXMgaW52YWxpZC48cD4KUmVmZXJlbmNlJiMzMjsmIzM1OzkmIzQ2OzFmMzEzMjE3JiM0NjsxNTI0NDY5MjI5JiM0NjsyNDFiOGFmCjwvQk9EWT48L0hUTUw+Cg==",
  "host": "REDACTED",
  "ip": "REDACTED",
  "path": "/",
  "port": 80,
  "vhost": "REDACTED"
}
```

While you could probably grep this using a decent bash script, I believe I have
a better option.

_Update_:

> Apparently the latest `jq` version
[has](https://github.com/stedolan/jq/issues/47) a `@base64d` filter so there is
definitely another way to accomplish this using only jq. But still it's nice to
have multiple [options](https://hub.docker.com/r/ilyaglow/jq).

# My Solution
To overcome this issue, I wrote a quick and dirty app called
[sonargrep](https://github.com/ilyaglow/sonargrep). It accepts gzipped data
from stdin and tries to find host entries that contain the data you're looking
for.
You can install sonargrep with the following command (assuming Go is already installed):
```sh
$ go install github.com/ilyaglow/sonargrep
```

# Usage

For example, here's how to get a list of WordPress-related IPs:
```sh
$ curl -L -s https://opendata.rapid7.com/sonar.https/2018-04-24-1524531601-https_get_443.json.gz \
    | sonargrep -w wordpress -i \
    | jq -r '.ip'
```

It accomplishes the following:
* Greps records containing "wordpress" (case-insensitive) in their HTTP response body from the `sonar.https` dataset.
* Extracts IPs using jq.
* Does all this without saving the 50GB file to disk.

_Update_:
I've created a Docker image for the latest jq GitHub version, so you can
perform the same task like this:
```sh
$ alias jq="sudo docker run -i --rm ilyaglow/jq"
$ curl -L -s https://opendata.rapid7.com/sonar.https/2018-04-24-1524531601-https_get_443.json.gz \
    | gunzip \
    | jq -r 'select((.data | @base64d) | match(".*wordpress.*", "i")) | .ip'
```

# Workflow

Here's how my opendata research workflow with
[sonargrep](https://github.com/ilyaglow/sonargrep) now looks: 

* Spin up a DigitalOcean droplet
* Start grepping
* Wait for an hour or so, while playing with the results that come in real-time
* Terminate the droplet

This approach allows for quick and efficient analysis of large datasets without
the need for extensive local storage or processing power.
