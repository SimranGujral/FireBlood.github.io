# Rustls and Servo

## Background and Motivation

Servo, Mozilla's parallel browser engine, is written in Rust. Servo aims to create a browser which is memory safe and fast, the next generation browser.Before I get to what this post is about, I think it’s worthwhile to explain the jargon surrounding it, including the importance of TLS, what RusTLS is, and the effort to integrate RusTLS and Servo.I believe the answers to these questions form the motivation for this project.
Before your browser connects to a website, it performs a 9 step handshake by means of which it makes sure that your connection is secure and the parties involved in the communication are not malicious. This is basically the SSL/TLS protocol. Every browser has its own implementation of the TLS protocol. As of today, Servo uses Rust-OpenSSL. Now, as I mentioned earlier, Servo is the supposed to be next generation browser which is written in Rust- which is inherently type safe and memory safe. However, its TLS stack is Rust-OpenSSL, which is just Rust bindings for OpenSSL which is written in C and assembly. One of the ambitions in the Servo community has been to move to Rust based crypto primitives for Servo's TLS stack.
We had a couple of things in mind, one was to make the certificate verification process parallel, to use a certificate verification library based on Rust based Crypto primitives, and to make the TLS stack much leaner*.

## Concepts
#### BoringSSL
BoringSSL is Google's fork of OpenSSL and was created in response to the Heartbleed bug found in OpenSSL. It has only the necessary features which Google required for their use.
#### Ring
Ring is a fork of BoringSSL which is written in Rust. It is a set of rust cryptographic primitives which are based of BoringSSL.
#### WebPKI
WebPKI is a certificate verification library which uses Ring’s cryptographic primitives for certificate verification.
#### Rustls
Rustls is the TLS stack written in Rust which uses WebPKI for certificate verification.
#### Hyper-Rustls
Hyper is a networking library. Hyper-Rustls uses Rustls to make its connections secure. Hyper-Rustls works as a connection between Hyper- a networking library and Rustls - a TLS implementation to make network connections secure.

Hence, it goes like this :

![Rustls Dependencies](https://github.com/SimranGujral/FireBlood.github.io/blob/master/RustlsDependencyChart.png)

<b>Figure 1 : </b>Rustls --> WebPKI --> Ring --> BoringSSL

For more Information:
<ul>
<li><a href="https://boringssl.googlesource.com/boringssl/">BoringSSL</a></li>

<li><a href="https://github.com/briansmith/ring">Ring</a></li>

<li><a href="https://github.com/briansmith/webpki">WebPKI</a></li>

<li><a href="https://github.com/ctz/rustls">RusTLS</a></li>

<li><a href="https://github.com/ctz/hyper-rustls">Hyper-RusTLS</a></li>
</ul>

## Concerns
So, this is how all the pieces of the puzzle fit together, and now we have to fit them into Servo.

However, when you set out to make a browser which is blazingly fast, you need to make sure that any piece you fit in does not slow it down.

Also, when the primary concern for your blazingly fast browser is security, you want to make sure the cryptographic protocols in the new 
version of the TLS stack that you use work correctly. 

Lastly, since you want to move to a leaner version, you want to try to make sure that  the stack supports most of the features that the web depends on. 

Hence, we come down to our three main concerns pre-integration:

1) Does Rustls support the most important and most of the features that the web needs and depends on?

2) Do these features work without glitches?

3) Is it fast enough, if not faster?

## Pre-Integration

There were two things we did to handle our concerns:
1) Feature Comparison between RusTLS and Rust-OpenSSL
2) Performance Comparison between RusTLS and Rust-OpenSSL

## Feature Comparison

This was important to handle concerns 1 and 2 and was done using the BoringSSL test suite. The BoringSSL test suite is basically a suite of unit tests which was designed for BoringSSL and can be used to check TLS implementations.
There is already a shim to test the TLS implementation for RusTLS.
It can be found here:

<a href="https://github.com/ctz/rustls/tree/master/examples/internal">RusTLS BoGo Shim</a>

I worked on a TLS implementation for Rust-OpenSSL to understand the difference in libraries and features between the two TLS implementations and tested this against the BoringSSL test suite.
The shim does not currently support all features tested by the BoringSSL test suite.
It does not pass all the tests posed by the BoringSSL test suite. However, this is not because the library is incorrect, its because the BoringSSL test suite was designed was BoringSSL and we are testing it on Rust-OpenSSL. The errors reported by Rust-OpenSSL are different in wording than the ones expected by the suite and this leads to failures inspite of the shim behaving correctly.
Currently the shim passes 245 tests and fails 251. 
This is still a work in progress.
However, this implementation greatly helped me in understanding the differences in the two libraries.


