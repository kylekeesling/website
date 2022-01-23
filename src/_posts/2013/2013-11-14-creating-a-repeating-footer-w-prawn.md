---
layout:     post
title:      Creating a Repeating Footer w/ Prawn
date:       2013-11-14
categories: rails ruby prawn pdf
---

I've been using [Prawn][prawnLink] for the past 3 years to generate PDFs in my Rails projects. It has a little bit of a learning curve up front but gives you much more power than just rendering out your web views as PDFs.

That said one problem I ran across was how to design/implement repeating footer. After a lot of digging I came up with the solution below:

{% gist 7478473 %}

If you are curious how you actually go about using this PDF in your project, you can refer to Ryan Bates' always awesome [Railscasts, episode #153][railsCastLink].

On a side note, Ryan [took the summer off][note1] and was originally going to come back in September, but then [later announced][note2] he wasn't ready to come back yet. While I can definitely understand burnout and needing some time away, I hope to see him back in action soon.

[prawnLink]: http://prawn.majesticseacreature.com/
[railsCastLink]: http://railscasts.com/episodes/153-pdfs-with-prawn-revised
[note1]: http://railscasts.com/announcements/11
[note2]: http://railscasts.com/announcements/11