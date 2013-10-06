---
layout: post
title: Ssl Client Certificates
description: For an SSL Client Certificate to get sent with SslStream, the PrivateKey member *must* be set.
tags: [mongodb, .net, ssl, x509]
image:
  feature: abstract-7.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> For an SSL Client Certificate to get sent with [SslStream](http://msdn.microsoft.com/en-us/library/system.net.security.sslstream.aspx), the PrivateKey member *must* be set.

[MongoDB](http://mongodb.org) is implementing [x509 authentication](https://jira.mongodb.org/browse/SERVER-7961).  This is great for enterprises who issue users certificates or for trusted servers communicating over SSL already.  When the server adds support, it means all the drivers also need to add support for it.  As we already built in [Kerberos support](http://docs.mongodb.org/manual/tutorial/control-access-to-mongodb-with-kerberos-authentication/), we made the system trivial to add other authentication systems and it took all of about 15 minutes to include x509 support.  However, testing it took hours of my time.

Enabling ssl can be completely done via a connection string, but to add in a client certificate for authentication requires some code. [^1]

{% highlight csharp %}
var cert = new X509Certificate2("client.pem");

var settings = new MongoClientSettings
{
    Credentials = new[] 
    {
        MongoCredential.CreateMongoX509Credential("CN=client,OU=kerneluser,O=10Gen,L=New York City,ST=New York,C=US")
    },
    SslSettings = new SslSettings
    {
        ClientCertificates = new[] { cert },
    },
    UseSsl = true
};

var client = new MongoClient(settings);
{% endhighlight %}

As you can see, I had a .pem file. It included both the certificate and the private key, but the client was sending the certificate, even though it's obvious I included it. Finally I checked the PrivateKey property on the X509Certificate2 instance and it was null. Hmmm.  What is wrong with .NET?

I split the pem file into client.crt and client.key with the intention of loading client.crt and then assigning the private key. Apparently, that is non-trivial.  In fact, I found no good examples of this.  However, a number of people had success on [StackOverflow](http://stackoverflow.com) using BouncyCastle, an open-source crypto API (ported from Java I think).  I obviously have nothing wrong with this, but how can the .NET framework not support this?

It turns out [openssl](http://www.openssl.org/) can create something that .NET does understand that includes both the certificate and the private key in one file and, when loaded, will have the PrivateKey already applied.  Running the below openssl command will generate client.pfx.  You'll need to set a password which we'll use when we construct the X509Certificate2.

{% highlight bash %}
openssl pkcs12 -export -out client.pfx -inkey client.key -in client.crt
{% endhighlight %}

At this point, passing this certificate into the SslStream's [AuthenticateAsClient](http://msdn.microsoft.com/en-us/library/ms145061.aspx) method will send the certicate along for the ride.

{% highlight csharp %}
var cert = new X509Certificate2("client.pfx", "mySuperSecretPassword");
{% endhighlight %}


[^1]: The string in the CreateMongoX509Credential is the RFC2253 format for the subject.  It can be found using `openssl x509 -in client.pem -inform PEM -subject -nameopt RFC2253`