As I mentioned earlier, RusTLS already has a TLS implementation to be tested against the suite. RusTLS passes all tests it is being tested against in the BoringSSL test suite. A major point of concern which emerged for us during these tests was that RusTLS does not support TLS v 1.0 and TLS v 1.1. It also does not support DHE and SHA1 which Servo was using, but wants to remove this dependency as per this issue:

<a href="hhttps://github.com/servo/servo/issues/8581">https://github.com/servo/servo/issues/8581 </a>

<a href="https://github.com/servo/servo/pull/16535">https://github.com/servo/servo/pull/16535 </a>


## Performance Comparison

For the performance comparison, I created a benchmarking framework for Hyper-Rustls and Rust-OpenSSL which measured timings for every step of the handshake. Results showed that a major portion of the time before establishing a connection goes in configuring and creating a connector, certificate verification takes lesser time. This was one of the reasons for not pursuing parallel certificate verification, since the overhead was not much.

The framework was tested on 101 websites of the Alexa 500.

Results were obtained by aggregating it over 10 clean connections without caching.

#### Results for performance comparison

As mentioned earlier, we tested three things:

1) Time to create a connector

2) Time to perform certificate verification

3) Time to establish the complete connection (This is step 1+ step 2 + other steps which happen in the handshake)

Results showed that Rust-OpenSSL is always faster for creating a connector.

Rustls was always faster for certificate verification.

Rustls was faster for 90 out of 101 websites for establishing the complete connection.

The complete results can be found here:

<a href="https://github.com/SimranGujral/FireBlood.github.io/blob/master/Alexa_500Results.xlsx">Hyper-Rustls vs Rust-OpenSSL Benchmarking results </a>

The Benchmarking implementation can be found here:

<a href="https://github.com/SimranGujral/hyper-openssl/blob/benchmarks/examples/alexa_bench.rs">Benchmarking Implementation</a>

The implementation uses the exact config parameters that Servo uses.
(Config Parameters here refer to the certificates used for verification, and the ciphers used in servo.
159 certificates were used for benchmarking.)


## Hyper-Rustls integration with Servo

The results show a minor improvement in performance with Hyper-Rustls. Though the performance improvement is probably a nano second, it is is indicative of a lot of things.

Firstly, it means that we do not have a performance regression. 
This point is important because it means we can move to a Rust Based solution without a performance loss. It means that we can usew the rust crypto primitives in Ring without slowing down the process of verification

Secondly, it is important to note here that we performed this benchmarking with Hyper-Rustls v 0.10. This is because Servo's network currently uses an older version of Hyper. Hence, if the current version is bumped and the new version of Hyper- Rustls is used, improvements are possible.

We have successfully integrated Hyper-Rustls with Servo. The PR for this can be found here:

<a href="https://github.com/servo/servo/pull/17938">PR for Hyper-Rustls integration with Servo</a>


There are a few caveats with this integration though. These are the primary reasons behind it not being merged as of yet.
We are working on these issues as of today:


1) The integration works for WPT's but does not work for Android. We need to check on that

2) For the integration to pass, we need to change a few tests which depend on Hyper Rust-OpenSSL

## Future Work

Apart from correcting the issues with the android implementation and modifying the tests to make it relevant to Hyper-Rustls, we have a few things to check for:

1) Though Rustls passes the tests for the BoringSSL test suite, we should still check for security implications if any for it.

2) While comparing the performance of Hyper-Rustls and Rustls, we found that Hyper-Rustls was faster. Since Hyper-Rustls is just Hyper using Rustls, it is interesting to see why it is faster.

3) According to benchmarking results, there are 11 websites for which Rust-OpenSSL is faster in establishing a connection, it would be interesting to see the characteristics of the websites which connect faster using Rust-OpenSSL.

## FAQs:

Though I hope that the post above answers all questions, I know of a few which I have faced while explaining this project which are not answered above.
If these do not answer your questions, please leave a note and I will try to answer it to the best of my capabilities.

1) Why doesn't Servo use NSS?

A: There are no Rust Bindings for NSS. There exist Rust Bindings for OpenSSL. Hence, we use that.

2) Rustls does not support TLS 1.1 and TLS 1.0? Is that a point of concern?

A: We performed a cipherscan. The results of which can be found here:

<a href="https://avadacatavra.github.io/servo/2016/10/10/cipherscan.html">Cipher Scan results</a>

As we see, only 1% of the websites depend on TLS 1.1. 

5 % of the websites depend on TLS 1.0. Support for TLS 1.0 is a cause for concern for us. 

Cipher scans can be performed using this tool:

<a href="https://github.com/mozilla/cipherscan">Cipher Scan</a>

*In essence this meant removing features which we did not need in a browser today. Over the years, the OpenSSL TLS stack has developed with the browser. During this process, a lot of things have come into existence and a lot of things have become obsolete. Hence, it makes sense to clean what we did not need anymore.

