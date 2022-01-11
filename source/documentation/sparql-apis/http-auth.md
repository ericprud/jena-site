---
title: HTTP Authentication
---

<i>[Old documentation](../archive/versions/http-auth-old.html) (Jena 3.1.1 to Jena 4.2.0)</i>

Jena 4.3.0 and later uses the JDK `java.net.http` package. Jena adds API support
for challenge-based authentication and also provide HTTP digest authentication.

## Authentication {#auth}

There are 5 variations:

1. Basic authentication
2. Challenge-Basic authentication
3. Challenge-Digest authentication
4. URL user (that is, `user@host.net` in the URL)
5. URL user and password in the URL (that is, `user:password@host.net` in the URL)

Basic authentication occurs where the app provides the user and password
information to the JDK `HttpClient` and that information is always used when
sending HTTP requests with that `HttpClient`. It does not require an initial
request-challenge-resend to initiate. This is provided natively by the `java.net.http`
JDK code. See `HttpClient.newBuilder().authenticate(...)`.

Challenge based authentication, for "basic" or "digest", are provided by Jena.
The challenge happens on the first contact with the remote endpoint and the
server returns a 401 response with an HTTP header saying which style of
authentication is required. There is a registry of users name and password for
endpoints which is consulted and the appropriate
[`Authorization:`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization)
header is generated then the request resent. If no registration matches, the 401
is passed back to the application as an exception.

Because it is a challenge response to a request, the request must be sent twice,
first to trigger the challenge and then again with the HTTP authentication
information.  To make this automatic, the first request must not be a streaming
request (the stream is not repeatable). All HTTP request generated by Jena are
repeatable.

The URL can contain a `userinfo` part, either the `users@host` form, or the `user:password@host` form.
If just the user is given, the authentication environment is consulted for registered users-password information. If user and password is given, the details as given are used. This latter form is not recommended and should only be used if necessary because the password is in-clear in the SPARQL
query.

### JDK HttpClient.authenticator

```java
    // Basic or Digest - determined when the challenge happens.
    AuthEnv.get().registerUsernamePassword(URI.create(dataURL), "user", "password");
    try ( QueryExecution qExec = QueryExecutionHTTP.service(dataURL)
            .endpoint(dataURL)
            .queryString("ASK{}")
            .build()) {
        qExec.execAsk();
    }
```

alternatively, the java platform provides basic authentication. 
This is not challenge based - any request sent using a `HttpClient` configured 
with an authenticator will include the authentication details. 
(Caution - including sending username/password to the wrong site!).
Digest authentication must use `AuthEnv.get().registerUsernamePassword`.

```java
    Authenticator authenticator = AuthLib.authenticator("user", "password");
    HttpClient httpClient = HttpClient.newBuilder()
            .authenticator(authenticator)
            .build();
```

```java
    // Use with RDFConnection      
    try ( RDFConnection conn = RDFConnectionRemote.service(dataURL)
            .httpClient(httpClient)
            .build()) {
        conn.queryAsk("ASK{}");
    }
```

```java
    try ( QueryExecution qExec = QueryExecutionHTTP.service(dataURL)
            .httpClient(httpClient)
            .endpoint(dataURL)
            .queryString("ASK{}")
            .build()) {
        qExec.execAsk();
    }
```

### Challenge registration

`AuthEnv` maintains a registry of credentials and also a registry of which service URLs
the credentials should be used. It supports registration of endpoint prefixes so that one
registration will apply to all URLs starting with a common root.

The main function is `AuthEnv.get().registerUsernamePassword`.

```java
   // Application setup code 
   AuthEnv.get().registerUsernamePassword("username", "password");
```

```java
   ...
   try ( QueryExecution qExec = QueryExecutionHTTP.service(dataURL)
        .endpoint(dataURL)
        .queryString("ASK{}")
        .build()) {
       qExec.execAsk();
   }
```

When an HTTP 401 response with an `WWW-Authenticate` header is received, the Jena http handling code
will will look for a suitable authentication registration (exact or longest prefix), and retry the
request. If it succeeds, a modifier is installed so all subsequent request to the same endpoint will
have the authentication header added and there is no challenge round-trip.

### <tt>SERVICE</tt>

The same mechanism is used for the URL in a SPARQL `SERVICE` clause.  If there is a 401 challenge,
the registry is consulted and authentication applied.

In addition, if the SERVICE URL has a username as the `userinfo` (that is, `https://users@some.host/...`),
that user name is used to look in the authentication registry.

If the `userinfo` is of the form "username:password" then the information as given in the URL is
used.

```java
    AuthEnv.get().registerUsernamePassword(URI.create("http://host/sparql"), "u", "p");
     // Registration applies to SERVICE.
    Query query = QueryFactory.create("SELECT * { SERVICE <http://host/sparql> { ?s ?p ?o } }");
    try ( QueryExecution qExec = QueryExecution.create().query(query).dataset(...).build() ) {
        System.out.println("Call using SERVICE...");
        ResultSet rs = qExec.execSelect();
        ResultSetFormatter.out(rs);
    }
```

## Examples

[jena-examples:arq/examples/auth/](https://github.com/apache/jena/tree/main/jena-examples/src/main/java/arq/examples/auth).