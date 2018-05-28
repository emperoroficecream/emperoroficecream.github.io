---
layout: post
title:  "Colors and Materials"
date:   2018-05-28 15:07:19
comments: true
tags: webgl computer-graphics color material
---


Think how you would draw a lemon by hand?
There are probably a million ways of doing it. But here's what I would do:

1. Draw an oval.
2. Fill in yellow inside that oval.

Now think how would a computer do the same thing? Recall how the computer graphics pipeline looks like:

1. The application informs GPU to draw an oval
2. Start out with a bunch of vertices transformed into screen space
3. Organize vertices into primitives (usually triangles, but webGL supports other types of primitives)
4. Primitives are rasterized into fragments, which then are shaded to compute a color at each pixel

You can see the process is not **too** much different from how we draw by hand: we both need to figure out the location, or the shape first (vertices for GPU), and then "fill in" the color (rasterization for GPU). 

Historically, the hardware that is specialized in rendering graphics does not give that much control to developers: it takes in a lot of parameters but the functions in the pipeline are fixed, hence the name "fixed-function rendering pipelines". What is exciting about openGL (and webGL) is, now that we have much more computing power and faster CPUs, the pipeline doesn't have to be fixed any more; developers gain much more freedom by configuring their own functions in the pipeline, programs we call "shaders". There are two types of shader programs: vertex shader and fragment shader, corresponding to different processes in the pipeline. Today we are gonna look at the part where we figure out what color to use, and how we can make the surface of our lemon look more photo-realistic than just a yellow shade.

## A quick primer on different models of light
Historically, there are two competing theories of light: Sir Isaac Newton's particle theory, which sees light consisted of particles travels in straight lines, and Christian Huygens' wave theory, stating light was a wave. James Clerk Maxwell managed to come up with a unified theory that explains electricity, magnetism and light, the electromagnetic theory of light. We are going to use different mental models of light to explain and replicate different aspect of how light works. When we try to explain human perception of color, we might find light as a wave more helpful since the difference in wavelength is the source of different colors. On the other hand, when we study how surfaces of objects reflect and/or refract light, seeing it as straight lines with direction makes matters much simpler. 

