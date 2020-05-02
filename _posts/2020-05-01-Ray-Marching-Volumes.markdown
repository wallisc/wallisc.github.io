---
layout: post
title:  "Ray Marching Volumes"
date:   2020-05-01 09:00:07 -0700
categories: rendering 
tags: [volume, shadertoy, raymarching, rendering]
---

![ShaderToy screenshot](/assets/RayMarchingVolumes/finish.gif)

I recently wrote a small ShaderToy that does some simple volumetric rendering. This decided to follow up something that breaks down how the ShaderToy works in 2 parts: part 1 will talk about modelling a something cloud-like using SDFs. Part 2 will go into using raymarching to render the volume. I highly recommend you view the interactive ShaderToy yourself [here][shadertoy-quality]. If you're on a phone or laptop, I suggest viewing the fast version [here][shadertoy-performance]. The goal for this was to be:
1. Real-time-ish
2. Simple
3. When compatible with goals 1 & 2: physically-based


I'll be starting from this scene with some starter code. I'm not going to go deep into the implementation as it's fairly straight forward, but just to give a sense of where we're starting from:
1. Ray trace against the opaque objects. All objects here are primitive objects with simple ray intersections (1 plane and 3 spheres)
2. Phong shading is used to calculate lighting, with the 3 orb lights using a tunable light falloff factor. No shadow rays are needed because the only thing being lit is a plane.

Here's what that looks like:

![ShaderToy screenshot](/assets/RayMarchingVolumes/empty.jpg)

# Part 1: Modelling a volume
So before we can even do any volumetric rendering, we need something to render! I decided to use signed distance functions (SDFs) to model my volume. Why distance fields functions? Because I'm not an artist and they're really great for making organic shapes in a few lines of code. I'm not going to go into signed distance functions because Inigo Quilez has done a really great job of that already. If you're interested, here's a great list of different signed distance functions and modifiers [here][InigoSDF]. And [another][InigoRayMarch] on raymarching those SDFs.

So lets start simple and throw a sphere in there:
![ShaderToy screenshot](/assets/RayMarchingVolumes/sphere.jpg)

Now we'll add an extra sphere and use a smooth union to merge the sphere distance functions together. This is taken straight from Inigo's page, but I'm pasting it here to convince you it's really simple:

{% highlight c %}
// Taken from https://iquilezles.org/www/articles/distfunctions/distfunctions.htm
float sdSmoothUnion( float d1, float d2, float k ) 
{
    float h = clamp( 0.5 + 0.5*(d2-d1)/k, 0.0, 1.0 );
    return mix( d2, d1, h ) - k*h*(1.0-h); 
}
{% endhighlight %}
And with that you get this:
![ShaderToy screenshot](/assets/RayMarchingVolumes/mergedSpheres.jpg)

Well, we know we want some sort of cloudy thing. The really cool thing about SDFs is how easy it is to distort the surface. So lets slap some fractal brownian motion noise ontop using the position to index into the noise function. Again the great Inigo Quilez a really great [article][InigoFBM] on fBM noise if you're interested. 

![ShaderToy screenshot](/assets/RayMarchingVolumes/fusedSpheres.jpg)

Sweet! This thing suddenly looks a lot more interesting with the fBM noise!
Finally we want to give the illusion that the volume is interacting with the ground plane. To do this, I added a plane signed distance just slightly under the actual ground plane and again re-use that smooth union merge with a really aggressive union value (the k parameter). And after that you get this:

![ShaderToy screenshot](/assets/RayMarchingVolumes/mergedGroundPlane.jpg)

And then a final touch is to adjust the fractal brownian motion noise with time so that the volume has a kind of rolling fog look. In motion it looks pretty good!
![ShaderToy screenshot](/assets/RayMarchingVolumes/finalshape.gif)

Woohoo, we have something that looks like a cloudy thing! The code for calculating the SDF is pretty compact too:
{% highlight c %}
float QueryVolumetricDistanceField( in vec3 pos)
{    
    vec3 fbmCoord = (pos + 2.0 * vec3(iTime, 0.0, iTime)) / 1.5f;
    float sdfValue = sdSphere(pos, vec3(-8.0, 2.0 + 20.0 * sin(iTime), -1), 5.6);
    sdfValue = sdSmoothUnion(sdfValue,sdSphere(pos, vec3(8.0, 8.0 + 12.0 * cos(iTime), 3), 5.6), 3.0f);
    sdfValue = sdSmoothUnion(sdfValue, sdSphere(pos, vec3(5.0 * sin(iTime), 3.0, 0), 8.0), 3.0) + 7.0 * fbm_4(fbmCoord / 3.2);
    sdfValue = sdSmoothUnion(sdfValue, sdPlane(pos + vec3(0, 0.4, 0)), 22.0);
    return sdfValue;
}
{% endhighlight %}

