---
title: AMD FidelityFX Super Resolution 1.0 (FSR) demystified
description: Journal of my experience porting AMD FSR to RetroArch on a GLSL Fragment Shading pass while making it work on OpenGL for the first time.
toc: true
layout: post
comments: true
categories: [graphics, shaders, upscaling]
---

## FSR overview
AMD FidelityFX Super Resolution 1.0 (FSR) has been in the technology/gaming news for a while, but in case someone is learning of it for the first time, it's an open-source **spatial image upscaler** implemented as a "compute shader" (you can think of it as AMD's response to NVIDIA's DLSS, if you're familiar with that technology).

Spatial in this context means that, to upscale an image, it only requires the image itself. It does not need frame history, motion vectors, a depth buffer... really, nothing else. That makes it very simple to integrate into any graphics pipeline.

It's meant to improve performance in 3D games by means of rendering the frame to a lower resolution than the target screen, and then using FSR to upscale it to the target resolution with a very cheap, heavily optimized algorithm that's capable of yielding a full-resolution image that is indistinguishable in quality from the same frame if it was rendered directly to the screen resolution.

Doing this with only spatial data, in a very cheap algorithm, is a challenge, and FSR delivers with amazing quality and performance. That makes it useful for so much more than just increasing performance. It's genuinely a great choice for increasing image quality too, in contexts where rendering natively to a higher resolution is not possible. RetroArch, and emulation in general, fall into that category.

