---
layout: page
title: Projects
permalink: /projects/
---

### Cook Scheduler

<iframe width="100%" height="400" src="https://www.youtube.com/embed/4qAVHYqCLlc" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Cook Scheduler is a fair batch scheduler built on Mesos that is designed for providing a great user experience when there are many users who collectively want more resources than your cluster has available at any given time. 

I worked on Cook Scheduler from 2014-2017 and was the lead engineer on the project from 2016-2017. While I no longer actively work on Cook, I like to remain involved.  

### Waiter

<iframe width="560" height="315" src="https://www.youtube.com/embed/-MsTfWjRLj4" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Waiter is a web service platform that runs, manages, and automatically scales services without requiring human intervention.

I worked on Waiter as the lead engineer in 2015, before switching to focus on Cook. 

### Snow

<iframe width="100%" height="400" src="//www.youtube.com/embed/Mv0t7sKHgpU" frameborder="0" allowfullscreen=""></iframe>

For my capstone project at Brown, my team and me used Disney's recent work, [A Material Point Method For Snow Simulation (Stomakhin et al., Siggraph 2013)](http://www.disneyanimation.com/technology/publications) to implement our own snow simulator.

Disney's method is based on particle simulation. To update the position and velocity of the snow particles, we map the particles to a grid to compute the macro effects of the particles as well as collisions. Then we map updated velocities back to the particles and handle more micro effects. This produces a wide variety of phenomena you would expect with snow.

However, Disney's implementation was run on the CPU, restricting them to at least 3 minutes per frame and on average, 6.5 minutes per frame (from their times listed). Our extension to Disney's work is to bring most of the computation to the GPU so that we can provide an interactive environment for simulating the snow.

### heatmap-js

<iframe width="100%" height="400" src="http://wyegelwel.github.io/heatmap-js-google/code/examples/mouseOver.html" frameborder="0" allowfullscreen=""></iframe>

I created [heatmap-js](http://www.github.com/wyegelwel/heatmap-js-google) address my problems with google maps [heatmap library](https://developers.google.com/maps/documentation/javascript/examples/layer-heatmap). Specifically, there is no support for contour plots and limited ways to customize the heatmap. 

To address these problems, the library provides first class access to the functions used to compute the *heat* that each point contributes. This ends up being quite powerful; for example, including support for contour plots was done by simply using a different heat function. 

This library is **NOT** ready for production yet. First, it is implemented to only work with google maps. Second, there is still an obvious performance problem when zooming. 
