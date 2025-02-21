# bindbc-openal
This project provides both static and dynamic bindings to API version 1.1 of [the OpenAL library](https://www.openal.org/) and the alternative [OpenAL Soft implementation](https://kcat.strangesoft.net/openal.html). They are `@nogc` and `nothrow` compatible and can be compiled for compatibility with `-betterC`. This package is intended as a replacement of [DerelictAL](https://github.com/DerelictOrg/DerelictAL), which is not compatible with `@nogc`,  `nothrow`, or `-betterC`.

## Usage
By default, `bindbc-openal` is configured to compile as a dynamic binding that is not `-betterC` compatible. The dynamic binding has no link-time dependency on the OpenAL library, so the OpenAL shared library must be manually loaded at runtime. When configured as a static binding, there is a link-time dependency on the OpenAL library&mdash;either the static library or the appropriate file for linking with shared libraries on your platform (see below).

When using DUB to manage your project, the static binding can be enabled via a DUB `subConfiguration` statement in your project's package file. `-betterC` compatibility is also enabled via subconfigurations.

To use OpenAL, add `bindbc-openal` as a dependency to your project's package config file. For example, the following is configured to use OpenAL via a dynamic binding that is not `-betterC` compatible:

__dub.json__
```
dependencies {
    "bindbc-openal": "~>1.0.0",
}
```

__dub.sdl__
```
dependency "bindbc-openal" version="~>1.0.0"
```

### The dynamic binding
The dynamic binding requires no special configuration when using DUB to manage your project. There is no link-time dependency. At runtime, the OpenAL shared library is required to be on the shared library search path of the user's system. On Windows, this is typically handled by distributing the OpenAL Soft DLL with your program or running the OpenAL installer on the end-user's system. On other systems, it usually means the user must install the OpenAL shared library through a package manager.

To load the shared library, you need to call the `loadOpenAL` function. This returns a member of the `ALSupport` enumeration (see [the README for `bindbc.loader`](https://github.com/BindBC/bindbc-loader/blob/master/README.md) for the error handling API):

* `ALSupport.noLibrary` indicates that the library failed to load (it couldn't be found)
* `ALSupport.badLibrary` indicates that one or more symbols in the library failed to load
* `ALSupport.al11` indicates that the library loaded successfully

```d
import bindbc.openal;

/*
This version attempts to load the OpenAL shared library using well-known variations
of the library name for the host system.
*/
ALSupport ret = loadOpenAL();
if(ret != ALSupport.al11) {

    // Handle error. For most use cases, its reasonable to use the error handling API in
    // bindbc-loader to retrieve error messages and then abort. If necessary, it's  possible
    // to determine the root cause via the return value:

    if(ret == ALSupport.noLibrary) {
        // GLFW shared library failed to load
    }
    else if(ALSupport.badLibrary) {
        // One or more symbols failed to load.
    }
}
/*
This version attempts to load the OpenAL library using a user-supplied file name.
Usually, the name and/or path will be platform specific, as in this example
which attempts to load `soft_oal.dll`, the default name of the OpenAL Soft shared
library, from the `libs` subdirectory, relative to the executable, only on Windows.
*/
// version(Windows) loadGLFW("libs/soft_oal.dll")
```

__dub.json__
```
"dependencies": {
    "bindbc-openal": "~>1.0.0"
}
```

__dub.sdl__
```
dependency "bindbc-openal" version="~>1.0.0"
```

## The static binding
__NOTE__: The static binding may cause a crash at runtime when linking dynamically. This is an issue I've been unable to solve yet.

The static binding has a link-time dependency on either the shared or static OpenAL libraries. On Windows, you can link with the static library or, to use the shared library, with the import library. On other systems, you can link with either the static library or directly with the shared library.

_Note that the OpenAL distribution does not contain a static library and the source is not available to build one. OpenAL Soft is open source and can be compiled as a static library._

When linking with the static library, there is no runtime dependency on OpenAL. When linking with the shared library (or the import library on Windows), the runtime dependency is the same as the dynamic binding, the difference being that the shared library is no longer loaded manually&mdash;loading is handled automatically by the system when the program is launched.

Enabling the static binding can be done in two ways.

### Via the compiler's `-version` switch or DUB's `versions` directive
Pass the `BindOpenAL_Static` version to the compiler and link with the appropriate library.

When using the compiler command line or a build system that doesn't support DUB, this is the only option. The `-version=BindOpenAL_Static` option should be passed to the compiler when building your program. All of the required C libraries, as well as the `bindbc-openal` and `bindbc-loader` static libraries, must also be passed to the compiler on the command line or via your build system's configuration.

When using DUB, its `versions` directive is an option. For example, when using the static binding:

__dub.json__
```
"dependencies": {
    "bindbc-openal": "~>1.0.0"
},
"versions": ["BindOpenAL_Static"],
"libs-windows": ["OpenAL32"],
"libs-posix": ["openal"]
```

__dub.sdl__
```
dependency "bindbc-openal" version="~>1.0.0"
versions "BindOpenAL_Static"
libs "OpenAL32" platform="windows"
libs "openal" platform="posix"
```

### Via DUB subconfigurations
Instead of using DUB's `versions` directive, a `subConfiguration` can be used. Enable the `static` subconfiguration for the `bindbc-openal` dependency:

__dub.json__
```
"dependencies": {
    "bindbc-openal": "~>1.0.0"
},
"subConfigurations": {
    "bindbc-openal": "static"
},
"libs-windows": ["OpenAL32"],
"libs-posix": ["openal"]
```

__dub.sdl__
```
dependency "bindbc-openal" version="~>1.0.0"
subConfiguration "bindbc-openal" "static"
libs "OpenAL32" platform="windows"
libs "openal" platform="posix"
```

This has the benefit that it completely excludes from the build any source modules related to the dynamic binding, i.e., they will never be passed to the compiler.

## `betterC` support

`betterC` support is enabled via the `dynamicBC` and `staticBC` subconfigurations for dynamic and static bindings respectively. To enable the static binding with `-betterC` support:

__dub.json__
```
"dependencies": {
    "bindbc-glfw": "~>1.0.0"
},
"subConfigurations": {
    "bindbc-glfw": "staticBC"
},
"libs-windows": ["OpenAL32"],
"libs-posix": ["openal"]
```

__dub.sdl__
```
dependency "bindbc-glfw" version="~>1.0.0"
subConfiguration "bindbc-glfw" "staticBC"
libs "OpenAL32" platform="windows"
libs "openal" platform="posix"
```

When not using DUB to manage your project, first use DUB to compile the BindBC libraries with the `dynamicBC` or `staticBC` configuration, then pass `-betterC` to the compiler when building your project.

## EFX/EAX Extension
Experimental support for the EFX and EAX extensions can be enabled by specifying the `AL_EFX` version when compiling:

__dub.json__
```
"dependencies": {
    "bindbc-openal": "~>1.0.0"
    "versions": ["AL_EFX"],
}
```

__dub.sdl__
```
dependency "bindbc-openal" version="~>1.0.0"
versions "AL_EFX"
```

The following example can be used to test support for EFX/EAX:

```D
bool usingEAXReverb = false;
ALuint loadReverb(ref ReverbProperties r)
{
    ALuint effect;
    alGenEffects(1, &effect);

    if(alGetEnumValue("AL_EFFECT_EAXREVERB") != 0)
    {
        /* EAX Reverb is available. Set the EAX effect type then load the
         * reverb properties. */
        usingEAXReverb = true;
        alEffecti(effect, AL_EFFECT_TYPE, AL_EFFECT_EAXREVERB);

        ALfloat* reflecPan = &r.reflectionsPan[0];
        ALfloat* lateRevPan = &r.lateReverbPan[0];

        alEffectf(effect, AL_EAXREVERB_DENSITY, r.density);
        alEffectf(effect, AL_EAXREVERB_DIFFUSION, r.diffusion);
        alEffectf(effect, AL_EAXREVERB_GAIN, r.gain);
        alEffectf(effect, AL_EAXREVERB_GAINHF, r.gainHF);
        alEffectf(effect, AL_EAXREVERB_GAINLF, r.gainLF);
        alEffectf(effect, AL_EAXREVERB_DECAY_TIME, r.decayTime);
        alEffectf(effect, AL_EAXREVERB_DECAY_HFRATIO, r.decayHFRatio);
        alEffectf(effect, AL_EAXREVERB_DECAY_LFRATIO, r.decayLFRatio);
        alEffectf(effect, AL_EAXREVERB_REFLECTIONS_GAIN, r.reflectionsGain);
        alEffectf(effect, AL_EAXREVERB_REFLECTIONS_DELAY, r.reflectionsDelay);
        alEffectfv(effect,AL_EAXREVERB_REFLECTIONS_PAN, reflecPan);
        alEffectf(effect, AL_EAXREVERB_LATE_REVERB_GAIN, r.lateReverbGain);
        alEffectf(effect, AL_EAXREVERB_LATE_REVERB_DELAY, r.lateReverbDelay);
        alEffectfv(effect,AL_EAXREVERB_LATE_REVERB_PAN, lateRevPan);
        alEffectf(effect, AL_EAXREVERB_ECHO_TIME, r.echoTime);
        alEffectf(effect, AL_EAXREVERB_ECHO_DEPTH, r.echoDepth);
        alEffectf(effect, AL_EAXREVERB_MODULATION_TIME, r.modulationTime);
        alEffectf(effect, AL_EAXREVERB_MODULATION_DEPTH, r.modulationDepth);
        alEffectf(effect, AL_EAXREVERB_AIR_ABSORPTION_GAINHF, r.airAbsorptionGainHF);
        alEffectf(effect, AL_EAXREVERB_HFREFERENCE, r.HFReference);
        alEffectf(effect, AL_EAXREVERB_LFREFERENCE, r.LFReference);
        alEffectf(effect, AL_EAXREVERB_ROOM_ROLLOFF_FACTOR, r.roomRolloffFactor);
        alEffecti(effect, AL_EAXREVERB_DECAY_HFLIMIT, r.decayHFLimit);
    }
    else
    {
        /* No EAX Reverb. Set the standard reverb effect type then load the
         * available reverb properties. */
        alEffecti(effect, AL_EFFECT_TYPE, AL_EFFECT_REVERB);

        alEffectf(effect, AL_REVERB_DENSITY, r.density);
        alEffectf(effect, AL_REVERB_DIFFUSION, r.diffusion);
        alEffectf(effect, AL_REVERB_GAIN, r.gain);
        alEffectf(effect, AL_REVERB_GAINHF, r.gainHF);
        alEffectf(effect, AL_REVERB_DECAY_TIME, r.decayTime);
        alEffectf(effect, AL_REVERB_DECAY_HFRATIO, r.decayHFRatio);
        alEffectf(effect, AL_REVERB_REFLECTIONS_GAIN, r.reflectionsGain);
        alEffectf(effect, AL_REVERB_REFLECTIONS_DELAY, r.reflectionsDelay);
        alEffectf(effect, AL_REVERB_LATE_REVERB_GAIN, r.lateReverbGain);
        alEffectf(effect, AL_REVERB_LATE_REVERB_DELAY, r.lateReverbDelay);
        alEffectf(effect, AL_REVERB_AIR_ABSORPTION_GAINHF, r.airAbsorptionGainHF);
        alEffectf(effect, AL_REVERB_ROOM_ROLLOFF_FACTOR, r.roomRolloffFactor);
        alEffecti(effect, AL_REVERB_DECAY_HFLIMIT, r.decayHFLimit);
    }

    /* Check if an error occured, and clean up if so. */
    int err = alGetError();
    if(err != AL_NO_ERROR)
    {
        if(alIsEffect(effect))
            alDeleteEffects(1, &effect);
        return 0;
    }

    return effect;
}
```

For `ReverbProperties` presets, see [source/bindbc/openal/presets.d](./source/bindbc/openal/presets.d).

To reiterate, EFX/EAX support is experimental. [Please report any issues](https://github.com/BindBC/bindbc-openal/issues) you may encounter with the binding.