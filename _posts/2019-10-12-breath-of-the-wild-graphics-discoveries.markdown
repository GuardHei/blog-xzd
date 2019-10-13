---
layout: post
title: Several Graphic Discoveries in Breath of the Wild
comments: true
date: 2019-10-12 07:50:48.000000000 +09:00
author: William Xie
tags: Analysis
---

# Introduction
>I've been playing Breath of the Wild for a really long time (200h+). After beating Ganon and playing around with the dlcs, I have paid more attention to the details of the world -- not the details in gameplay (though I have already been impressed again and again), but the subtle graphic details that contribute to build every gorgeous frame of a vivid fantasy world. I'll try to analyze some of my interesting discoveries (and probably been discussed before somewhere else). Be free to point out any of my mistakes!

## 1. Rimlight Effect
To depict the evilness of player's strongest enemy Ganon, the developers mainly uses the black and purple-red colors, as shown in the image. ![Ganon (in DLC)](https://i.loli.net/2019/10/12/eK1kLjItqugOJr2.jpg)
Since I'm not an artist, I just want to point out the implementation of the effect outlining Ganon's body. These purple lines have uneven widths and can be easily inferred that it is just a common and simple rimlight effect. The power of the outline purple-red color is calculated by `1.0 - saturate(dot(view_direction, surface_normal))`. This effect is also used in Dark Souls for Spirits (or called Phantoms), and is a lot of mobile games because it is very cheap.

In addtion, this rimlight effect is also added when rendering the characters.
![Link](https://i.loli.net/2019/10/13/MycoErxJ1tRiC4e.jpg)
Link's body is lit at the edge, which is the rimlight effect. Many mobile games use this technique combined with bloom to simulate skin sub-surface scattering (SSS) effect at a low cost. SSS gives the translucent feeling of the skin. In most cases, the edges of the human body are the least thick, so the most translucent. Therefore, it pretty much looks like a strong rimlight effect at the edge of body. It works especially well with the cel shading on characters in BOTW, which removes lighting details.

## 2. Screen Space Reflection (SSR)
From some articles I was already told that BOTW only used SSR inside shrines, probably because of relatively smaller stages within the shrine, meaning lower performance pressure. And in the open world, the developers used a delayed reflection capture camera to render the reflections in several different frames in order to reduce the render pressure. Actually, SSR is not really an expensive post process operation on current generation's consoles, like PS4, Xbox One, or gaming PCs, especially it has a constant cost as it is a screen space operation. However, considering the game also runs on the handheld consoles with limited bandwidth (pretty low frequency for gpu ram), BOTW's decision of using SSR is really brave.
![SSR](https://i.loli.net/2019/10/12/BOnFgUNIZycf6qj.jpg)
Above image is taken inside a shrine, and it can be clearly seen that the wall is a SSR material, reflecting Link's appearance. BOTW didn't really use any magics. What the developers have done is pretty much just reducing the resolution of the raymarching process, and number of steps. It explains why the reflection color usually has a long tail and is not precise. Moreover, the unclear reflection images and the rough wall material make me think that it is actually a glossy reflection with blurring. The blurred reflection doesn't improve the visual quality of reflection, though it has an unexpected use in another place I'm going to talk about later.

## 3. Area Light?!
I was super impressed when I first saw this scene.
![Area Light](https://i.loli.net/2019/10/12/yS1HXIPOEjuCiqF.jpg)
I have thrown a guardian spear on the ground, and you can see that its emissive spear head acts as a light source, and the ground below is rectangularly lit, which is the shape of the spear head. The first thought jumped out of my mind was -- wtf, they implemented area lights in BOTW? Area lights are known for costly performance. However, in BOTW, you can throw whatever amount of guardian weapons on the ground as you want, each acting like an area light, and not affecting the framerate. So how do they manage to make this?

My initial thought was an exaggerating bloom effect, but the lit surface looks still correct from other view angles (with a normal bloom effect), which implies the surface is lit up by lights -- or is that really the case?

I soon realized the answer after changing to this view angle.
![SSR Area Light](https://i.loli.net/2019/10/13/YWH6deGQcmo7Fhw.jpg)
In this image you can see that even the small orange patterns light up the ground, and the lit area is thicker than expected. The answer is very clear actually at this moment -- it is just the SSR effect, so the lit area is thicker due to the imprecise raymarching result. I didn't think about SSR at first because the ground didn't reflect the "non-emssive" parts of the guardian spear, which acted differently from the walls. The developers have changed the parameters so that the ground's SSR only reflects colors with high brightness -- in most cases, it means lights, creating this illusive "area light" effect.

This is super clever to me, as this simulation literally has no extra cost -- there is already a SSR effect, so the performance won't be affected. But at the same time, the visual result is way better than the solution of simple point lights, which is probably more expensive due to increased lighting computation in pixel shader.

[Left Unfinished]