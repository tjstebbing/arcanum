Arcanum - Authentication as a service.
======================================

Arcanum provides a key-value store for password hashes with a simple API for
verifying passwords with built-in backoff and a variety of APIs for setting 
new passwords. 

Arcanum removes passwords from your regular database and is intended to be 
run in isolation from your primaty data store (On a different machine). A
unique secret key is then used by your application to authenticate users 
against the Arcanum service. 

This approach means that your password hashes are useless without both 
databases, ideally making it vastly more difficult\* for a cracker to
exploit your customer's passwords should you be compromized.

\* _YMMV based on your security practices!_

Current practices are a problem
-------------------------------

More and more frequently we hear reports of large, popular web services
losing millions of customer passwords along with personal user data as 
crackers gain access to servers.  Given the propensity for humans to use
the same password for many services, and the impossibility of providing 
a 100% secure, un-crackable system, it behoves us as architects of these
products to do our utmost to protect our customers against exploits of 
all kinds. 

It is common practice to store password hashes with user data, I'm sure
this pattern is familiar to you:

*Relational*:

```
                                     Table "users"
    Column    |           Type           |          Modifiers                        
--------------+--------------------------+---------------------------
 id           | integer                  | not null default nex..
 username     | character varying(30)    | not null
 password     | character varying(128)   | not null
 ...  other personal data

```

*NoSQL*:

```javascript 
{ "username" : "pomke", 
  "password" : "EFemjntKiIj3apL9nw==$w80kGMRmG47XNMKgrr6igupFrCYOQs6Nto9bsA==",
  ... other personal data
}
```

It is a convenient and obvious way to store data, however it also
means passwords are conveniently stored with identifiable user information
which when stolen and brute-forced provides crackers with a fantastically 
abusable resource.


Seperation of concerns
----------------------

UNIX systems have understood the need for a seperation of passwords from 
user data since the mid 80s, maintaining password hashes in the [shadow password
file](http://en.wikipedia.org/wiki/Shadow_password), seperate from the passwd
file containing other user information. 

In this same way Arcanum fills the role of the shadow password file for 
internet-facing systems, albeit with a slightly different set of requirements.

A system using Arcanum generates a unique secret key for a user which it uses 
to key the password, this should not be a value ever exposed to users, it 
shouldn't be the auto-incremented ID in your user table, and most definitely 
not the username or email address (UUIDs are a great idea). 

*Relational*:

```
                                     Table "users"
    Column    |           Type           |          Modifiers                        
--------------+--------------------------+---------------------------
 id           | integer                  | not null default nex..
 username     | character varying(30)    | not null
 arc_key      | character varying(32)    | not null
 ... other personal data

```

*NoSQL*:

```javascript 
{ "username" : "pomke", 
  "arc_key" : "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  ... other personal data
}
```

Your application can then communicate directly with Arcanum via HTTP
using this key to store and verify passwords for user-requests. 


API
===

Arcanum is accessible via HTTP/REST and understands JSON. Communication
is relatively simple and a variety of actions are available:


Setting passwords:
------------------

Setting passwords is as simple as posting to a password setting URL. These urls
all function in basically two forms, with or without the validation of providing
an old password:

*POST /\<key\>/\<new password\>* 

Sets a new password for the given key, without any validation checking or 
hashing, what you pass is what gets stored.

Returns json: 

* "[true]" 

```sh
curl --d "" http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8/EFemjntKiIj3apL9nw%3d%3d%24w80kGMRmG47XNMKgrr6igupFrCYOQs6Nto9bsA%3d%3d
```

*POST /\<key\>/\<old password\>/\<new password\>* 

Sets a new password, however validates that the old password matches. 

Returns json:

* "[true]" if the password was set.
* "[false, 'NOUSER']" if no user matches the key.
* "[false, 'MISMATCH']" if old password does not match.
* "[false, 'BACKOFF', \<seconds\>]" if a backoff threshold applies.

```sh
curl --d "" http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8/EFemjntKiIj3apL9nw%3d%3d%24w80kGMRmG47XNMKgrr6igupFrCYOQs6Nto9bsA%3d%3d/jW4t9FJn4FyYYpMtbw%3d%3d%2frvZyV%2bgXg6ZBU2bUtbN9K18e5nPjg%3d%3d
```

Checking passwords:
-------------------

*GET /\<key\>/\<password\>* 

Check if the password matches the key.

Returns json:

* "[true]" if the password was set.
* "[false, 'MISMATCH']" if password does not match.
* "[false, 'BACKOFF', \<seconds\>]" if a backoff threshold applies.

```sh
curl http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8/EFemjntKiIj3apL9nw%3d%3d%24w80kGMRmG47XNMKgrr6igupFrCYOQs6Nto9bsA%3d%3d
```

Hashing:
--------

Arcanum can handle hashing for you, simply prepend the hash function to the path
of any URL and provide plaina-text values for passwords in the URL. Each hash 
function uses a salt which is configurable in the Arcanum config file. 

*GET /SHA256/\<key\>/\<password\>* 

This would test if the password matches, after applying the SHA256 hash function,
this assumes the password was first stored using the same hash method.

```sh
curl http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8/correct+horse+battery+staple
```

*POST /HMAC/\<key\>/\<old password\>/\<new password\>* 

This would update the password if old password matches, using the HMAC hash
function, this assumes the password was first stored using the same hash method.

```sh
curl -d "" http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8/butterfly27/correct+horse+battery+staple
```


Configuration:
==============

TODO