But this is just rendering it as an opaque. We want something nice and fluffy! Which leads to part 2...


# Part 2: Volume Rendering
Lets talk about the physics we are simulating here first. A volume represents a large amount of particles within a span of space. And when I say large amount, I mean a LARGE amount. Enough that representing those particles individually is unfeasible today, even for offline rendering. Things like fire, fog, and clouds are great examples of this. In fact, everything is technically volumetric, but for the sake of this, lets just scope this to cloudy-like things. We represent those particles as density values, generally in some 3D grid (or something more sophisticated like OpenVDB). 

When light is going through a volume, a couple things can happen when light hits a particle. It can either get scattered and go off in another direction, or some of the light can get absorbed by the particle and diffused out. In order to stay within the constraints of real-time, we will be doing what's called single scattering. What this means is that we assume that we assume light only scatters once when the light hits a particle and flies towards the camera. This means we won't be able to simulate multiscattering effects such as fog where things in the distance tend to get blurrier. But for our purposes that's okay. Here's a visual of what single scattering looks like with ray marching:

![ShaderToy screenshot](/assets/RayMarchingVolumes/AlgorithmVisual.jpg)

The pseudo code for this looks something like this:
{% highlight c %}
for n steps along the camera ray:
   Calculate how much of the ray was absorbed in the this step
   for m lights:
      for k steps towards the light:
         Calculate how much of the light go absorbed in this step
      Calculate lighting based on how much was absorbed
Blend results on top of opaque objects pass using the absorption value
{% endhighlight %}

So we're talking about something with O(n * m * k) complexity. So buckle up, your GPU's about to get a little toasty.

## Calculating absorption
So first lets just tackle absorbtion of light in a a volume just along the camera ray (i.e. lets not march towards lights..yet). To do that, we need to do 2 things:
1. Ray march inside a volume
2. Calculating absorption/lighting at each step

So lets talk about implementation. To calculate how much light gets absorbed at each point, we use the [Beer-Lambert Law][BeerLamber], which describes light attenuation through a material. The math for this is surprisingly simple:
{% highlight c %}
float BeerLambert(float absorption, float distanceTraveled)
{
    return exp(-absorption * distanceTraveled);
}
{% endhighlight %}

To ray march, the volume. We just take a fixed sized steps along the ray and get the density/absorption at each step. It might not be clear why we need to take fixed steps yet vs something like sphere tracing, but once density isn't uniform within the volume, it'll become clearer why we need fixed steps. Here's what the code for ray marching and accumulating absorption looks like below. Some variable are outside of this code snippet but feel free to refer to the ShaderToy for the complete implementation.
{% highlight c %}
const vec3 volumeAlbedo = vec3(0.8);
const float marchSize = 0.6f;
for(int i = 0; i < MAX_VOLUME_MARCH_STEPS; i++)
{
	volumeDepth += marchSize;
	if(volumeDepth > depth) break;
	
	vec3 position = rayOrigin + volumeDepth*rayDirection;
	bool isInVolume = QueryVolumetricDistanceField(position) < 0.0f;
	if(isInVolume)
	{
		float previousLightFactor = opaqueLightFactor;
		opaqueLightFactor *= BeerLambert(ABSORPTION_COEFFICIENT, marchSize);
		float absorptionFromMarch = previousLightFactor - opaqueLightFactor;
		
		for(int lightIndex = 0; lightIndex < NUM_LIGHTS; lightIndex++)
		{
			float lightVolumeDepth = 0.0f;
			vec3 lightDirection = (GetLight(lightIndex).Position - position);
			float lightDistance = length(lightDirection);
			lightDirection /= lightDistance;
			
			vec3 lightColor = GetLight(lightIndex).LightColor * GetLightAttenuation(lightDistance);  
			volumetricColor += absorptionFromMarch * volumeAlbedo * lightColor;
		
		}
		volumetricColor += absorptionFromMarch * volumeAlbedo * GetAmbientLight();
	}
}
{% endhighlight %}
And this is what that gets you:
![ShaderToy screenshot](/assets/RayMarchingVolumes/AbsorptionOnly.jpg)

It looks like cotton candy! And perhaps for some effects, this actually could be good enough! But what's missing here is self shadowing. Light is reaching all parts of the volume equally. But that's not physically correct, depending on how much volume is between the point being rendered and the light, you will have different amounts of incoming light.

## Self shadowing
At this point, we've actually already done all the hard work. We need to do the same thing we did to calculate absorption along the camera ray, but just along the light ray. The code for calculating how much light reaches each point looks like this:

