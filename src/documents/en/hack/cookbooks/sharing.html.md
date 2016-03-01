---
title: "Sharing API"
layout: "default"
category: "hack"
menuOrder: 42
toc: true
---

# Sharing API

As of when this documentation is being written we still are developping a way
to enable the share of documents between Cozy Clouds. That means this
documentation could be slightly out of date but we will update it as fast as
possible.

Yes, you have read me well: you very well might be able to share -- in a
secure, private and reliable fashion -- with other proud Cozy owners!


## Nomenclature

So we all *share* -- pun intended -- the same vocabulary from the beginning let
us define it before we go into details:

* **sharer**: is the user that wants to share a document with his friend
the...

* **recipient**: is the user that will receive the shared document. If you are
still following you should have guessed that he is the sharer's friend.

* **docType**: is the type of a given document. Cozy works in such a way that
every application can manage one or several type of documents. The document
types an application can manage are set through *permissions* and that is for
that same purpose that we will use it.



## API

Now that we are all set let's dive into the technical details! To keep things
simple, after we have explained the basics, we will divide the
explanations into two parts: the first will describe the protocole from the
point of view of an application -- in other words what you have to code in
your application to be able to share -- and the second will explain the entire
process -- in other words what's happening behind the curtains.


### Basics

#### The scenario

The idea is pretty straight forward: Alice would like to share something from
her Cozy with Bob (who also happens to own a Cozy -- he is such a good friend)
but doesn't want to do it by making it publicly accessible.  A perfect solution
would be to be able to share it from one Cozy to another: ladies and gentlemen,
this is what we have done and just for Alice!


#### The Prerequisites

We have two prerequisistes for this to work:

1. The **url** of the recipient's Cozy: if Alice wants to share with Bob she
   needs to know Bob's url. Just like if she wants to send a postcard to Bob
   she needs Bob's address.<br />
   *On a side note we, Cozy Cloud, will not provide the url of someone's Cozy:
   privacy is our core feature so there is no way for us to divulge such
   information.*

2. An actual **file** to share. Even if, on the philosophical side of things,
   the act of sharing suffices itself and does not require data to be
   meaningful we think it would be a waste of resources -- amongst all -- to
   allow it. Hence hipsters you are warned: if you want to share, share
   something!


### The Protocole

#### The point of view of an application

Okay so you're Alice and you have a document you want to share but your
application doesn't permit such operation yet. Let's change that!

The structure on which the protocole relies is what we have called "*sharing*"
(we are no poets). This structure will be send to the recipient and requires
the following information:

```javascript
var sharing = {
    desc: "Hey Bob this is my picture from my last holidays!",
    targets: [
        { url: "bob.hiscozycloud.com" }
    ],
    rules: [
        { id: "2zsxedcrf", docType: "picture" }
    ],
    sync: "false"
};
```


> *Alice*: And what are exactly those fields?<br />
> *Cozy*: We've got you covered, the explanations are just below. ;-)


The fields in the sharing structure are:

* **desc**: the description of what is shared as defined by the sharer.
* **targets**: an array containing all the url of the recipients. There may be
  more than one recipient.
* **rules**: a set of rules matching the documents shared. A rule is composed
  of the id of the document and its docType. You can have as many rules as you
  like, one for each document shared.<br />
  Even though the docType is not needed to precisely identify a file, it is
  however useful for security measures.
* **sync**: a boolean telling wether or not the share is synchronous (if a
  change was made on the document by the sharer will it be propagated to the
  recipients?).

And that is all you need to send a share request to your recipient(s)!


> *Alice*: That's almost too easy!<br />
> *Cozy*: That's what we are here for: making your life easier!


Now that you have everything it is high time to share. Here is how you could
proceed:

```javascript
var Client = require('request-json').JsonClient;

// Connect to the data-system (of the sharer)
var client = new Client("http://localhost:9101");

// Post your sharing request using your sharing structure defined above
client.post("sharing/", sharing, function(err, res, body) {
    if(err) {
        // handle error
    } else {
        // show must go on!
    }
});
```


Here you go! You shared your document(s) to your recipient(s). You Cozy will
handle the rest of the operations and will display a notification when your
request will have been accepted or -- we sure don't hope so -- denied. :-)
