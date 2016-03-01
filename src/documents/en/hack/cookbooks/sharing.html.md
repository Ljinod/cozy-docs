---
title: "Sharing API"
layout: "default"
category: "hack"
menuOrder: 42
toc: true
---

# Cozy-Sharing

*DISCLAIMER*
As of when this documentation is being written we still are developping a way
to enable the share of documents between Cozy. That means this
documentation could be slightly out of date but we will update it as fast as
possible.

Yes, you have read me well: you very well might be able to share -- in a
secure, private and reliable fashion -- with other proud Cozy owners!

<br />
<br />

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

<br />
<br />


## API

Now that we are all set let's dive into it! This first part tries to keep
things simple: in the following paragraphs we will explain why we implemented
this, what we need to make it work and then what code an application has to add
to enable this feature.


### Basics

The idea is pretty straight forward: Alice would like to share something from
her Cozy with Bob (who also happens to own a Cozy -- he is such a good friend)
but doesn't want to do it by making it publicly accessible.  A perfect solution
would be to be able to share it from one Cozy to another: ladies and gentlemen,
this is what we have done!


We have two prerequisites for this to work:

1. The **url** of the recipient's Cozy: if Alice wants to share with Bob she
   needs to know Bob's url. Just like if she wants to send a postcard to Bob
   she needs Bob's address.<br />
   *On a side note we, Cozy Cloud, will not provide the url of someone's Cozy:
   privacy is our core feature so there is no way for us to divulge such
   information.*

2. An actual **file** to share. Even if, on the philosophical side of things,
   the act of sharing suffices itself and does not require data to be
   meaningful we think it would be a waste of resources -- amongst all -- to
   allow it.


### [The Coding Love](http://thecodinglove.com)

Okay so you're Alice and you have a document you want to share but your
application doesn't permit such operation yet. Let's change that!

The structure on which the protocol relies is what we have called "*sharing*"
(we are no poets). This structure will be sent to the recipient and requires
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

<br />

> *Alice*: And what are exactly those fields?<br />
> *Cozy*: We've got you covered, the explanations are just below. ;-)

<br />

The fields in the sharing structure are:

* **desc**: the description of what is shared as defined by the sharer or her
  application.
* **targets**: an array containing all the url of the recipients. There may be
  more than one recipient.
* **rules**: a set of rules matching the documents shared. A rule is composed
  of the id of the document and its docType. You can have as many rules as you
  like, one for each document shared.<br />
  Even though the docType is not needed to precisely identify a file, it is
  however useful for security measures.
* **sync**: a boolean telling wether or not the share is synchronous (if a
  change was made on the document by the sharer, will it be propagated to the
  recipients?).

And that is all you need to send a share request to your recipient(s)!

> *Alice*: That's almost too easy!<br />
> *Cozy*: * blush *

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

<br />

Here you go! You shared your document(s) to your recipient(s). Your Cozy will
handle the rest of the operations and will display a notification when your
request will be accepted or -- we sure don't hope so -- denied. :-)

<br />
<br />

## The Protocol

Ha! You have seen that there is more text to read. But what could we possibly
be explaining down here? Well, what is interesting but "technical": the
underlying protocol!

In the following paragraphs we will try to explain as clearly as possible what
manipulations are done on what and by who.<br />
To do so we will first detail the complete structure of a sharing document,
then we will see XXX

### The complete sharing document

The document created in the previous paragraph is just a "sample" of the
complete document ; indeed take a look at its final form:

```javascript
var sharing = {
    id: "1aqwzsxed",
    docType: "sharing",
    desc: "The description of the document",
    rules: [
        { id: "2zsxedcrf", docType: "picture" },
        { id: "3edcrfvtg", docType: "event" }
    ],
    targets: [
        { url: "bob.hiscozycloud.com",
          pwd: "1poiuytreza",
          replID: "4rfvtgbyh" },
        { url: "charles.cozy.hk",
          pwd: "2mlkjhgfdsq",
          replID: "5rfvtgbyh" }
    ],
    sync: "true"
};
```

Let us describe the new fields:

* **id**: this is the id generated by CouchDB when the sharing document is
  created in the database.
* **docType**: the document type. Concretly it is used to manage permissions
  and as a filter when we want to retrieve documents.
* **pwd**: a password linked to a single share and to a unique recipient. It
  is used to authenticate the sharer.
* **replID**: the replication id generated by CouchDB. To share data we use the
  replication feature provided by CouchDB. We will tell more on this later.


> *Alice*: Then, could you explain how we end up with this structure?
> *Cozy*: It will be my pleasure!


### Raiders of the lost fields

If you recall correctly the structure the application sends to the data-system
resembles this:

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

When we receive such request we immediatly store the document in the database
for further use, and in order to do so we have to add a **docType**. This is
mandatory if we want to manage our document and the applications that can
access it.<br />
With this step CouchDB generates the **id** and the **docType** was
automatically added by Cozy-sharing: we got rid of the first two items on our
list!

> *Alice*: Is the id field useful besides being an identifier?
> *Cozy*: You're right, it serves another purpose that we will explain in an
> upcoming paragraph: access control.

With those two fields set we can then send an actual request to the targets.
Each target will receive a different request hence no information is shared
between targets.

Once the target's Cozy has receive the request, it will also create a sharing
document copying all information received in the request except for the id,
CouchDB will take care of this.

> *Bob*: But you're creating a document on my Cozy even if, in the end, I
> decline?
> *Cozy*: Hi Bob! Well yes, if we don't store it and only keep it in RAM then
> if you decide to reboot your Cozy to install a new version of the platform,
> you would loose the request in the process.
> *Cozy*: If it's the idea of storing that troubles you imagine this a
> receiving an e-mail. :-)

The document is generated, Cozy waits for the approbation of the owner. Here we
have two possible scenarios: the blue one where the recipient accepts the
request and the red one where the recipient rejects it.

Let us first rapidly discuss the latter, which is the easiest. If the request
is rejected then both the sharer and the recipient deletes their respective
sharing document (or only the corresponding "target" object in the sharer's
document if there are multiple targets) and that's it. The sharer is notified
and can do nothing more.

The second scenario is a bit more complicated but not too much for you to
understand, we did it, so can you!

> *Cozy*: Welcome to the real world!
> *Bob* and *Alice*: What?
> *Cozy*: No, nothing... It was just a glitch.
