WebID-Token Authentication
=============================================

This document specifies an HTTP authentication mechanism suitable for use in situations where an HTTP client is attempting to authenticate to another HTTP server on a different domain. It is very easy to implement and requires no extra crypto, nor cookies.

For illustration purposes, we'll say Alice is a user with a WebID at `https://alice.example/profile#me` and wants to access a protected resource at `https://bob.example/foo`.

It is imperative that no requests are passed in clear, and that HTTPS is deployed on all the servers.

Status
------

Pending implementation.


Walkthrough
-----------

**Step 1.**  Alice performs an HTTP GET on an access-controlled page served by Bob, but does not authenticate herself, so Bob returns a 401 error. The response includes a header telling Alice she can authenticate using this protocol and try again, as well as a unique and random string (i.e. nonce).

```http
> GET /foo HTTP/1.1
> Host: bob.example
...
< HTTP/1.1 401 Authorization Required
< WWW-Authenticate: WebID-Token nonce="xyz123"
...
```

The response uses the standard WWW-Authenticate HTTP header with a new keyword
for this new authentication type.

**Step 2.**  Alice creates a empty resource containing the SHA1 sum of the nonce, under a dedicated container (i.e. folder) where she has write access (e.g. `alice.example/tokens/`). The hashing is required in order to avoid writing any arbitrary data stored in the nonce.

`sha1sum('xyz123') -> 2b743ea5699560665032496d957cd8c0075029d5`

```http
> PUT /tokens/2b743ea5699560665032496d957cd8c0075029d5
> Host: alice.example
...
< HTTP/1.1 201 Created 
```

Note: the location of this folder is linked to from her WebID profile like so:

```turtle
<#me> <http://www.w3.org/ns/solid/terms#tokens> </tokens/> .
```

**Step 3.**  Alice repeats the GET, this time including a header which identifies her via her WebID and includes the nonce:

```http
> GET /foo HTTP/1.1
> Host: bob.example
> Authorization: WebID-Token webid="https://alice.example/profile#me" nonce="xyz123"
...
```

The WebID string must be dereferenceable. Note that sending identity strings like this may reveal more to Bob than desirable.

**Step 4.**  Bob's server checks to see if Alice was able to create a resource containing the valid nonce value under the folder linked from her profile information:

  * **Step 4.1.** Bob's server dereferences Alice's WebID and checks if she points to a valid folder using the `tokens` term from the Solid vocabulary.
```http
> GET /profile HTTP/1.1
> Host: alice.example
...
<#me>
    <http://www.w3.org/ns/solid/terms#tokens> </tokens/> .
...
```
  
  * **Step 4.2.** In Alice's WebID profile, Bob's server then finds the URI of the tokens folder and concatenates the SHA1 sum of the nonce to it. Then it performs a simple `HEAD` request to check if the resource exists.
```http
> HEAD /tokens/2b743ea5699560665032496d957cd8c0075029d5 HTTP/1.1
> Host: alice.example
...
< HTTP/1.1 200 OK
```
  
**Step 5.**  Bob's server will now issue a bearer token that can then be passed to applications (as another form of binding Alice's identity to a specific credential on Bob's server) and then returns a successful response to Alice's 2nd `GET` request:

```http
< HTTP/1.1 200 OK
< Authorization: WebID-Bearer-Token abc789
...
```

Alice's application can now persist the bearer token in a private location, so that it can be reused by applications if they require credentials for bob's server. In doing so, she avoids the need to go through the complete handshake in the future.

A future request to Bob's server may look as follows:

```http
> GET /bar/ HTTP/1.1
> Host: bob.example
> Authorization: WebID-Bearer-Token abc789
...
< HTTP/1.1 200 OK
```


**Step 6.** Cleanup

Alice can now safely delete the temporary token resource she created on her server, as it is no longer needed.
```http
> DELETE /tokens/2b743ea5699560665032496d957cd8c0075029d5
> Host: alice.example
...
< HTTP/1.1 200 OK
```
