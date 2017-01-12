# Web Share Target API Interface

**Date**: 2017-01-12

This document is a rough spec (i.e., *not* a formal web standard draft) of the
Web Share Target API. This API allows websites to register to receive shared
content from either the [Web Share API](https://github.com/mgiuca/web-share), or
system events (e.g., shares from native apps).

This API has 2 proposed designs: both require the user agent to support [web
app manifests](https://www.w3.org/TR/appmanifest/), while the second requires
support for [service workers](https://www.w3.org/TR/service-workers/). We are
planning to explore both approaches to determine which is more acceptable. The
[Web Share API](https://github.com/mgiuca/web-share) is not required, but
recommended.

Examples of using the Share Target API for sharing can be seen in the
[explainer document](explainer.md).

**Note**: The Web Share Target API is a proposal of the [Ballista
project](https://github.com/chromium/ballista), which aims to explore
website-to-website and website-to-native interoperability.

## App manifest

The first thing a handler needs to do is declare its share handling capabilities
in its [web app manifest](https://www.w3.org/TR/appmanifest/):

### Approach 1:

We expect apps that are sharing data to share one or more fields, including
title, text, and URL. These fields can be passed to the target app as query
parameters, and each is optional. If a web app can handle a share, they should
include (in the manifest) a template URL (relative to the domain) that the
shared data can be inserted into.

```WebIDL
partial dictionary Manifest {
  string share_url_template;
};
```

The share_url_template member will contain placeholders for each field of the
form "{field}". Each placeholder with be replaced with the value of the
corresponding field, that has been shared by the source app. If a given field
was not shared, it's placeholder will be replaced with an empty string. An
example url template is here:

```json
"share_url_template": "/share?title={title}&text={text}&url={url}"
```

The `share_url_template` member also allows the target web app to specify which
attributes of the shared data it cares about. e.g. for Web Share, the passed
data includes title, text, and URL, but the receiving web app may only care
about message and URL.

### Approach 2:

```WebIDL
partial dictionary Manifest {
  boolean supports_share;
};
```
The `"supports_share"` member of the manifest, if `true`, indicates that the app
can receive share events from requesters, or the system.

### Things to note

The declarative nature of the manifest allows search services to index and
present web applications that handle shares.

Handlers declaring `supports_share` or `share_url_template` in their manifest
will **not** be automatically registered; the user must explicitly authorize
the registration. How this takes place is still under consideration (see [User
Flow](explainer.md#user-flow), but will ultimately be at the discretion of the
user agent (the user may be automatically prompted, or may have to explicitly
request registration).

**For consideration**: We may wish to provide a method for websites to
explicitly request to prompt the user for handler registration. There would
still be a requirement to declare `supports_share` in the manifest. For now,
we have omitted such a method from the design to keep control in the hands of
user agents. It is easier to add such a method later than remove it.

## Handling incoming shares

### Approach 1

Recall the URL template from "Approach 1":

`
/share?title={title}&text={text}&url={url}
`

This will be filled with the share data, and opened by the browser, when the
user selects the target app.

For example, if a source app shares the data:

```JSON
{
  "title": "Example",
  "text": "I'm an example",
  "url": "https://www.example.com"
}
```

The browser will then launch the picker UI, and the user picks some target
app. If the target app is www.example.com, the browser will launch the
following URL in a new window or tab:

`
https://www.example.com/share?title=Google&text=Search%20the%20web&url=https://www.google.com
`

Thus, the receiving web app should handle the shared data as desired, at that
URL.

### Approach 2

Handlers **must** have a registered [service
worker](https://www.w3.org/TR/service-workers/).

When the user picks a registered web app as the target of a share, the
handler's service worker starts up (if it is not already running), and a
`"share"` event is fired at the global object.

```WebIDL
partial interface WorkerGlobalScope {
  attribute EventHandler onshare;
};

interface ShareEvent : ExtendableEvent {
  readonly attribute ShareData data;

  void reject(DOMException error);
};

dictionary ShareData {
  DOMString? title;
  DOMString? text;
  DOMString? url;
};
```

The `onshare` handler (with corresponding event type `"share"`) takes a
`ShareEvent`. The `data` field provides data from the sending application.

How the handler deals with the data object is at the handler's discretion, and
will generally depend on the type of app. Here are some suggestions:

* An email client might draft a new email, using `title` as the subject of an
  email, with `text` and `url` concatenated together as the body.
* A social networking app might draft a new post, ignoring `title`, using `text`
  as the body of the message and adding `url` as a link. If `text` is missing,
  it might use `url` in the body as well. If `url` is missing, it might scan
  `text` looking for a URL and add that as a link.
* A text messaging app might draft a new message, ignoring `title` and using
  `text` and `url` concatenated together. It might truncate the text or replace
  `url` with a short link to fit into the message size.

A share fails if:

* The handler had no registered service worker.
* There was an error during service worker initialization.
* There is no event handler for `share` events.
* The event handler explicitly calls the event's `reject` method (either in the
  event handler, or in the promise passed to the event's
  [`waitUntil`](https://www.w3.org/TR/service-workers/#wait-until-method)
  method).
* The promise passed to `waitUntil` is rejected.

Once the event completes without failing, the share automatically succeeds, and
the requester's promise is resolved. The end of the event's lifetime marks the
end of the share, and there is no further communication in either direction.

The Share Target API is defined entirely within the service worker. If the
handler needs to provide UI (which should be the common case), the service
worker must create a foreground page and send the appropriate data between the
worker and foreground page, out of band. The `share` event handler is [allowed
to show a
popup](https://html.spec.whatwg.org/multipage/browsers.html#allowed-to-show-a-popup),
which means it can call the
[`clients.openWindow`](https://www.w3.org/TR/service-workers/#clients-openwindow-method)
method.

## Where do shares come from?

Share events can be sent from a variety of places:

* Built-in trigger (e.g., user picks "Share" from a browser's menu, to share the
  URL in the address bar).
* A native application.
* A web application using the [Web Share
  API](https://github.com/mgiuca/web-share).

There will usually be a picker that lets the user select a target app. This
could be the native system app picker, or a user-agent-supplied picker. The apps
could include other system apps and actions alongside the web app handlers.

If an event comes from a web app, the `data` field of the event should be a
clone of the `data` parameter to `navigator.share`. If the event comes from some
other source, the user agent may construct the `data` object as appropriate.
