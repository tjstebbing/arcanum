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

![Chromie](https://github.com/pomke/arcanum/raw/master/diagram.png)

API
===

Arcanum is accessible via HTTP/REST and understands JSON. Communication
is relatively simple and a variety of actions are available:


Setting passwords:
------------------

Setting passwords is as simple as posting some JSON to a URL.

*POST /\<key\>* 

```javascript
//POST to http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8
{ "newpass" :  "w80kGMRmG47XNMKgrr6igupFrCYOQs6Nto9bsA==" }
```
Sets a new password for the given key, without any validation checking or 
hashing, what you pass is what gets stored.

Returns json: 
* "[true]" 


```javascript
//POST to http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8
{ "newpass" :  "w80kGMRmG47XNMKgrr6igupFrCYOQs6Nto9bsA==",
  "oldpass" : "/rvZyV+gXg6ZBU2bUtbN9K18e5nPjg==" }
```

Sets a new password, however validates that the old password matches. 

Returns json:

* "[true]" if the password was set.
* "[false, 'NOUSER']" if no user matches the key.
* "[false, 'MISMATCH']" if old password does not match.
* "[false, 'BACKOFF', \<seconds\>]" if a backoff threshold applies.


Checking passwords:
-------------------

*POST /\<key\>* 

```javascript
//POST to http://arcanum.local/6ba7b810-9dad-11d1-80b4-00c04fd430c8
{ "check" :  "w80kGMRmG47XNMKgrr6igupFrCYOQs6Nto9bsA==" }
```

Check if the password matches the key.

Returns json:

* "[true]" if the password was set.
* "[false, 'MISMATCH']" if password does not match.
* "[false, 'BACKOFF', \<seconds\>]" if a backoff threshold applies.


Hashing:
--------

Arcanum can handle hashing for you, simply prepend the hash scheme 
(see config.json) to the path of any URL and provide plaina-text values 
for passwords. New hash schemes are configurable in the config file, the 
standard scheme '/std/' is pretty good.

*GET /std/\<key\>* 

```javascript
//POST to http://arcanum.local/std/6ba7b810-9dad-11d1-80b4-00c04fd430c8
{ "check" :  "correct horse battery staple" }
```

This would test if the password matches, after applying the std HMAC/SHA256 hash 
function, this assumes the password was first stored using the same hash method.


*POST /alt1/\<key\>* 

```javascript
//POST to http://arcanum.local/alt1/6ba7b810-9dad-11d1-80b4-00c04fd430c8
{ "newpass" :  "correct horse battery staple",
  "oldpass" : "butterflies27" }
```

This would update the password if old password matches, using the alt1 hash
function, this assumes the password was first stored using the same hash method.


Hash schemes should be given non standard names, rather than 'sha256' to avoid
revealing too much information should the requests be logged somewhere in your 
infrastructure. 


Configuration:
==============

Arcanum has a JSON config file, config.json with the following params:

* _port_ The port to listen on.
* _backend > module_ Is a node.js module path, by default we use riak.
* _backend > \*_ All other arguments are passed to this module.
* _backoff > enabled_ Should Arcanum manage expomential backoff, that is refuse repeat attempts. 
* _backoff > min_attempt_threshold_ how many attempts to allow before applying backoff. 
* _backoff > max_time_seconds_ max time backoff will block a user in seconds. 
* _hashes > scheme_id_ provides a scheme for hashing passwords.
* _hashes > scheme_id  > module_ a node.js module path which provides a hash function.
* _hashes > scheme_id  > \*_ all other arguments are passed to this module.



