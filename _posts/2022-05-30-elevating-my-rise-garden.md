---
layout: post
title: Elevating My Rise Garden
description: Combining different parts of the Kotlin ecosystem to improve my hydroponic system.
---

> This post is a bit more of a journal entry than an instruction manual. If you too are someone who dabbles in
> hydroponics, I hope you find this post inspiring, and maybe you'll want to join me in what I'm building.

# Where It All Started

When the COVID-19 lockdowns started for us in March 2020, my wife and I picked up a number of new hobbies. I was just
starting to get into outdoor gardening by building some elevated planters for our patio and my wife was getting into
indoor plants. For both of us, there is just something about tending to and cultivating something that grows. Seeing it
start small, the little changes each week, and then months later when the plants are huge. And I particularly liked the
part where you got to eat it! Looking back it is not really a surprise, that once autumn started, I would find
hydroponic gardening as a way to continue indoors.

In September 2020, we purchased
a [3 tier Rise Gardens hydroponic system](https://risegardens.com/products/triple-indoor-garden) (I promise this isn't
an ad). Really easy to put together, looked good in the house, and had all the tech a software engineer could want. Over
the next year as we grew everything from herbs to lettuce to peppers to tomatoes, I learned a lot about hydroponic
gardening and thought of so many, different ways I could make my use of the system better.

![](/assets/elevated-rise-garden.png)

# Project 1 : Dosing Nutrients

Nutrients within the water of a hydroponic system are usually measure as EC (electrical conductivity) or PPM (parts per
million). The pH of the water is also important to measure and maintain. Over the lifetime of the plants within the
system, the pH should remain constant while the EC is typically increased as the plants get bigger. High levels of EC
can damage younger plants and not enough EC can cause problems with older plants. It's a balancing act.

Most tap water has too high of a pH level so some form of acid needs to be added to lower it to around 6. Adjusting the
pH of water is pretty easy, and you can get both pH up and pH down mixtures that you add to the water to adjust its
balance. The amount you add is pretty small, usually measured in milliliters. You can also use a pH "buffer" which is
a concentrated liquid with a set pH measurement you add to water to resist changes to the pH balance.

For nutrients, the primary elements for plants are nitrogen (N), phosphorus (P), and potassium (K). There are a number
of other micronutrients which should also be included, but most fertilizers will include those as well, so you don't
need to worry about them too much. Different mixes of N, P, and K are needed by the plants within a hydroponic system
depending on what stage of development the plant is at. Is the plant still in the vegetative phase, growing tall and
lots of leaves? This plant will need good amounts of nitrogen and potassium. Is the plant starting to flower or ripening
fruit? Should lower the potassium and increase the phosphorus.

All of this water management gets rather tedious. Not because it's hard, but because measuring out pH balancer and
liquid nutrients every time you need to add water to the hydroponic system can become time-consuming. Automation to the
rescue!

I'm not very good at the hardware side of things, but I'm familiar enough to figure things out. So I grabbed the
Raspberry PI I had lying around; bought a few peristaltic dosing pumps, relays, and other electronic components; and
built myself a little dosing station I could use when needing to refill my hydroponic system with water.

![](/assets/elevated-dosing-station.png)