Since we are drawing a lemon, the choise of color is a no-brainer, [lemon](https://xkcd.com/color/rgb/#lemon). But technically how do we render this color on a screen? If you have done web development, it'd be just one line of CSS: 
```
color: #FDFF52 // BTW, is the surveyed color for lemon: https://xkcd.com/color/rgb/#lemon
``` 
But what exactly is happening when we set the hex code this color? Let's take a brief moment to dive into color science :rainbow:.



## Quantifying colors

When the cryptic `FDFF52` specifies a color in [sRGB color space](https://www.w3.org/TR/css-color-4/#sRGB), which is the de facto color space used on the web. We might be more familiar with RGB color space, and sRGB is a modern adaptation of it (as modern as 1996). Both of them use a coordinate system of three primary colors: red, green, and blue, and blend them together to create other colors. So `FDFF52` is actually made up of three components, and to decode it we divide each of them by `0xFF`. So we have:
```
0xFD / 0xFF = 253 / 255 = 0.99
0xFF / 0xFF = 255 / 255 = 1
0x52 / 0xFF = 82 / 255 = 0.322
```
We can see while the hex code is a compact way to set RGB values, there are alternatives too: integers, ranging from 0 to 255, and floats, from 0 to 1. Imagine for a second a world where no such system exists. Meaning, there is no way to quantify color. How do people communicate about colors? Solely relying on verbal means, we always need a reference: blood-red, apple-green, sky-blue. We use common objects in our daily life as a pool of epithets to describe colors. The problem with this approach is references are ambiguous, vague and subjective. Sky is not always blue, and even when it is, there are numerous shades of blue the sky has. Another possible way of communicating colors is simply replicating colors.

A person with normal vision has three types of cone photoreceptor cells that sense different and yet overlapping wavelengths of light. They have different peaks of spectral sensitivity and are thus named L (long), M, (medium), and S (short), roughly corresponding to red, green and blue.

If we compare the spectral distribution of a point on a lemon in real life, and an image of lemon displayed on computer screen, you might be surprised that they don't look very close enough, even though humans perceive both colors as identical. The reason is our cone cells are sensitive to different lights, so the ultimate color sensation registered is a combined result of spectral distribution and cone cell sensitivities to different spectral powers. If we overlay the real lemon spectral distribution with human cone cell sensitivity graph, we get a graph that now looks much closer to the on-screen lemon one. The phenomenon when two colors are composed of different wavelengths of light but appear the same to the human eye is called metamerism. The huge lesson here is, to replicate a color, we don't necessarily need its compositions of lights; we just need to know color matching functions.

The CIE 1931 RGB color matching functions answer questions like this: given three lights, red, green and blue, how do you combine and adjust the power of each light to simulate any given color?

This is what they come up with:

![Human cone sensitivities](https://upload.wikimedia.org/wikipedia/commons/thumb/6/69/CIE1931_RGBCMF.svg/325px-CIE1931_RGBCMF.svg.png)


To make matters even simpler, let's take out of brightness out of the equation, meaning light red and dark red are the same color, as long as it's the same red. Chromaticity is colors without luminosity, or holding luminosity constant. Moreover, chromaticity is calculated in such a way that it is a ratio of primary colors instead of absolute amount. More specifically, in our chromaticity coordinate system, `r + g + b = 1` is always true. Another great thing about chromaticity is, now that we have essentially reduced a three dimension coordinate system into two, thus can easily draw a diagram of chromaticity(r and b, since we can always calculate by doing `b = 1 - r - g`) and every perceptible wavelength:

![Chromaticity Diagram](https://upload.wikimedia.org/wikipedia/commons/thumb/1/16/CIE1931_rgxy.png/325px-CIE1931_rgxy.png)

The three points representing primaries form a triangle, which is also the gamut of that color space.

Notice something that doesn't seem quite right? Say if the target wavelength is 520 nm, and the red light you need is a negative value! How could that be?

It turns out that with just red, green and blue primaries, there are colors that humans can perceive but cannot be expressed in these primaries. In an experiment conducted in the 1920s, researchers provide three primary lights to the subject and ask them to simulate a given light. To simulate a light with 520 nm wavelength (a bright green), the subject not only has to remove the red light, but also add it to the other side. 

<img src=https://cdn-images-1.medium.com/max/800/0*Sm_qW8K7YU9MqTjY.png alt="Wright Guild Color Matching Experiments" style="width:300px">

source of image: https://medium.com/hipster-color-science/a-beginners-guide-to-colorimetry-401f1830b65a


To adjust the chromaticity diagram so the curve would only have on non-negative values, we can apply transformations to the RGB color space and end up with three new primaries, X,Y and Z:

![Gamut of the CIE RGB in the CIE xy chromaticity diagram](https://upload.wikimedia.org/wikipedia/commons/thumb/6/60/CIE1931xy_CIERGB.svg/325px-CIE1931xy_CIERGB.svg.png)


There is a little problem: X, Y and Z are imaginary colors that don't exist, meaning they do not correspond to any spectral distribution and are purely artificially invented. So why is it still useful? Tons of other color spaces that are created for specific digital displays use the CIE XYZ color space as a standard reference. Among them is sRGB, the color space we mentioned earlier and the standard color space for web. Each pixel on our screen has three sub pixels, each being the primary color of that display, which does not necessarily match those of sRGB. With XYZ color space, translating our hex code into a color is just series of linear transformations: sRGB -> XYZ -> display color space. That's how our lemon remains looking the same yellow across devices. 

Now that we have an oval filled in with a realistic lemon yellow. But we still miss something. How can we tell it's a lemon? 

# Materials

## A primer on vectors

It's useful to brush up on linear algebra a little bit as lights are denoted as vectors in computer graphics. 

* a vector has a direction and a length
* a unit vector (or a normalized vector) of one vector has the same direction but has a length of 1 
* a surface normal is a unit vector that is perpendicular to the tangent plane to that surface
* the dot product of two vectors is the product of their lengths and the cosine of angle between them, so for two normal vectors their dot product is just their angle's cosine

## The big idea that makes computer graphics possible

> We do not expect to be able to display the object exactly as it would appear in reality, with texture, overcast shadows, etc. We hope only to display an image that approximates the real object closely enough to provide a certain degree of realism. -- Bui Tuong Phong

Now take a moment of how you might approach the problem of rendering graphics on a computer. A brute force, perfectionist approach is:
* compute every pixel on every surface in the scene and render it
* the smallest unit of rendering is the pixel

Apparently this would cost you a lot of computing power. 

Another approach is divide the surface into a finite number of smaller pieces (polygons, triangles, .etc). That way we don't have to compute color for every pixel; each tiny piece is an approximation of that divided area. It is a balance between computing power and photo-realistic perfection. Sometimes a certain degree of realism is good enough, and most importantly, cheap and feasible. 
## Components of surface color

* Emissive: light emitted by the object itself
* Ambient: light of the environment, most of which comes from reflected surfaces
* Diffuse: incident light scattered  at many angles; appear matte
* Specular: incident light reflected at one angle; appear shiny

![Phong model](https://upload.wikimedia.org/wikipedia/commons/thumb/6/6b/Phong_components_version_4.png/655px-Phong_components_version_4.png)

We'll focus on diffuse and specular surfaces.

## Diffuse material

Let's assume for simplicity's sake that light source is always from the same direction, like the sun. A diffuse material is defined as a surface that scatters light in all directions above the plane according to [Lambert's cosine law](https://en.wikipedia.org/wiki/Lambert%27s_cosine_law). This law states that the intensity of incident light is proportional to the angle between the direction of the light and the surface normal. This makes intuitive sense. Imagine a globe, like our Earth, hit by sunlight: the equator gets the most intense light, while the poles get the least. However, there is a balancing factor: as the angle increases, the area of the surface visible through the pixel proportionally increases, hence more photons hitting the viewer's eye even though less light per unit area is reflected. The net result is the viewer's location does not matter. The shade of the direction depends only on the angle to the light itself. So, in this function we has taken out one variable, which makes diffuse surface faster to compute.

## Shading normals

So far our lemon is still made up of triangles. To have a tessellated surface made of triangles look smooth, we need a new concept: shading normals. Recall shading is the process of interpolating colors between vertices. Geometric normals for our lemon's triangles have discontinuities at the vertices because each triangle's surface is flat. A shading normal could be the normal at a vertex. Note that there are different ways of extracting shading normals as they are made up, unlike geometric normals.

## Diffuse reflection equation
Now we can have an equation where we use two coefficients to adjust the contributions of ambient and diffuse reflection: fragment color = K<sub>A</sub> * material + K<sub>D</sub> * material * the dot product of surface normal and incident light. 

## Specular material

There are a lot of examples of specular materials, mostly artificial: polished metals, plastics, polished wood, glass, .etc. We know from experience that these "shiny" objects look different when viewed from different angles so now, unlike in our diffuse surface color equation, we need to take into account of the viewer's location.

## Blinn-Phong reflection model
One standard way to simulate specular material is this model: specular color = MAX(the dot product of surface normal and half angle, zero), raised to a power of shininess. Half angle is the surface normal at which you point a mirror surface so the light reflected directly towards the viewer, aka the halfway between incident light and viewer's direction. An intuitive way to think why we use this dot product is to think the surface has microfacets or "wrinkles". Smoother surface has less ans "shallower" microfacets and vice versa. The angle between light and half angle is a distribution of individual angles for microfacets. Rougher surfaces have a larger range of these angles, making the specular color smaller.

Recall from "shading normals" section there are other ways of shading. One of them is instead of shading at each vertex, we do shading at each pixel, making a much better reflection model. In our case, we only need to interpolate the existing shading normals at vertices. Recall again from our highly abstracted pipeline, we have a step where vertices are transformed, and is programmable. This is vertex shader. We can feed vertices as before. As output, we get the screen position and transformed normals. Vertex normals are interpolated per pixel, which then are input to the fragment shader, and then evaluates the reflection model and computes the RGB to be displayed. This step used to be taken at vertex shader. The flexibility of shader programming makes it possible to delay this step and add this Phong shading algorithm.

# Transparent material
Transparent material is more challenging than diffuse and specular in the sense that when light reaches the surface, it is not only reflected, but also refracts and changes its course. There are still a couple of simple models to simulate transparency.

## Screen-door transparency
This model is a naive solution of blending two overlapping colors: show a little bit of both! Say there is a white object seen through a red glass, we only need to draw every other pixel to be red and leave the rest white. A more generic way to calculate filtered color is to use an "over" operator:
color of filtered object = alpha * color of filter + (1 - alpha) * color of original object. Essentially a linear interpolation between two colors.

---
References:

1. [A Beginner’s Guide to (CIE) Colorimetry
](https://medium.com/hipster-color-science/a-beginners-guide-to-colorimetry-401f1830b65a)
2. [Color: From Hexcodes to Eyeballs](http://jamie-wong.com/post/color/)
3. [CIE 1931 color space](https://en.wikipedia.org/wiki/CIE_1931_color_space)






