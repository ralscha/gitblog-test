---
published: 2019-01-01T10:10:59.167Z
summary: Happy new year
tags: [java, database]
title: A new year
updated: 2019-01-01T06:29:02.130Z
---

When writing an application where users can sign up and sign in
with passwords, it is recommended to follow [guidelines](https://pages.nist.gov/800-63-3/)
to check the password quality and prevent the users from choosing 
simple and easy-to-guess passwords. One recommendation is to check the passwords against a list of commonly used or compromised passwords and reject those.

In a [previous blog post](https://golb.hplar.ch/2018/05/Check-for-commonly-used-or-compromised-passwords.html) 
I showed you different ways how to do it.

One of the largest databases of compromised passwords is [*Pwned Passwords*](https://haveibeenpwned.com/Passwords)
from [*have i been pwned?*](https://haveibeenpwned.com/).

The service provides an [HTTP API](https://haveibeenpwned.com/API/v3#PwnedPasswords) that makes it very easy to check if a particular password
is in the *Pwned Passwords* database. The API is free to use, does not require an API key, and can be accessed from a browser
or a back end. This blog post focuses on the back end and shows you different ways to access this database with Go.

## HTTP API

The first and most obvious way to access the database is by sending an HTTP request to the
*Pwned Passwords* HTTP API. The standard library of Go contains an HTTP client so that we can implement this without any 3rd party library.

To check if a password exists in the database, you can't simply send
the password in plaintext to the service. That would not be very secure. So instead, the API implemented a [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity) model
that allows a password to be searched for by partial hash. 

Here is the workflow a client has to implement

1. Create SHA-1 hash of the password
2. Take the first five characters of the hash and send it to the *Pwned Passwords* HTTP API.
3. The response contains a list of all SHA-1 hashes that start with the same five characters.
4. The client has to iterate through the list and search if the hash from Step 1 is in the list
5. If it's in the list, the password exists in the HIBP password database.

This is an example of how the response looks.

```
0018A45C4D1DEF81644B54AB7F969B88D65:1
00D4F6E8FA6EECAD2A3AA415EEC418D38EC:2
011053FD0102E94D6AE2F8B83D76FAF94F6:1
012A7CA357541F0AC487871FEEC1891C49C:2
0136E006E24E7D152139815FB0FC6A50B15:2
```

The first five characters of the hash are not part of the response. The number after the colon (:) 
says how many times the password appeared in a breach.

To enhance security even further, it is possible to tell the service to add padding. 
Because each five-character prefix returns a different amount of hashes, a man-in-the-middle could determine what password the client asked for based on the response size. With padding, each response approximately has the same size.

If padding is enabled, the service adds random hashes at the end with an appearance count
of 0

```
FF849C5DE93593756DDF7AECBBE40B9A947:9
FF90FEE99C8E2961AD1077ADEE4C6FEFF30:1
FFA7B0AD1884D08B5AE2644C8D9D76BD4B5:1
85EB3DA4B91A01D5389998B53B24F9091B0:0   <--- padding records
7CC3E32849664C6E4FD387F94B7262D625C:0   <---
3DFAA395724C94E783EB77CB6C7BF429BC2:0   <---
```

To enable padding, send the request header `Add-Padding: true`.

Here is a Go program that checks if the password "123456" is in the *Pwned Passwords* database.

[github:https://github.com/ralscha/blog2022/blob/master/hibp-go/client/main.go#L17-L71]


For more information about the API, check out the [documentation](https://haveibeenpwned.com/API/v3#PwnedPasswords).


## Self-Host

Not everybody likes the idea of sending requests over the Internet 
to check for compromised passwords. This could be because of security concerns
or the concern of being dependent on a 3rd party service.

Fortunately, there is a way to self-host the database and access it locally.
*Pwned Passwords* is the only service from *have i been pwned?* that allows you to
download the complete dataset. 

The dataset is free and can be downloaded without an account.
If you want to follow along, go to the [donwload page](https://haveibeenpwned.com/Passwords) and
download the SHA-1 (ordered by hash) file. I downloaded version 8 with 847,223,402 passwords
Download the file if possible with torrent. If that does not work,
use the Cloudflare link. Then, unzip the file.

Each line of the text file contains an SHA-1 encoded password. 
After the hash follows a colon and the number of appearances in a breach.

```
000000005AD76BD555C1D6D771DE417A4B87E4B4:10
00000000A8DAE4228F821FB418F59826079BF368:4
00000000DD7F2A1C68A35673713783CA390C9E93:873
00000001E225B908BAC31C56DB04D892E47536E0:6
00000006BAB7FC3113AA73DE3589630FC08218E7:3
00000008C4037D3E893F8E1FA7BAD32B9F60948C:3
00000008CD1806EB7B9B46A8F87690B2AC16F617:5
```

To have fast access to the data, we need to import the file
into a database. A text search would take too long. 

You can import this data into any database. I will show you how to use an embedded database in Go for this blog post. A key/value store
fits the data structure perfectly, and the Go ecosystem offers a few options for this kind of database. I tested [bbolt](https://pkg.go.dev/go.etcd.io/bbolt), [goleveldb](https://github.com/syndtr/goleveldb), [BadgerDB](https://github.com/dgraph-io/badger) and [Pebble](https://github.com/cockroachdb/pebble).
I will show you the code with Pebble, which creates the smallest database
and is the fastest importer. If you are interested in the Go code of the
other implementations, check out [this repository](https://github.com/ralscha/blog2022/tree/master/hibp-go/others).

Pebble is the database engine underneath the [CoackroachDB](https://www.cockroachlabs.com/) database. 
It is a RocksDB/LevelDB inspired key-value database.

The importer reads the *Pwned Passwords* text file and
imports the data into the Pebble database. Pebble is a file-based 
database, so you must specify a directory when opening the database with `pebble.Open`.

[github:https://github.com/ralscha/blog2022/blob/master/hibp-go/pebble/importer/main.go#L16-L81]

Because Pebble allows arbitrary binary values as the key, the import program 
converts the SHA-1 hex string into the raw form. So instead of 40 bytes 
for the hex string, it only stores 20 bytes per key. In addition, for the appearance value, it uses [`binary.PutUvarint`](https://pkg.go.dev/encoding/binary#PutUvarint) to convert the number
into an unsigned variable length encoded number to save even further space. 

The import process takes a few minutes.

<br>

Next, we need a program that reads the data. The following application
is a simple command-line tool where you can pass the password as the argument.

[github:https://github.com/ralscha/blog2022/blob/master/hibp-go/pebble/read/main.go#L14-L41]

The program calculates the SHA-1 of the input and runs a key lookup (`db.Get`).



## Self-host HTTP API Server

The disadvantage of using an embedded database is that only one process
can access the data. So if multiple services need to read the database, you must copy the 
database. But instead of doing that, we can write our own *Pwned Passwords* HTTP API
server. Thanks to the comprehensive Go standard library, we can do that
without any additional 3rd party library. 

You find [my implementation on GitHub](https://github.com/ralscha/blog2022/blob/master/hibp-go/api_server/main.go). 

The server opens the database we created in the previous section and then waits for incoming requests.
It uses Pebble's prefix key scan feature to find all keys that start with a prefix. It also checks for the `add-padding` request header and adds random hashes to the response. 

I tried to copy all the features of the original service. If you are interested, you find the actual *Pwned Passwords* service [here](https://github.com/HaveIBeenPwned/PwnedPasswordsAzureFunction).
It runs as an Azure Function.

Because my implementation behaves the same as the original, it is possible to take any HIBP client and point it to the Go service. For example, take the first example from this blog post, change the URL and run it. It still works.


## Bloom

The downside of the previous solutions is the huge database size on disk. 
It needs about 20GB of storage for the database.

If that is a problem, you could look at a different data structure. [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) is an interesting solution. A Bloom filter is a data structure
designed to tell, in a space-efficient manner, if an element is present in a set.
To achieve space efficiency, a Bloom filter is a probabilistic data structure.
It tells you that an element is definitely not in the set or possibly in
the set. That means it reports false positives. 

When we store the *Pwned Passwords* file in a Bloom filter, we can check if a password is definitely not in the set. But the Bloom filter might tell us that a password is in the set where in reality it's not (false positive).

The probability of the false positives is configurable and determines the size of the Bloom filter in memory. The smaller the false positive rate, the larger the memory usage.

The following application uses [this Bloom library](https://github.com/bits-and-blooms/bloom).      
Also, check out the [Awesome Go list](https://github.com/avelino/awesome-go#data-structures) 
for other Bloom filter implementations.

Reading the whole *Pwned Passwords* text file into the Bloom filter takes a long time, and it would not be feasible
to do that every time the application starts. So instead, an import program imports the data once and stores the binary representation of the Bloom filter as a file.

[github:https://github.com/ralscha/blog2022/blob/master/hibp-go/bloom/importer/main.go#L14-L72]

A reading program reads the serialized Bloom filter from disk into memory and then looks up the password. 

[github:https://github.com/ralscha/blog2022/blob/master/hibp-go/bloom/read/main.go#L12-L42]

Here is an overview of how the false positive rate influences the size of the Bloom filter.

| false-positive rate  | size (bytes)  |
|--:|--:|
| 10 %  | 507,542,888  |
| 1 %  | 1,015,085,752  |
| 0.1 %  | 1,522,628,608  |
| 0.01 % | 2,030,171,472  |
| 0.001 %  | 2,537,714,328  |
| 0.0001  % | 3,045,257,192  |
| 0.00001 %  | 3,552,800,048  |
| 0.000001 %  | 4,060,342,912  |

 
<br>

That concludes this tutorial about accessing the *Pwned Passwords* database with Go. You have
seen that it is pretty easy to self-host the database and access it without sending requests over the Internet and dependent on a 3rd party service.