To activate the dosing pumps I need to manipulate the GPIO pings on the Raspberry PI. For this I
used [Pi4J](https://pi4j.com/). Pi4J is a nice Java library for working with all the I/O capabilities of a Raspberry PI.
I built a [Ktor](https://ktor.io/) server around this, so I could activate each pump from a REST API. I ran the server
as a Docker container on the Raspberry PI for easy deployment and updating. Then I built a really
simple [Jetpack Compose](https://developer.android.com/jetpack/compose) Android app to call that REST API.

![](/assets/elevated-android-pumps.png)

Combine all of this, and I could now dose out exact measurements of pH balancer and nutrients from my Android phone -
without having to hand measure everything - whenever I was adding water to the hydroponic system. But I wanted more! Or
rather, to have to do less.

# Project 2 : Water Measurements

Each time I would add water to the hydroponic system, I would measure the EC and pH level of the system to know how much
pH balancer and nutrients I needed to add. Again, while not difficult, this was another step which needed to be manually
performed each time water was added. Plus, I now had a station which could dose out nutrients. If only I could get
readings of the pH level and EC into the Raspberry PI, I would only need to add water to the system and nutrients could
be added automatically.

This is when I splurged and got myself some [Atlas Scientific](https://atlas-scientific.com/) sensors. These sensors are
really high quality and can be submerged within the water of the hydroponic system indefinitely. They also sell a hat
for connecting these sensors to a Rasbperry PI, which made integration incredible easy.

![](/assets/elevated-dosing-auto-1.png)

Once these sensors were hooked up to the Rasberry PI, I updated the Ktor server I had to take a pH and EC reading every
minute. I stored these measurements in a SQLite database Using [SQLDelight](https://cashapp.github.io/sqldelight/) on
the Raspberry PI, and created REST endpoints to be able to retrieve them. Then I updated the Android app to show a graph
for each sensor of the readings taken.

![](/assets/elevated-android-graphs.png)

Finally, I tucked the Raspberry PI into the hydroponic system, put the nutrient dispenser over the water reservoir, and
created a schedule which automatically check the pH and EC levels of the reservoir and dose out adjustments as needed.
All I had to do now was to add water or adjust the constraints of the pH and EC levels, and the system would take care
of itself.

![](/assets/elevated-dosing-auto-2.png)

# Project 3 : Taking it to the Cloud

With everything so automated, I was having fun looking at graphs of the sensors readings and making adjustments to the
dosing constraints and schedule. But this is where I was a little sad: I couldn't really show anyone. See with me not
wanting to expose the Raspberry PI server to the internet, I could only really use the Android app when I was on my
local network. This meant I couldn't really show off all my work unless someone was over at our house. That's no fun; I
wanted to show everyone what I had created!

So I started by getting a [Digital Ocean](https://www.digitalocean.com/) droplet created with Docker installed. Set
about securing that droplet with [Nginx](https://www.nginx.com/) and added a [MongoDB](https://www.mongodb.com/) Docker
container as well. Then I built yet another server - this time
with [Spring Boot](https://spring.io/projects/spring-boot) - to expose a REST API for users, devices, and sensors which
could be used by the Raspberry PI.

Sensor readings can be submitted directly to the API. And to dose out nutrients, the Raspberry PI connects to a
websocket, and the server forwards actions which need to be performed. This also allowed me to know if the Raspberry PI
is currently connected, since I could flip a flag whenever the Raspberry PI connected to or disconnected from the
websocket.

![](/assets/elevated-android-devices.png)

This also allows you to now log in to the incredibly basic website - built
with [Compose for Web](https://compose-web.ui.pages.jetbrains.team/), a natural pairing for the Android app - and check
out how my hydroponic system is doing! All the code and guest login information can be found on GitHub, in
the [elevated](https://github.com/bnorm/elevated) repository readme.

![](/assets/elevated-webpage.png)

# Project 4 : What's Next?

I'm not sure what's next. Probably small improvements here and there, particularly in the UI portions of the project.
There's also a large number of hardcoded things throughout the project which I would like to remove to truly support
multiple Raspberry PIs. I'd also like to add a few more sensors, to be able to measure the water level for example, so
the server can send me a notification when I need to add water. But in general I've been having a lot of fun working
with such a wide range of things: from software to hardware, from backend to frontend, and all the different libraries
and frameworks available within the Kotlin ecosystem.

# Summary

All the code for this project can be found on [GitHub](https://github.com/bnorm/elevated). I'm not really an expert in
any one particular area (especially not Android or web development) so I welcome any and all contributions to the
project. And ff you found this inspiring, have your own hydroponics system, and want to do something similar to what I
have done, hit me up on [Twitter](https://twitter.com/bnormcodes)! Maybe we can get your Raspberry PI automatically
measuring and dispensing nutrients as well!
