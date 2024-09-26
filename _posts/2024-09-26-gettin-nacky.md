# Rejection
It's hard. Nobody likes that feeling, especially after putting in a bunch of work, double-especially when that work is on a Wayland protocol.

That's right, the target of today's wayland-protocols governance update: [NACKs](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/341).

A NACK is intended to mean something like:
> this idea does not belong in wayland-protocols for `[technical reason]`

It's supposed to be the last resort when all other alternatives and gentler nudges have been exhausted.

There's been a lot of confusion over this concept over the years, specifically along the lines of:
* Who can actually NACK?
* When can NACKs be used?
* What's stopping my protocol from being NACKed?

I'm glad you asked.

# Definition
I've put up a [comprehensive proposal](to reform and define the NACK). The short of it is:
* Only people in [this file](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/MEMBERS.md) can NACK a protocol
* NACKs can only be used for extreme circumstances to block a protocol which does not belong in wayland-protocols
* NACKs now carry consequences if they are used improperly, including the potential removal of anyone using them improperly

This should cover all the basic cases. It's important to remember that a NACK can always be removed, which is to say that there's always room for discussion in Open Source.

If you're considering submitting a protocol proposal, don't worry too much about this! A NACK won't ever be the first thing you see, and you'll have ample time and room to discuss your ideas before anyone even considers bringing it up.
