---
layout: post
title: "Markdown Tips for Jekyll"
author: "Wolf Vollprecht"
categories: posts
---

<div class="subtitle">
How to apply custom style with custom HTML tags and CSS for specific elements with MarkDown
</div>

As mentioned in the previous blog post, I just set up a new Jekyll instance for blogging like a pro (as opposed to blogging with Medium). Medium already really limits the ways of styling content --- something I've always disliked. For this blog I wanted to have two different ways of displaying source code: an expanded view, and a slightly smaller, non-expanded view. However, as far as I know there is no way to add some HTML tags to Jekylls source code renderer. That's why I came up with a small hack.

This should work fine to apply custom style to code blocks, but can also work to style specific images or head lines. Note that for those you can also achieve the same effect by using inline HTML and write out an image tag with a special class yourself. But if you'd rather have a quick way of styling specific page elements in MarkDown, this trick might help. 

First, I just add a new element *before* my source code using custom HTML (but not surrounding it because then the source code isn't properly highlighted!):  

```md
<expand-sourcecode></expand-sourcecode>
```py
import numpy as np
np.array([1,2,3,4])
# ... and so on
```

And then it's easy to apply some specific CSS to the immediately following sibling element, thanks to CSS:

```css
expand-sourcecode + div.highlight
{
    width: 100%;
    min-width: 100%
}
```

The effect will be as follows:

<wide-source></wide-source>
```py
def quick_widget(package_name, version, has_view=True):
    def quick_widget_decorator(cls):
        from traitlets import Unicode
        name = cls.__name__
# and so on, and so on...
```

Pretty neat if you want to add some styling to otherwise unstylable elements!

If we want to add a wide image, that's straight forward now, too -- just use the same idea:

![]({{site.baseurl}}/assets/images/placeholder/wolf.jpg)

<wide-source></wide-source>
![]({{site.baseurl}}/assets/images/placeholder/wolf.jpg)