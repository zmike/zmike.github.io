---
published: true
---
[![fml.png]({{site.url}}/assets/fml.png)]({{site.url}}/assets/fml.png)

# Addendum
It turns out [this](https://gitlab.freedesktop.org/mesa/mesa/-/issues/9589) was the product of a tiler optimization I did [earlier this year](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/21801) to pipeline texture uploads without splitting renderpasses. I was (wrongly) assuming that the PBO stride would always match the image format stride, which broke functionality in literally just this one corner case.