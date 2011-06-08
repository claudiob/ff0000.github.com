---
layout: post
title: "Drag'n'drop sorting in Django admin"
author: claudiob
excerpt: "Many clients have asked us for an easy way to change the order in which content appear in their web sites. We have created a Django package called sortable that allows elements to be reordered in the administration panel with a drag'n'drop interface."
---

Many clients have asked us for an easy way to change the order in which content appear in their web sites. We have created a Django package called [sortable](https://github.com/ff0000/django-sortable) that allows elements to be reordered in the administration panel with a drag'n'drop interface.

## An image is worth a thousand words

Here is what you can do once you add `sortable` to the installed applications of a Django project:
![sortable with Django admin](/images/django-admin-sortable.png)

The `sortable` package also works nicely when [Django grappelli](http://code.google.com/p/django-grappelli/) is used for administration:
![sortable with Django Grappelli admin](/images/django-admin-grappelli-sortable.png)


To install `sortable`, run `pip install sortable` or [download the source code](https://github.com/ff0000/django-sortable).