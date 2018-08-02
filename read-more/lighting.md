# Lighting

Lighting for the clouds was a difficult problem to get exactly right. Looking at the images in the readme, you can see the lighting went through many iterations. In the beginning, the cloud implementation looked like this:

```
vec3 sunDir;                                      //normalized direction of the sun
vec3 r;                                           //current look direction vector
vec3 eyePos;                                      //current player position or estimate
vec4 color = vec4(0);                             //accumulator
float stepsize = <reasonable distance>            //distance to step
for(int i = 0; i < TOTAL_STEPS; i++) {            //TOTAL_STEPS in (30, 100)
  if(color.w > 0.99) break;                       //early exit
  vec3 cloudPos = eyePos + r * float(i) * stepsize;
  float density = getDensity(cloudPos);           //lookup in density function, 3d perlin noise
  if(density > 0.01) {
    float lightingStepsize = <smaller distance>
    float light = 0.0;
    for(int j = 0; j < LIGHTING_STEPS; j++) {     //LIGHTING_STEPS in (5, 15)
      if(light > 0.99) break;
      vec3 lightPos = cloudPos + sunDir * float(j) * lightingStepsize;
      light = (1. - light) * getDensity(lightPos);//accum light
    }
    light = 1.0 - light;
    density = (1.0 - color.w) * density;           
    color += vec4(vec3(light) * density, density);//accumulate color and opacity
}
return color.rgb / (color.w + 0.0001);            //account for opacity and scale accordingly
```

Let's assume TOTAL_STEPS = 32 and LIGHTING_STEPS = 10. At worst this algorithm will take 320 calls of getDensity per pixel per frame. getDensity is a fbm perlin noise generator, so it will do a texture read 8 times with the implementation I was using. This is 2560 texture reads per pixel per frame in the worst case.

Unfortunately, this kind of fidelty doesn't even look good. There is intense banding and aliasing. To fix the aliasing I fudged the start position of the cloudPos and lightPos by a hash of the current time (ticks) and gl_FragCoord.xy, but even with this the banding was noticable and looked quite bad.

### Hacks

Somewhere along the development process I experimented with hacking in the lighting, by setting color equal to a mix between white and a nice looking color based on the value returned from the secondary ray march. This looked okay, and would have worked for a setting where it's always daytime. I attempted to generalize this approach to night time using something like
```
//0.0 <= timeOfDay < 1.0 // 0.0 is sunrise and 0.5 is sunset
sunBrightness = smoothstep(-0.05, 0.05, timeOfDay) - smoothstep(0.45, 0.55, timeOfDay);
```
and modified the colors based on that, but this proved too tedious.

### Better?

Later on, during changes to the atmosphere pipeline, things got moved around and the code was as such:

```
//attenuate opacity
for(i ...) {
  ...
  //get density
  float light = getLight(cloudPosition, sunDirection);
  //combine light and opacity to get color
  ...
}
```

and getLight looked like this:

```
#define CJSTEPS 6
//cloud lighting steps
#define THICKNESS 500.
//cloud thickness in meters
#define HEIGHT 3000.
//cloud height in meters
float getLight(vec3 r, vec3 sunDirection) {
  float sum = 0.;
  float dist = THICKNESS / float(CJSTEPS) / 5.;
  float cjInit = hash(...) //dithering as described above
  float totalDistance = cjInit;
  for(j = 0; j < CJSTEPS; j++) {
    float sampleDist = dist * (float(j) + cjInit);
    vec3 currPos = r + sampleDist * sunDirection;
    if (currpos.y < HEIGHT - 20. || currpos.y > HEIGHT + THICKNESS + 20.) break; //early exit
    float dens = getDensity(...); //3d perlin noise read
    if (dens > 0.0) {
      totalDistance += dist * dens;
      sum += dens * (1.0 - sum);
    }
  }
  float beerPowder = log(1.0 + sum * totalDistance / float(cjSteps)) / log(2.72);
  return beer_powder(beerPowder);
}
```

