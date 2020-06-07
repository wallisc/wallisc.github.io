---
layout: page
title: Personal Projects
permalink: /projects/
---

## TracerBoy
TracerBoy is a DX12 physically-based path tracer leveraging DXR. It support for a variety of materials including subsurface scattering, depth of field, and SVGF-style denoising.

The code is available on [GitHub][github-tracerboy]

![TracerBoy screenshot](/assets/TracerBoy/dragons.png)
> Dragons rendered with a variety of physically-based materials including metal, subsurface-scattering and clear cloat. Dragon model from Delatronic from Benedikt Bitterli's [resources page][Benedikt]


![TracerBoy screenshot](/assets/TracerBoy/bistro.png)
> Render of the bistro scene from the [ORCA library][Orca]


## ShaderToy

# Volumetric Rendering Sample
This is a simple ShaderToy I wrote as an exporation into volumetric rendering. I wrote a [part 1]({% post_url 2020-05-03-Volumetric-Rendering-Part-1 %}) and [part 2]({% post_url 2020-05-03-Volumetric-Rendering-Part-2 %}) post breaking down how it works.

ShaderToy available [here][shadertoy-quality]. For phone or laptops, I'd recommend the performant version [here][shadertoy-quality]

![ShaderToy screenshot](/assets/RayMarchingVolumes/finish.gif)

# Moana Water Shader
An interactive water shader based on Moana. The shader does 1 primary ray and 2 levels of refraction (with each level also tracing a reflection ray). The water is ray marched as an sdf while the opaque objects are all spheres and a ground plane that are ray traced for performance reasons. Parallax occlusion mapping is used on the sand to give a sense of real geometry as well as some small material tricks to simulate wet sand close to the water.

ShaderToy available [here][shadertoy-moana]
![ShaderToy screenshot](/assets/Moana/water.gif)

# Winter Cabin
The scene is modelled entirely using SDFs. Screen-space blurring is done on the snow to simulate subsurface scattering and in the distance for DOF. I added some manual point light placement to fake area lighting from the windows.

ShaderToy available [here][shadertoy-cabin]
![ShaderToy screenshot](/assets/ShaderToy/cabin.gif)


[Orca]: https://developer.nvidia.com/orca
[Benedikt]: https://benedikt-bitterli.me/resources/
[github-tracerboy]: https://github.com/wallisc/TracerBoy

[shadertoy-moana]: https://www.shadertoy.com/view/wlsyzH
[shadertoy-cabin]: https://www.shadertoy.com/view/3lt3W7
[shadertoy-quality]: https://www.shadertoy.com/view/tsScDG
[shadertoy-performance]: https://www.shadertoy.com/view/wssBR8
