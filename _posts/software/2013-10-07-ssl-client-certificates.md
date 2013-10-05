---
layout: post
title: Ssl Client Certificates
description: "SslStream won't send my client certificates."
tags: [mongodb, .net, ssl, x509]
image:
  feature: abstract-7.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: false
---

> For an SSL Client Certificate to get sent with [SslStream](http://msdn.microsoft.com/en-us/library/system.net.security.sslstream.aspx), the PrivateKey member *must* be set.

It took me a little while to figure out what was going on.  I had a .pem file that includes both the certificate as well as the private key. I finally checked the PrivateKey property on the X509Certificate2 instance and it was null.

{% highlight csharp %}
var cert = new X509Certificate2("client.pem");
Debug.Assert(cert.PrivateKey != null);
{% endhighlight %}

Oops.  What is wrong with .NET? So I split them out into client.crt and client.key.  At this point, using client.crt file was exactly the same, but I now needed to assign the private key. Hmmm, that is apparently non-trivial as well and I actually found no good examples of this.  A lot of [StackOverflow](http://stackoverflow.com) questions indicated to use BouncyCastle.  That is all well and good, but surely there is a better way.

As I had created these files using [openssl](http://www.openssl.org/), it turns out they can create something that .NET does understand where the certificate and the private key exist in one file.  Generated liked below...

{% highlight bash %}
openssl pkcs12 -export -out client.pfx -inkey client.key -in client.crt
{% endhighlight %}

... the .NET framework will pick up both the certificate and assign the PrivateKey.  At this point, passing this certificate into the SslStream's [AuthenticateAsClient](http://msdn.microsoft.com/en-us/library/ms145061.aspx) method will send the certicate along for the ride.

{% highlight csharp %}
var cert = new X509Certificate2("client.pem");
Debug.Assert(cert.PrivateKey != null);

new SslStream(
  innerStream: GetNetworkStream(),
  leaveInnerStreamOpen: false);

sslStream.AuthenticateAsClient(
  targetHost: "localhost", 
  clientCertificates: new X509Certificate2Collection(cert),
  enabledSslProtocols: SslProtocols.Default,
  checkCertificateRevocation: true);
{% endhighlight %}