Using the Beer-Powder law as described in [Horizon Zero Dawn's paper](http://killzone.dl.playstation.net/killzone/horizonzerodawn/presentations/Siggraph15_Schneider_Real-Time_Volumetric_Cloudscapes_of_Horizon_Zero_Dawn.pdf). 

I implemented this as:

```
float beer_powder(float d) {
  float beer = exp(-d);
  float powder = 1.0 - exp(-d * 2.0);
  return 2.0 * beer * powder;
}
```

As you can see, this implementation has a lot of hacks and number-fudging but works on the principle of the Beer-Powder law. This law basically maps higher numbers to darker colors except for very small numbers, which are also mapped to higher numbers. The input to this is basically an accumulated sum of densities, scaled by the time spent in the cloud. In principle, in most cases, a ray will go through a cloud until it reaches the top or the bottom or samples a density of 0 repeatedly. Here we know that based on how many samples it took, the darker the cloud should be. This is the scaling in the second to last line. I applied a logarithmic scale to this because otherwise, values were too large and everything was too dark. Your mileage may vary.

This approach was too time consuming for the result. This style of lighting looked decent, but felt fake and there were still imperfections on distant clouds. At this point, I was optimizing the speed of rendering, and I considered alternative options for calculating the lighting.

The most viable approach was the cone sampling from Horizon Zero Dawn, but that approach still takes 6 samples of the density function per nonzero density within a cloud. Here are some ideas I tried to make work that ultimately failed:

### Lighting from precalculated data

In the main post, you may recall that in optimizing the clouds I used a low res framebuffer (512 x 512) to determine the positions of the clouds. This framebuffer uses the same algorithm as the cloud opacity algorithm, except on reaching a nonzero density writes a 1 to the R channel of the framebuffer (it also returns the multiplier for the draw distance in the A channel). That means there are 16 to 23 unused bits of data we can precompute. My initial concept for using this space was to determine the minimum start and maximum end of a cloud in this low res buffer, then use this data as a rough estimate for the cloud lighting. Each step along a raymarch towards the sun that is within the bounds of the start and end accumulates a number that is fed into the Beer-Powder law. After all, the lighting is just a rough estimate anyway and doesn't have to be completely accurate. As a rough estimate, this would be N texture reads from a 512 x 512 texture instead of 96 * N texture reads from a 256 x 256 texture and would be worth the brain trouble.

As it turned out, after setting up the framework and implementing this approach, the data is not precise enough and the lighting ends up looking very bad (Read: a glitchy mess). It is not feasable to improve this quality, either, without significant overhead in texture size and the time it takes to calculate that texture.

### Lighting the right way, but cheaply

If you read the HZD section on lighting, they advise 6 samples in a random cone towards the sun. My final approach was similar, but I wanted to make it as greedy as possible. The initial code looks something like this:

```
getLight(vec3 r, vec3 sunDirection) {
  //setup vars
  ...
  const float D = 3.5;
  vec3 noise1 = getRandomVec(pos); //takes in a vec3 and returns a random normalized vec3
  vec3 noise2 = getRandomVec(pos * 20.0);
  vec3 n1 = r + D * 1.0 + noise1;
  vec3 n2 = r + D * 2.0 - noise1;
  vec3 n3 = r + D * 3.0 + noise2;
  vec3 n4 = r + D * 4.0 - noise2;
  vec3 n5 = r + D * 5.0 + noise1 * .5 + noise2 * .5;
  vec3 n6 = r + D * 20.0 - noise1 * .5 + noise2 * .5;
  //then get densities for all of these vectors
  //and add them to a sum
  //then scale by a certain amount (linearly)
  //and call beer_powder
}
```

Firstly, this removes banding for free because of the randomness and makes a nice contour. Unfortunately, these 6 calls are still expensive, so my first experiment was in removing as many of them as possible while keeping the clouds looking nice.

It turned out that you can remove all except for the first and keep the shadows looking defined. The only problem with this is that there is not a drastic contour. This proved alright for uncolored clouds, and was what I kept until then. 

Other things I tried to make this algorithm faster was using a cheaper density function for within the lighting. By adjusting the number of FBM octaves, I could reduce the number of texture calls by 2N for each octave removed. Removing one or two octaves kept the shadows looking decent, but the speed was fast enough using the previous improvement that in this iteration I decided to keep all octaves. Another technique outlined in the HZD paper was switching to a cheaper function once the opacity is greater than 0.3. This seemed promising, but all cheaper functions for calculating lighting did not look good enough, and if they did, using them as the initial function was a better option.

### Adding the color to the clouds

The 1 sample approach worked fine for uncolored clouds, but as soon as I added in the lighting (simply mix between a color from the atmosphere and the sun color based on the final lighting value, easily extends the clouds to HDR with some tweaking), the lack of shadows was apparent. After much experimentation, I finally settled on this:

```
vec3 noise(vec3 seed) {
  return vec3(fract(seed.x + seed.z * 11.0174), fract(seed.y * 1.021321321 + seed.x * 0.13214), fract(seed.z * 7.31231 + seed.y * 1.321));
}

float Beer_Powder(float d) { //The Real-time Volumetric Cloudscapes of Horizon: Zero Dawn
    d *= 0.6; // increase this if raining...
    float beer = exp(-d);
    float powder = 1.0 - exp(-d * 2.0);
    return 2.0 * beer * powder;
}

float getLight(vec3 pos, vec3 sunDir, float time, sampler2D noiseSampler) {

  const float sunEquiv = (1. - pow(DBCUTOFF, 1./15.)) / .7;
  sunDir *= sunDir.y < -0.05 ? -1.0 : 1.0;
  sunDir = normalize(sunDir - vec3(0, sunEquiv, 0));

  vec3 n1 = noise(pos);

  vec3 cone1 = 3.5 * D * sunDir + n1;
  float sum = getDensityCheap(pos + cone1, time, noiseSampler) * 3.5;

  vec3 med = 10.0 * D * sunDir - n1;
  sum += getDensityCheap(pos + med, time, noiseSampler) * 1.5;

  vec3 far = 20.0 * D * sunDir;
  sum += getDensityCheap(pos + far, time, noiseSampler) * 2.0;
    
  vec3 farther = 50.0 * D * sunDir;
  sum += getDensityCheap(pos + farther, time, noiseSampler) * 5.0;

  return Beer_Powder(sum);
}
```

and clamped the light to be in \[0.0, 1.0\]. This uses the cheaper density function as described earlier. Some things to note are... 

sunEquiv; this allows the clouds to be lit from below at sunrise and sunset. 

Not using the random vector for the farther samples; it doesn't make a lot of difference.

Multiplying the densities by seemingly random numbers; these are the weights. We value the detailed sample and the farthest sample heavily.

```d *= 0.6```; Made it look nicer.

Anyway, this final result is obviously slower than the previous implementation, but it did look good.
