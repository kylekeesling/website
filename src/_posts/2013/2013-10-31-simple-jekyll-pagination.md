---
layout:     post
title:      Simple Jekyll Pagination
date:       2013-10-31 8:36:00
categories: jekyll coding html
---

I didn't really like the way [Jekyll's documentation][jekyll-pagination] went about building pagination links, mainly because it repeated the code necessary to render the button. That said I came up with the little snippet below that stores the proper URL using a [Liquid capture tag][liquid-capture] then just plops it into the link, allowing you to only code your button once :)

{% gist 7248951 %}

[jekyll-pagination]: http://jekyllrb.com/docs/pagination/#render_the_paginated_posts
[liquid-capture]: http://docs.shopify.com/themes/liquid-basics/logic