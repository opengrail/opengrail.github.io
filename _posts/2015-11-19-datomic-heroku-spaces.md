---
layout: post
title:  "Datomic On Heroku!"
date:   2015-11-19 19:59:38 +0100
categories: jekyll update
---

**Datomic On Heroku**

**Datomic** is an immutable, append-only databases that stores all of your new and historic data. It's easy to keep a history of prices or customers or whatever but not just as individual entities but as part of the history of the entire database. It comes from Rich Hickey and the other folks at Cognitect that brought you Clojure. It retains the ACID properties of the traditional RDBMS and has revived a logic programming interface to make queries using Datalog.  The other interesting aspect of Datomic is that it can offload the job of permanent storage to another database - like Postgres or Riak. There's a ton more innovation from Datomic so check out the videos and other introductory material on [the Datomic website][datomic-site]

![Datomic components]({{ url }}/assets/DatomicHerokuVisuals-1.jpg)

**Heroku** is a well known Platform As A Service (PAAS) which is rightly famed for its simplicity and its ease of use.  It has some strong opinions about how to make applications scalable using a stateless application model and has long supported the now hotness that is Linux containers. It has also made using Postgres incredibly easy with tools to fork and clone databases in seconds. Whilst Heroku started with Ruby, it can now run Java, node.js and even Clojure apps. To get started with Heroku go and download the [Heroku Toolbelt][heroku-cli] so you can have a CLI.

**BFFs FTW?**

I like Datomic and Heroku on their own merits. A year or so ago when I first started getting into Datomic I thought they could be friends. Two interesting, powerful and yet simple technologies with a lot in common: a focus on simplicity, friendly to Clojure and more than a passing interest in Postgres. 

A small diversion on the technology components involved with Datomic. Apps contain the Datomic Peer library which reads directly from the storage. The transactor is used occasionally to record novelty into the database.

Surely they would hit it off. But no, there was trouble in paradise: Datomic, and most specifically the transactor, assumes that it is configured behind a secure network while Heroku assumes that you want everything out in the open. It was like taking a nun to a nude beach.

![Datomic components]({{ url }}/assets/DatomicHerokuVisuals-2.jpg)

**Leveraging strategic synergies in skinny jeans**

Well that was awkward. But it seems like the naturalists are putting on some clothes. Whilst Heroku rose to fame with the high profile start-ups that used its service (Uber and Automatic as a few examples), it is turning its attention to capturing the hearts and minds of larger, more established enterprises. It hasn't quite put on a three-piece suit and tie but a pair of skinny jeans and a conference tee shirt is already making it seem more attractive to the nuns. 

**Spaced out**

Ok listen, forget the nuns, let's get back to what Heroku are doing to appeal to more traditional businesses: a service called Private Spaces which is Virtual Private Cloud (VPC), done in a way that seems true to the spirit of the Heroku platform - simple and easy to use. You can deploy web apps and worker processes as easily as before but they can now be hidden behind the curtain provided by the VPC. This makes a better match for Datomic, which just to clear it up, is also not really a nun.

I have been fortunate to have access to the private Beta version of Private Spaces for work reasons and it meant that I could try out my idea again on the side. Maybe the introductions would be more cordial this time around. And of course it was, so let's get into the details...

**Pack, Hack and Stack**

Heroku apps are deployed using a buildpack, which is the script needed to install and run up any service on the platform. There are some official build packs for Ruby, Java, node.js, etc... but you can can extend them or make your own and people do. So I did. Datomic has a dependency on Java so it made sense to use the Heroku official Java buildpack as a starting point for the hacking.

**Starting, a fight**

Datomic itself comes in a few flavours: Free, Pro and Pro+. The Free version is obviously the most accessible so I wanted to support that one and indeed it is the default. Free is great for playtime but only stores the data in memory or a disk that is local to the app. It is not a great fit for Heroku: Did I say Heroku is built on stateless principles? The disks are also ephemeral. So with the free version everything is wiped when you restart your app. All to say that it's OK to kick the tyres but it doesn't really take advantage of the Heroku platform very well and one could even say that the Heroku platform is slightly hostile. If deleting all of the database's data could be described as hostile.

