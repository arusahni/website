<!--
    .. title: Migrating from Gmail to Fastmail
    .. slug: gmail-to-fastmail
    .. date: 2020-08-02 17:47:35 UTC-04:00
    .. tags: tech
    .. description: In which I describe my migration from Gmail to Fastmail as an email and calendar provider.
-->

In 2004, Google launched Gmail. This service changed everything. You didn't
have to worry about running over your few megabytes of quota - your storage
space was "unlimited" (with a ticker and everything)! Deletion was a thing of
the past, you now archive! Folders were so ninety-ninety-late, there
were labels! I got an invite within the first two weeks of it launching, and it
was good.

A decade or so later, Google launched Inbox, which brought innovations like
bundles, snoozing, highlights, pinning, sweeping, and smart filtering to the
deluge of email that flooded your account each day. I switched from Gmail to
Inbox, and it was better.

Then, in true Google fashion, it was sacrificed at the altar of project
mismanagement (or whatever the lack of product strategy is called). And it
was bad.

Since being forced back to Gmail, I've constantly lamented the death of a
service that made dealing with email less painful for me. Gmail is not only
without new innovation, but it's also slow; It regularly fails to load new
messages, or seemingly loses track of what it should be showing, necessitating
a hard refresh. Sure, Google has thrown a few bones at it, like smart replies,
but I respond to so few emails that spending a few seconds to formulate a
response has never been an issue.

Given these concerns, I realized that the "stickiness" of Gmail was gone.
I have the means to pay for service, and nothing is keeping me on Gmail
(other than the fact that everyone's been using my Gmail address for 16 years).
Leaving Gmail sounded doable, and I owned a personal domain on that I'd love to
use for email. The only question was: where do I go for hosting?

<!-- TEASER_END -->>

## Assessing the Field

At the time of my research, there were a handful of buzzy, well-regarded
options, both free and paid.

* **Gsuite** would do nothing other than allow me better custom domain integration.
  It doesn't help that Gsuite accounts are second class citizens in the Google
  ecosystem.
* **Outlook Premium** is Microsoft's offering. However, I'm wary of Microsoft
  pushing Exchange over more open protocols, so an instant no-go.
* **Self hosting** is always an option, but it's risky. Unless one is
  willing to keep up with security patches, harden their networks, manage
  backups, and provide reliable connectivity, this is a nonstarter. My time is
  worth more than what I'd save dealing with these potential headaches, so I
  wrote it off.
* **Protonmail** acquired a decent amount of hype because of their security and
  privacy-focused approach to hosting. They went as far as making it so that
  you need special tooling to decrypt your messages. This is an attractive
  offering, but also makes it so that you need to run a compatibility layer to
  use their service with standard email clients. Additionally, they don't
  offer a calendaring service (which is nice to have as most calendar
  applications use one's email address as a form of identity). Additional
  storage is also expensive for their service, costing $1/mo/GB over 5GB.
* **Hey** is a service by DHH (of Rails/Basecamp fame) that claims to have a
  solution for to many of the annoyances that taint ones
  relationship with their inbox. They're currently in a public beta, and I have
  an invite. Unfortunately, they're a nonstarter: no custom domain support.
  This is on their roadmap, but until it happens, no deal. They're also the most
  expensive provider on this list at $99/year.
* **Fastmail** is another well-regarded provider. They've been around for
  a while, have a competitive feature set, and present an air of competence.
  Of particular interest to me was their Gmail import/sync tool, and robust
  custom domain support.

While Protonmail was tempting with its privacy-first approach,
[email is fundamentally insecure](https://security.stackexchange.com/a/30094/29671). 
Given my threat model, I went with Fastmail as the sheer amount of
functionality won me over.

## Switching to Fastmail

Making the jump was pretty straightforward. I created an account and opted
to start with my custom domain.  After following their detailed instructions for
configuring my DNS settings, I was off to the races!

### Gmail migration

Next, I had to address my Gmail inbox - I'd like to have all my messages in one place.
I once tried downloading my entire Gmail inbox via IMAP. It took ages.
Thankfully, Fastmail's utility was quicker than that, and everything
synced over in a few hours.

Once enabled, Fastmail's Gmail sync continues to run. This means that I can gradually
migrate everything to my new address while continuing to receive email for both
in my new inbox.

### The clients

I use Gmail for the web, and for Android.  While the Gmail client can read
IMAP, I opted to jump into the deep end and switched to the Fastmail web and
mobile clients.

#### Web

The web client is noticeably snappier for me than the bloated mess that Gmail
has become, so switching was generally an upgrade. It also had the added
benefit of consuming far less RAM.  They have a comparable set of keyboard
shortcuts, and a well-supported native dark mode. The only thing I find myself
missing is having my unread message count in the favicon.

#### Mobile

It's pretty obvious that it's not a native app.  My guess is Fastmail went with
a cross-platform framework, like React Native, to optimize for development
speed over user experience. As a result, all the patterns for the platform feel
"off".

Performance is okay. I'll occasionally hit stutters, or overly long loading
indicators, and the frame rate doesn't feel similar to native apps on my 90 Hz
display. Offline caching is hit or miss, and there's no ability to adjust
caching per label.

A positive, however, is that one can do almost everything on mobile that they
can do on desktop, including setting filters (unlike with Gmail). And notifications
have been timely and prompt.

At some point, I'll probably kick the tires on a new app.

## Miscellaneous

* **Calendaring** - Like Google, Fastmail offers an integrated calendaring
  feature. They don't offer automatic event detection, but other features work. I'm not
  quite ready to switch calendaring services, so I was really happy to discover
  that, once connected to my Google Calendar, Fastmail calendar lets me use it
  as my default. This is wonderfully considerate.
* **Hangouts** - Hi, it's me: one of the 3 people who still use Hangouts. Naturally,
  it's not integrated with the Fastmail webapp, but there's
  [a standalone web client](https://hangouts.google.com/).
* **Smart Categorization** - I didn't realize just how reliant I was on
  Google's smart email categorization. Unfortunately, I'm now back to having to
  set filter rules to label emails. While tedious for the first few weeks, the
  work naturally lessened.  Additionally, it forced me to confront just how
  many unnecessary mailing lists I was on, at which point I KonMari'd them from
  my life.

## 5 months later...

I wrote most of the above sections 5 months ago. I, a procrastinator, didn't
get around to finishing this until just now. In that time, I've had no issues
with Fastmail, and plan on continuing to use their service. If this interests
you, [save on your first year with my referral link](https://ref.fm/u23826120)!
