#Lighting

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
    density = (1.0 - color.w) * density;           
    color += vec4(vec3(light) * density, density);//accumulate color and opacity
}
return color.rgb / (color.w + 0.0001);            //account for opacity and scale accordingly
```

Let's assume TOTAL_STEPS = 32 and LIGHTING_STEPS = 10. At worst this algorithm will take 320 calls of getDensity per pixel per frame. getDensity is a fbm perlin noise generator, so it will do a texture read 8 times with the implementation I was using. This is 2560 texture reads per pixel per frame in the worst case.

Unfortunately, this kind of fidelty doesn't even look good. There is intense banding and aliasing. To fix the aliasing I fudged the start position of the cloudPos and lightPos by a hash of the current time (ticks) and gl_FragCoord.xy.