![Datomic components]({{ url }}/assets/DatomicHerokuVisuals-3.jpg)

**ZOMG BFFs FTW fr raal**

So for the Datomic Free version story is not so great. But Datomic has another free to access version called the Pro Starter Pack. This allows up two web apps (also called Peers) to connect to the database and any one of the backend database storage engines can be brought into play.  With Datomic Starter Pack plus Heroku Postgres, we really do have game on.

**Datomic Starter Pack** [Ceremony alert]

There is a little bit of ceremony (registration and email) to obtain a license for the Datomic Starter Pack so I'll let you do that ... we'll still be here when you come back.

OK, so you're back and armed with your license you are now only a few seconds from deploying your first app.

**Heroku Private Spaces** [Ceremony alert]

OK, I was fibbing.  There is also a little ceremony on Heroku as you will need to have an Enterprise account - contact the Heroku account manager to organise that.

OK, so you're back again. Phew! Let's get started...

![Datomic components]({{ url }}/assets/DatomicHerokuVisuals-4.jpg)

**Create your transactor app**

{% highlight bash %}
heroku apps:create my-xtor -s my-space
{% endhighlight %}

**Configure the Datomic buildpack**

{% highlight bash %}
heroku buildpacks:set https://github.com/opengrail/heroku-buildpack-datomic -a my-xtor
{% endhighlight %}

**Provision your database**

{% highlight bash %}
heroku addons:create heroku-postgresql
{% endhighlight %}

The Postgres addon will be configured at the lowest cost - free - option, which is limited to 10k rows.

**Clone the demo transactor app**

{% highlight bash %}
git clone https://github.com/raymcdermott/space-xtor
{% endhighlight %}

**Configure your Datomic license for Heroku**

{% highlight bash %}
heroku config:set DATOMIC_LICENSE_PASSWORD=license-password
heroku config:set DATOMIC_LICENSE_USER=license-user
heroku config:set DATOMIC_TRANSACTOR_KEY=license-key
{% endhighlight %}

**Push the app**

{% highlight bash %}
git push heroku master
{% endhighlight %}

**Create your web app**

{% highlight bash %}
heroku apps:create my-xtor-web-front -s my-space
{% endhighlight %}

**Attach your database**

{% highlight bash %}
$ heroku addons -a my-xtor

Add-on                                           Plan       Price
───────────────────────────────────────────────  ─────────  ─────
heroku-postgresql (postgresql-xyzabc-1234)       hobby-dev  free 
 └─ as DATABASE                                                  

The table above shows add-ons and the attachments to the current app (space-xtor) or other apps.

heroku addons:attach postgresql-xyzabc-1234 -a my-xtor-web-front
{% endhighlight %}

**Clone the demo web app**

{% highlight bash %}
git clone https://github.com/raymcdermott/xtor-web-front
{% endhighlight %}

**Push the demo web app**

{% highlight bash %}
git push heroku master
{% endhighlight %}

**Test it all out**

{% highlight bash %}
heroku open -a my-xtor-web-front
{% endhighlight %}

**Operating costs**

All of the Datomic software and Heroku Postgres services are free. Be careful to watch your dyno use on Heroku as it's possible to go beyond the free tier if you play for too long.

[ A friend from Heroku (thanks @michrome) made me aware of a small caveat with respect to Heroku Postgres: plans below Standard-4 do not exist "in" the Private Space. In practise this is not a big issue as the connection must always be made with SSL ]

Obviously you can spend as much as you want with Datomic or Heroku once you get going. 

The code is on github so you can fork it and please let me know if there are any problems.

**Thanks**

Thanks for making it through, I hope you enjoy the wonders of Datomic on a great PAAS.

Zing me or ping me if this was useful via Twitter.


[datomic-site]: https://www.datomic.com
[heroku-cli]: https://toolbelt.heroku.com