You can find more high-level information and demos on the GPUOpen website: [https://gpuopen.com/fidelityfx-superresolution/](https://gpuopen.com/fidelityfx-superresolution/)

## Low-level overview
FSR is open-source under a permissive license that allows it to be used for any purpose. The source code is available on GitHub: [https://github.com/GPUOpen-Effects/FidelityFX-FSR](https://github.com/GPUOpen-Effects/FidelityFX-FSR)

The repository contains:
- a [PDF document](https://github.com/GPUOpen-Effects/FidelityFX-FSR/raw/master/docs/FidelityFX-FSR-Overview-Integration.pdf) with an overview and basic instructions for integrators;
- a demo example implementation with full source-code on the [`sample/` folder](https://github.com/GPUOpen-Effects/FidelityFX-FSR/tree/master/sample);
- and the lib headers on the [`ffx-fsr/` folder](https://github.com/GPUOpen-Effects/FidelityFX-FSR/tree/master/ffx-fsr).

The headers are extensively documented in comments, and the sample is very small and easy to understand. Everything needed for integration is conveniently available on the repository. Nice!

### `ffx_a.h` ??? Shader Portability lib
This is the first header we must `#include`. It's described in a comment in the file itself as:
>Common central point for high-level shading language and C portability for various shader headers.

This file is like a framework for writing portable shaders. It has a lot of defines and typedefs and function definitions for pretty much anything you may ever imagine you need while writing shader code. This file is much bigger than FSR itself, so I think only a fraction of it is actually being used.

### `ffx_fsr1.h` ??? Spatial Scaling & Extras
This file is FSR proper. It's described in comments as:
>FSR is a collection of algorithms relating to generating a higher resolution image. This specific header focuses on single-image non-temporal image scaling, and related tools.
>
>The core functions are EASU and RCAS:
>- **[EASU] Edge Adaptive Spatial Upsampling:** 1x to 4x area range spatial scaling, clamped adaptive elliptical filter.
>- **[RCAS] Robust Contrast Adaptive Sharpening:** A non-scaling variation on CAS.
>RCAS needs to be applied after EASU as a separate pass.
>
>Optional utility functions are:
>- **[LFGA] Linear Film Grain Applicator:** Tool to apply film grain after scaling.
>- **[SRTM] Simple Reversible Tone-Mapper:** Linear HDR {0 to FP16_MAX} to {0 to 1} and back.
>- **[TEPD] Temporal Energy Preserving Dither:** Temporally energy preserving dithered {0 to 1} linear to gamma 2.0 conversion.

These optional extras are unexpected and very nice! Should use them if needed.

### EASU ??? Edge Adaptive Spatial Upsampling
EASU is the first pass of two required, it is the upscaling pass. You read from the smaller image and output to the bigger buffer. It's as simple as that, really!

For the reading part, you just have to implement three very simple callback functions that FSR will use to gather texture components `r`, `g` and `b` from the source sampler. GLSL f32 example:
```glsl
AF4 FsrEasuRF(AF2 p) { return textureGather(Source, p, 0); }
AF4 FsrEasuGF(AF2 p) { return textureGather(Source, p, 1); }
AF4 FsrEasuBF(AF2 p) { return textureGather(Source, p, 2); }
```

This example is using types from `ffx_a.h`. You don't have to use these if you don't want to. `AF4` is the same as `vec4` and `AF2` is `vec2` in GLSL. `float4` and `float2` in HLSL, respectively.

Now all that's left is setting up some constants with the input and output dimensions, and you can call the `FsrEasu*` function. And that's it, you've implemented FSR upscaling, congrats!

### RCAS ??? Robust Contrast Adaptive Sharpening
After using EASU the image is upscaled but it doesn't look too nice yet. RCAS is the second pass, it's a sharpening filter specialized for use in combination with EASU. Just like the first pass, we need to set up some callback functions:
```glsl
AF4 FsrRcasLoadF(ASU2 p) { return AF4(texelFetch(Source, p, 0)); }
void FsrRcasInputF(inout AF1 r, inout AF1 g, inout AF1 b) {}
```

Yes, the second callback is a no-op. It's for color conversion, but the output of EASU can go straight into RCAS, so in practice it should never be necessary. So, all we needed was a `texelFetch`. Can't be this easy, you think, but it actually is! We can call the `FsrRcas*` function now, and job done.

## A "compute shader", you say
AMD's documentation and sample app does all this on a compute shader. I know nothing of compute shaders, never used it. The only thing I understand is fragment shaders. Also, just learning how to use a compute shader won't help, RetroArch currently doesn't support those, and I'm not going to implement a whole new shader architecture just for FSR.

So, what is FSR doing? Let's dive deeper. I look at the [source-code](https://github.com/GPUOpen-Effects/FidelityFX-FSR/blob/bcffc8171efb80e265991301a49670ed755088dd/ffx-fsr/ffx_fsr1.h#L315) and it does, well, math. Just like normal shader code, only math. It makes sense, it was us who setup the callback functions for input, and it's us who define where the output goes, so there's no magic going on in between those two things, only math. Which means, no external requirements besides the ones we have already provided. This is not a "compute shader", it is just a shader, pretty generic, it runs on anything that can do math. I set it up on a fragment pass, output to `FragColor`, et voil??, I get great upscaling as a result!

But, wait, OpenGL can do math, too. And indeed, although AMD didn't test and don't want to support FSR on OpenGL, it just works unmodified. I was actually developing on OpenGL all along and only noticed when the thing was already running, so really no problems there.

Obviously this is not optimal. For AAA games using this thing to try to extract every bit of performance out of it this probably won't cut it. But that's not the purpose in RetroArch, we are only interested in a nice upscaling filter for retro games. FSR on a fragment shading pass delivers that.

## Conclusion
FSR is merged in RetroArch's main development branch and already on a released version. Community response has been very positive. I think the quality of the results look great. Here's the announcement on [Twitter](https://twitter.com/libretro/status/1433511745641922572) with some good pictures demonstrating it, and the release [announcement for RetroArch 1.9.9](https://www.libretro.com/index.php/retroarch-1-9-9-released/), the first version to include the feature.

AMD deserves credit for a quality release of open-source code. This was my first foray into shaders and graphics programming in general, and I was able to get from nothing to a working implementation in exactly one day of work. For experienced graphics developers, this thing must be so easy it's not even funny. They'll probably try to push it out to an intern, so boring this thing must be.

Also commendable is the work RetroArch's developers have put into its renderer. The only reason I could get this done at all is because all the infrastructure for upscaling was already in place, so my work boiled down to just using it.

If you're asking yourself if you should implement FSR on your graphics renderer, the answer is probably yes, if your renderer already has some support for upscaling. Even if performance hasn't been a problem. Users seem to love FSR, they want it, and it's free, there's no reason not to.

If you don't have upscaling in place, it could get a little more tricky, I guess, but I wouldn't know. If you want to add upscaling, them consider FSR as a valid option. As far as I know, all alternatives involve some kind of temporal dimension. FSR is much simpler, and the results are satisfactory. I think a temporal alternative must be considerably higher quality to justify the extra effort (and all problems with temporal artifacts must be solved).

Being pragmatic, FSR looks good, if it's good enough for your use-case, them it's good enough.

## Links
* Source-code: [https://github.com/libretro/slang-shaders/tree/master/fsr](https://github.com/libretro/slang-shaders/tree/master/fsr)
* Initial merge request: [https://github.com/libretro/slang-shaders/pull/189](https://github.com/libretro/slang-shaders/pull/189)
* Final merge request: [https://github.com/libretro/slang-shaders/pull/192](https://github.com/libretro/slang-shaders/pull/192)

I release this code into the public domain under the terms of the [Unlicense](http://unlicense.org/).