{% highlight c %}
float GetShadowFactor(in vec3 rayOrigin, in vec3 rayDirection, in float maxT, in int maxSteps, in float marchSize) {
    float t = 0.0f;
    float shadowFactor = 1.0f;
    for(int i = 0; i < maxSteps; i++) {                       
        t += marchSize;
        if(t > maxT) break;

        vec3 position = rayOrigin + t*rayDirection;
        if(QueryVolumetricDistanceField(position) < 0.0) {
            shadowFactor *= BeerLambert(ABSORPTION_COEFFICIENT, marchSize);
        }
    }
    return shadowFactor;
}
{% endhighlight %}

And adding self shadowing gets us this:
![ShaderToy screenshot](/assets/RayMarchingVolumes/SelfShadow.jpg)

## Soften edges
At this point, I was actually feeling pretty good about my volume. I showed this off to our super talented VFX lead at The Coalition James Sharpe, for some feedback. He immediately caught that the edges of the volume looked way to hard. Which is completely true, for things like clouds, they're constantly diffusing with the space around them and so the edges are blending with the empty space arount the volume that should lead to really soft edges. James suggested a great idea of lowering density based how close you are to the edge. Which, because we're working with signed distance functions, is really easy to do! So lets add a function we can use to query density at any point in the volume:

{% highlight c %}
float GetFogDensity(vec3 position)
{   
    float sdfValue = QueryVolumetricDistanceField(position)
    const float maxSDFMultiplier = 1.0;
    bool insideSDF = sdfDistance < 0.0;
    float sdfMultiplier = insideSDF ? min(abs(sdfDistance), maxSDFMultiplier) : 0.0;
    return sdfMultiplier;
}
{% endhighlight %}

And here's how that looks:
![ShaderToy screenshot](/assets/RayMarchingVolumes/softEdges.gif)
## Density Function

And now that we have a density function we're using, it become easy to add some noise into the volume that will give us a little extra detail and fluff to the volume. In this case, I just re-use the fbm function we used for tweaking wit the volume's shape.
{% highlight c %}
float GetFogDensity(vec3 position)
{   
    float sdfValue = QueryVolumetricDistanceField(position)
    const float maxSDFMultiplier = 1.0;
    bool insideSDF = sdfDistance < 0.0;
    float sdfMultiplier = insideSDF ? min(abs(sdfDistance), maxSDFMultiplier) : 0.0;
   return sdfMultiplier * abs(fbm_4(position / 6.0) + 0.5);
}
{% endhighlight %}

And with that, we're here:
![ShaderToy screenshot](/assets/RayMarchingVolumes/NoiseDensity.jpg)

## Opaque Self Shadowing
The volume is looking pretty good at this point! One thing is that there's still some light leaking through the volume. Here we see green light leaking somewhere where the volume definitely should be blocking it:
![ShaderToy screenshot](/assets/RayMarchingVolumes/LightLeak.jpg)
This is because opaque objects are rendered before the volume is rendered, so they don't take into account shadowing that should happen from the volume. This is pretty easy to fix, we have a GetShadowFactor function that we can use to calculate shadowing, so we just need to call this for our opaque object lighting. With that we get this:

![ShaderToy screenshot](/assets/RayMarchingVolumes/shadowOnOpaque.gif)

In addition to getting some really nice colored shadows, it also really helps ground the shadow and sell the volume as a part of the scene. And that's it! There's certainly more I could have done but this felt like it hit the visual bar I wanted for a sample I wanted to keep relatively simple.

# Optimizations
At this point, this blog is running a little long but I'd make some quick mentions of some optimizations:
1. Before marching towards a light, verify that a reasonable amount of light even reachs that point. In my implementation, I look at the luminosity of the light multiplied by the material albedo and make sure it's a big enough value to justify ray marching.
2. If almost all the light has been absorbed by the volume, terminate early, no point marching for no visible gain
3. Defer the lighting of opaque objects until after marching the volume. If all the light was absorbed by the volume, you can skip the lighting of the opaque.

# Conclusions
That's about it! Personally I was surprised how you can get something fairly physically-based with a relatively small amount of code (~500 lines). Thanks for reading, hopefully it was mildly interesting. If you have questions, my DMs are open on Twitter.




Oh and one more thing, here's a fun tweak where I add emissive based on the SDF distance to make an explosion effect. Because we could always use more explosions

![ShaderToy screenshot](/assets/RayMarchingVolumes/explosion.gif)

[BeerLamber]: https://en.wikipedia.org/wiki/Beer–Lambert_law
[InigoRayMarch]: https://www.iquilezles.org/www/articles/raymarchingdf/raymarchingdf.htm
[InigoSDF]: https://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm
[InigoFBM]: https://iquilezles.org/www/articles/fbm/fbm.htm
[shadertoy-quality]: https://www.shadertoy.com/view/tsScDG
[shadertoy-performance]: https://www.shadertoy.com/view/wssBR8
