---
title: Introducing zq.lu
author: Noa Resare
date: "2026-05-31"
---
There are lots of times when public keys from asymmetric key pairs are copied, shared, put into configuration files and compared. However, the way in which we encode our public keys are at best inconvenient to the point where it is easy to get something wrong. Take for example the public key that I would copy to the `~/.ssh/authorized_keys` file on a remote server:

`ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMOmfeR5oaRJvme4/uFNitESVwdHwACESMqdTSxxIP+UytIBlUU+37/7qyCKkkRWFlvsRyjSQbfnLRE+UZTlH8Y=`

It is long, it contains a whitespace character and other characters that the copy functionality in my terminal considers a new word. If I get the copying wrong there is no checksum that makes it easy to determine that the key is invalid. Don't even get me started on the JWK format.

The `zq.lu` project attempts to address these problems by introducing a new way to represent public keys. A key looks like this: `zq.luDCkyil5qf1qCJwm5lnSe97x5mROvXUMfwpOASJ0MgalZTIj`

* The first 5 characters uniquely identifies the format, they are unlikely to appear at the beginning of other files and also happen to be a web address: `zq.lu` 
* it encodes the key in an efficient way
* it uses only letters, numbers and a single dot
* it is easy to recognise as a zq.lu by looking at it
* it contains an CRC16 checksum, making it easy to detect accidentally altered keys as invalid

If your software expects users to put public keys in configuration or anywhere else where it might need to be copied, you should consider supporting the `zq.lu` format. To the best of my knowledge, there is no other key format where a valid key is also a valid `zq.lu` key, which means that supporting it can be done in a way that is completely compatible with existing functionality.

See https://zq.lu for a specification and reference implementation.
