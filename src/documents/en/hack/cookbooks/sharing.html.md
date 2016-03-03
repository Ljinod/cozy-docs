---
title: "Cozy-sharing"
layout: "default"
category: "hack"
menuOrder: 42
toc: true
---

# Cozy-Sharing

*DISCLAIMER*<br />
As of when this documentation is being written we still are developping a way
to enable the share of documents between Cozy. That means this
documentation could be slightly out of date but we will update it as fast as
possible.

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

2. An actual **file** to share. Even if, on the philosophical side of things,
   the act of sharing suffices itself and does not require data to be
   meaningful we think it would be a waste of resources -- amongst all -- to
   allow it.


### The Coding [Love](http://thecodinglove.com)

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

> *Alice*: And what are exactly those fields?<br />
> *Cozy*: We've got you covered, the explanations are just below. ;-)


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
* **sync**: a boolean telling wether or not the share is synchronous. If it is
  set to true then every change the sharer makes on the document is propagated
  to the recipients.

> *Alice*: That's it?<br />
> *Cozy*: Yup, that's all the information you need to send. :-)

Now that you have everything it is high time to share. Here is how you could
proceed:

```javascript
var Client = require('request-json').JsonClient;

// Connect to the data-system (of the sharer)
var client = new Client("http://localhost:9101");

// Add the credentials of your application
client.setBasicAuth(process.env.NAME, process.env.TOKEN);

// Post your sharing request using your sharing structure defined above
client.post("sharing/", sharing, function(err, res, body) {
    if(err) {
        // handle error
    } else {
        // show must go on!
    }
});
```

> *Alice*: Credentials?<br />
> *Cozy*: Indeed, we don't want any application to be able to share your files.
> Your application needs to ask for the **"sharing" permission** to be able to
> share.

If you don't know how to declare the permissions your application requires,
please take a look at the corresponding documentation: [how to add
permissions](https://docs.cozy.io/en/hack/getting-started/play-with-data-system.html).
As we said in the dialogue above, to be able to share an application needs the
permission called **sharing**.

<br />

Here you go! You shared your document(s) to your recipient(s). Your Cozy will
handle the rest of the operations and will display a notification when your
request will be accepted or -- we sure don't hope so -- denied. :-)

<br />
<br />

## The Protocol

In the following paragraphs we will try to explain as clearly as possible what
manipulations are done, on the server side of things, on what and by who. In
short if you are not afraid to get technical, read on!

To do so we will first detail the complete structure of a sharing document,
then we will see how this document is completed, after that we will explain the
ways of access control, and finally introduce the replication which is at the
root of the sharing process.


### The complete sharing document

The document created in the previous paragraph is just a "sample" of the
complete document:

```javascript
var sharing = {
    id: "1aqwzsxed",
    docType: "sharing",
    desc: "A description of the documents",
    rules: [
        { id: "2zsxedcrf", docType: "picture" },
        { id: "3edcrfvtg", docType: "event" }
    ],
    targets: [
        { url: "bob.hiscozycloud.com", pwd: "password1", repID: "4rfvtgbyh" },
        { url: "charles.cozy.hk", pwd: "password2", repID: "5rfvtgbyh" }
    ],
    sync: "true"
};
```

The new fields are:

* **id**: this is the id generated by CouchDB when the sharing document is
  created in the database.
* **docType**: the document type. Concretly it is used to manage permissions
  and as a filter when we want to retrieve documents.
* **pwd**: a password linked to a single share and to a unique recipient. It
  is used to authenticate the sharer and it is generated by the recipient.
* **replID**: the replication id generated by CouchDB. To share data we use the
  replication feature provided by CouchDB. We will tell more on this later.


> *Alice*: Could you explain how we end up with this structure?<br />
> *Cozy*: It will be my pleasure!


### Raiders of the lost fields

If you recall correctly the structure the application sends to the data-system
resembles this:

```javascript
var sharing = {
    desc: "A description of the documents",
    targets: [
        { url: "bob.hiscozycloud.com" },
        { url: "charles.cozy.hk" }
    ],
    rules: [
        { id: "2zsxedcrf", docType: "picture" },
        { id: "3edcrfvtg", docType: "event" }
    ],
    sync: "true"
};
```

When we receive such request we immediatly store the document in the database
for further use, and in order to do so we have to add a **docType**. This is
mandatory if we want to manage our document and the applications that can
access it.<br />
To sum up: CouchDB generates the **id** and the data-system automatically adds
the **docType**. We found the first two items on our list!

```javascript
    id: "1aqwzsxed",    // generated by CouchDB
    docType: "sharing", // automatically added by Cozy-sharing
```

> *Alice*: Is the id field useful besides being an identifier?<br />
> *Cozy*: You're right, it serves another purpose that we will explain in an
> upcoming paragraph: access control.

With those two fields set we can finally send an actual request to the targets.
A request looks like this:

```javascript
var request = {
    id: "1aqwzsxed",
    desc: "A description of the documents",
    rules: [
        { id: "2zsxedcrf", docType: "picture" },
        { id: "3edcrfvtg", docType: "event" }
    ],
    sync: "true",
    url: "alice.cozycloud.com"
};
```

As you can see we create the request by extracting only the relevant
information from the sharing document. We also do not communicate the list of
recipients, so you don't have to worry about having the url of your Cozy in the
open.<br />
We add the url of the sharer so that the recipient can reply -- if you happen
to use an onion router then we would not be able to tell from whom the request
came.

<br />

Once the recipient's Cozy receives the request, it also creates a sharing
document copying all the information received. The id is also kept but it will
be used as a login. Again we let CouchDB generate the id of newly created
documents to avoid any collision.

> *Bob*: If I understand correctly  you're creating a document on my Cozy even
> if, in the end, I decline?<br />
> *Cozy*: If it's the idea of storing that troubles you, imagine this a
> receiving a new e-mail. :-)

The document is generated, Cozy then waits for the approbation of its owner.
Here we have two possible scenarios: the blue one where the recipient accepts
the request and the red one where the recipient rejects it.

Let us first rapidly discuss the latter, which is the easiest. If the request
is rejected then both the sharer and the recipient delete their respective
sharing document (or only the corresponding "target" object in the sharer's
document if there are multiple targets) and that's it. The sharer is notified
and can do nothing more.

The second scenario is a bit more interesting: in order to do things properly
we have to operate a certain way.


### Accepting a request: the ways of access control

As you may have guessed if Alice shares some document with you it means she
will have an access on your Cozy. Thus it is important that with this access
she can only do what she is supposed to and nothing more.

To acheive that we first generate a unique password. This password, combined
with the id the sharer gave us, will serve as a login/password to authenticate
the transaction on the recipient's Cozy. To memorize and later on check those,
an *Access* document is created:

```javascript
var Access = {
    login: "1aqwzsxed", // the id of the sharing document of the sharer
    token: "AVeryLongAndComplicatedPassword",
    app: "7ujnik,ol", // the id of the sharing document of the recipient
    rules: [
        { id: "2zsxedcrf", docType: "picture" },
        { id: "3edcrfvtg", docType: "event" }
    ]
};
```

You don't have to know in details the meaning of this "Access" document, what
is important to note here is that we do limit the permissions to just the
shared documents. Nothing more!

Now that the access control is in place the recipient's Cozy can reply to the
sharer's who can initiate the *replication*.


### Sharing is...replication, and caring

It is not always a good solution to reinvent the wheel when something already
exists and does the job in a sufficient way. That is with that in mind that we
decided to rely on CouchDB to replicate data from one CouchDB instance to
another.


