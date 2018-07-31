# Cloud-Development
My process in writing a skybox with limited experience

... in China. 

This summer (the summer of 2018), I was invited for an internship by one of my classmates at RPI, [Yuze Ma](https://github.com/bobmayuze) at the Chinese company [Codemao](codemao.cn). Naturally, going overseas without understanding even 一点中文 was a great idea. So there I was, in China, and assigned to the graphics team of a game engine under supervision of [Mikola Lysenko](https://github.com/mikolalysenko). He provided me one simple task: Make clouds.

Ok seems easy right, we've all seen clouds: 

[Cloud :o](./images/cloud_ref1.png) [Cloud :O](./images/cloud_ref2.png) [Cloud! :c](./images/cloud_ref3.jpg)

#Cool.

So how hard could it be to make them? Well, first off, it turns out Mikola (aka mik) is also one of the developers of [regl](https://github.com/mikolalysenko/regl), a wrapper for webgl, so I would first have to learn webgl and regl. Also, the codebase is written in [typescript](https://github.com/Microsoft/TypeScript), and seeing as I didn't even know js it was looking pretty rough. However, after one training montage I was ready to start making some clouds.

To begin, I learned from [the best](https://www.shadertoy.com/). and pieced together a github test envrionment that mirrored the codebase without fear of stumbling through mysterious things like ~client server interactions~, ~regl contexts~, and ~well written code~. [This](https://github.com/Maydit/regl-cloudtest) is what I came up with. As you can see in the earlier stages, it's a simple raymarcher, but let's be honest, I didn't really know exactly what that meant when I started this either, so I'll explain it for you: [skip this](#marching forward)

Graphics is composed in a series of steps, called the graphics pipeline, where the input polygons are converted into pixels and the pixels are given colors. This pipeline is executed by shaders (programs written in shader language) on the GPU. Regl made it super simple to [~~copy~~ borrow some code](https://github.com/regl-project/regl/tree/master/example) and just write a shader, so I guess you don't really need to know that, but what is important is that the (fragment) shader takes the input pixel and outputs its color. In this case, I had to take the pixel and give it either a sky color output or a cloud color output. Doing this with raymarching means that the input pixel is converted to a world direction (in the form of a vector) and the shader follows that vector for a certain amount of steps to 'see' what's in that direction, gets its color, and returns it.

#Marching forward

This raymarcher does what the useful reference sources do and samples a [perlin noise](https://en.wikipedia.org/wiki/Perlin_noise) generator for the density of the cloud, and uses that density as the opacity of the cloud. For the cloud's lighting, at each point where the density is nonzero, a secondary ray is marched towards the sun, samples the same density function, and uses this value to shade the cloud darker based on the obsruction.

At a first test, this produced some beautiful clouds:

[fuck](./images/first_clouds.png)

After a rude awakening to aliasing, I produced this beauty:

[I cry evertim](./images/second_clouds.png)

#Packing up

Ok, looks good, after that I was completely done. Jk jk. 
