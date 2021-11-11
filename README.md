# shadertrixx

CNLohr's repo for his Unity assets and other shader notes surrounding VRChat.  This largely contains stuff made by other people but I have kind of collected.

## The most important trick

```glsl
#define glsl_mod(x,y) (((x)-(y)*floor((x)/(y)))) 
```

This makes a well behaved `mod` function that rounds down even when negative.  For instance, `glsl_mod(-0.3, 1)` is 0.7.

Also note: Using this trick in some situations actually produces smaller code than regular mod!!

Thanks, @d4rkpl4y3r - this originally actually comes from an epic bgolus forum post: https://forum.unity.com/threads/translating-a-glsl-shader-noise-algorithm-to-hlsl-cg.485750/

## Struggling with shader type mismatches?

You can put this at the top of your shader to alert you to when you forgot a `float3` and wrote `float` by accident.

```glsl
#pragma warning (default : 3206) // implicit truncation
```

## My recommended packages and order:

1. Import VRC SDK Worlds: https://vrchat.com/home/download
2. Import Udon Sharp: https://github.com/MerlinVR/UdonSharp/releases
3. Import CyanEmu: https://github.com/CyanLaser/CyanEmu/releases
4. Import VRWorld Toolkit: https://github.com/oneVR/VRWorldToolkit/releases
5. Import AudioLink: https://github.com/llealloo/vrc-udon-audio-link/releases

To try:
Thing that detects broken refernces to Udon Scripts https://github.com/esnya/EsnyaUnityTools/releases  (This is probably deprecated because U# should do everything this does automatically)

## When opening worlds from git using the .gitignore file from here

1. Open project in Unity Hub for correct version of Unity.
3. Import VRC SDK
2. **Configure your player settings under editor preprocessor to include UDON**
4. Import UdonSharp
5. Import AudioLink
6. Import Esnya Tools
7. Import VRC World Toolkit
8. Run Window->UdonSharp->Refresh All UdonSharp Assets
9. Repeat 6 til no new assets.
10. Close and reopen Unity
11. Open Scene
12. EsnyaTools -> Repair Udon
13. VRWorldToolkit -> World Debugger 
14. Fix all errors.

## Basics of shader coding:

* Unity shader fundamentals: https://docs.unity3d.com/Manual/SL-VertexFragmentShaderExamples.html
* From @Orels: quick reference for HLSL: https://developer.download.nvidia.com/cg/index_stdlib.html
* List of intrinsic functions: https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-intrinsic-functions
* The common built-in header for unity-provided functions: https://github.com/TwoTailsGames/Unity-Built-in-Shaders/blob/master/CGIncludes/UnityCG.cginc
* Unity's built-in shader variables: https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html
* Defining parameters for your shader/material: https://docs.unity3d.com/Manual/SL-Properties.html
* Unity surface shader examples: https://docs.unity3d.com/Manual/SL-SurfaceShaderExamples.html

Unity's overview page: https://docs.unity3d.com/2019.4/Documentation/Manual/shader-writing.html

## Additional tricks

From @Lyuma
 * [flatten] (UNITY_FLATTEN macro) to force both cases of an if statement or
 * force a branch with [branch] (UNITY_BRANCH macro);
 * force loop to unroll with [unroll] (UNITY_UNROLL) or
 * force a loop with [loop] (UNITY_LOOP)
 * there's also [call] for if or switch statements I think, not sure exactly how it works.

### Lyuma Beautiful Retro Pixels Technique

If you want to use pixels but make the boundaries between the pixels be less ugly, use this:
```glsl
float2 coord = i.tex.xy * _MainTex_TexelSize.zw;
float2 fr = frac(coord + 0.5);
float2 fw = max(abs(ddx(coord)), abs(ddy(coord)));
i.tex.xy += (saturate((fr-(1-fw)*0.5)/fw) - fr) * _MainTex_TexelSize.xy;
```

### Scruffy Ruffle's utilitiy functions

```glsl
bool isVR() {
    // USING_STEREO_MATRICES
    #if UNITY_SINGLE_PASS_STEREO
        return true;
    #else
        return false;
    #endif
}

bool isVRHandCamera() {
    return !isVR() && abs(UNITY_MATRIX_V[0].y) > 0.0000005;
}

bool isDesktop() {
    return !isVR() && abs(UNITY_MATRIX_V[0].y) < 0.0000005;
}

bool isVRHandCameraPreview() {
    return isVRHandCamera() && _ScreenParams.y == 720;
}

bool isVRHandCameraPicture() {
    return isVRHandCamera() && _ScreenParams.y == 1080;
}

bool isPanorama() {
    // Crude method
    // FOV=90=camproj=[1][1]
    return unity_CameraProjection[1][1] == 1 && _ScreenParams.x == 1075 && _ScreenParams.y == 1025;
}
```

### Merlin's IsMirror()

```glsl
bool IsInMirror()
{
    return unity_CameraProjection[2][0] != 0.f || unity_CameraProjection[2][1] != 0.f;
}
```

### Not-shaders

From @lox9973 This flowchart of how mono behaviors are executed and in what order: https://docs.unity3d.com/uploads/Main/monobehaviour_flowchart.svg

## tanoise

Very efficient noise based on Toocanzs noise. https://github.com/cnlohr/shadertrixx/blob/main/Assets/tanoise/README.md

## scrn_aurora

tanoise-modified aurora, originally written by nimitz, modified further by scrn.  https://github.com/cnlohr/shadertrixx/tree/main/Assets/scrn_aurora

## Defining Avatar Scale

The "magic ratio" is `view_y = head_to_wrist / 0.4537` (in t-pose) all unitless.

"It's mentioned many places that armspan is the defining scale, but that comment is more specific (armspan is 2 * head_to_wrist, and the ratio to height)" - Ben


## Multiply vector-by-quaterion

From @axlecrusher :
```glsl
float3 vector_quat_rotate( float3 v, float4 q )
{ 
	return v + 2.0 * cross(q.xyz, cross(q.xyz, v) + q.w * v);
}

float3 vector_quat_unrotate( float3 v, float4 q )
{ 
	return v + 2.0 * cross(q.xyz, cross(q.xyz, v) - q.w * v);
}
```

Create a 2x2 rotation, can be applied to a 3-vector by saying vector.xz or other swizzle.  Not sure where this one came from
```glsl
		fixed2x2 mm2( fixed th ) // farbrice neyret magic number rotate 2x2
		{
			fixed2 a = sin(fixed2(1.5707963, 0) + th);
			return fixed2x2(a, -a.y, a.x);
		}
```

## Using depth cameras on avatars.

If an avatar has a grab pass, and you're using a depth camera, you may fill people's logs with this:

```
Warning    -  RenderTexture.Create: Depth|ShadowMap RenderTexture requested without a depth buffer. Changing to a 16 bit depth buffer.
```

A way around this is to create a junk R8 texture with no depth buffer `rtDepthThrowawayColor`, and your normal depth buffer, `rtBotDepth` and frankenbuffer it into a camera.  NOTE: This will break camrea depth, so be sure to call `SetTargetBuffers()` in the order you want the camreas to evaluate.

```cs
	CamDepthBottom.SetTargetBuffers( rtDepthThrowawayColor.colorBuffer, rtBotDepth.depthBuffer );
```

## Is your UV within the unit square?

```glsl
any(i.uvs < 0 || i.uvs > 1)
```
```
   0: lt r0.xy, v0.xyxx, l(0.000000, 0.000000, 0.000000, 0.000000)
   1: lt r0.zw, l(0.000000, 0.000000, 1.000000, 1.000000), v0.xxxy
   2: or r0.xy, r0.zwzz, r0.xyxx
   3: or r0.x, r0.y, r0.x
   4: sample_indexable(texture2d)(float,float,float,float) r0.y, v0.xyxx, t0.xwyz, s0
   5: movc o0.xyzw, r0.xxxx, l(0,0,0,0), r0.yyyy
   6: ret 
```
(From @d4rkpl4y3r)

And the much less readable and more ambiguous on edge conditions version:
```glsl
any(abs(i.uvs-.5)>.5)
```
```
   0: add r0.xy, v0.xyxx, l(-0.500000, -0.500000, 0.000000, 0.000000)
   1: lt r0.xy, l(0.500000, 0.500000, 0.000000, 0.000000), |r0.xyxx|
   2: or r0.x, r0.y, r0.x
   3: sample_indexable(texture2d)(float,float,float,float) r0.y, v0.xyxx, t0.xwyz, s0
   4: movc o0.xyzw, r0.xxxx, l(0,0,0,0), r0.yyyy
   5: ret 
```
(From @scruffyruffles)

## Write Unity Texture Assets from C

https://gist.github.com/cnlohr/c88980e560ecb403cae6c6525b05ab2f

## Shadowcasting

Make sure to add a shadowcast to your shader, otherwise shadows will look super weird on you.  Just paste this bad boy in your subshader.

```glsl
		// shadow caster rendering pass, implemented manually
		// using macros from UnityCG.cginc
		Pass
		{
			Tags {"LightMode"="ShadowCaster"}
			Cull Off
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			#pragma multi_compile_shadowcaster
			#pragma multi_compile_instancing
			#include "UnityCG.cginc"

			struct v2f { 
				V2F_SHADOW_CASTER;
				float4 uv : TEXCOORD0;
			};

			v2f vert(appdata_base v)
			{
				v2f o;
				UNITY_SETUP_INSTANCE_ID(v);
				TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
				o.uv = v.texcoord;
				return o;
			}

			float4 frag(v2f i) : SV_Target
			{
				SHADOW_CASTER_FRAGMENT(i)
			}
			ENDCG
		}
```

## Instancing

To enable instancing, you must have in your shader:
 * `#pragma multi_compile_instancing` in all all passes.
 * Optionally
```glsl
        UNITY_INSTANCING_BUFFER_START(Props)
            // put more per-instance properties here
        UNITY_INSTANCING_BUFFER_END(Props)
```
 * An example thing you could put there, in the middle is:
```glsl
	UNITY_DEFINE_INSTANCED_PROP( float4, _InstanceID)
```
 * I've found it to be super janky to try to access the variable in the fragment/surf shader, but it does seem to work in the vertex shader.
 * In your vertex shader:
```glsl
	UNITY_SETUP_INSTANCE_ID(v);
```
 * Access variables with `UNITY_ACCESS_INSTANCED_PROP(Props, _InstanceID).x;`
 * Access which instance it is in the list with `unity_InstanceID` - note this will change from frame to frame.
 * To change the value see following example:
```cs

using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class MaterialPropertyInstanceIDIncrementer : UdonSharpBehaviour
{
    void Start()
    {
		MaterialPropertyBlock block;
		MeshRenderer mr;
        int id = GameObject.Find( "BrokeredUpdateManager" ).GetComponent<BrokeredUpdateManager>().GetIncrementingID();
		block = new MaterialPropertyBlock();
		mr = GetComponent<MeshRenderer>();
		//mr.GetPropertyBlock(block);  //Not sure if this is needed
		block.SetVector( "_InstanceID", new Vector4( id, 0, 0, 0 ) );
		mr.SetPropertyBlock(block);
    }
}
```

## Grabpasses

You can add a grabpass tag outside of any pass (this happens in the SubShader tag).  You should only use `_GrabTexture` on the transparent queue as to not mess with other shaders that use the `_GrabTexture`

```
        Tags { "RenderType"="Transparent" "Queue"="Transparent" }
        LOD 100

        GrabPass
        {
            "_GrabTexture"
        }

```

You should use the `_GrabTexture` name so that it only has to get executed once instead of once for every material.

You can then index into it as a sampler2D.


```glsl
            sampler2D _GrabTexture;
```

```glsl
float2 grabuv = i.uv;
#if !UNITY_UV_STARTS_AT_TOP
grabuv.y = 1 - grabuv.y;
#endif
fixed4 col = tex2D(_GrabTexture, grabuv);
```

Or, if you want to grab into it from its place on the screen, like to do a screen-space effect, you can do this:

in Vertex shader:
```glsl
	o.grabposs = ComputeGrabScreenPos( o.vertex );
```
in Fragment shader:
```glsl
	col = tex2Dproj(_GrabTexture, i.grabposs );
```

editor's note: I spent a long time trying to find a good way to do this exclusively from the fragment shader, and I did not find one.

## Are you in a mirror?

Thanks, @Lyuma

```glsl
bool IsInMirror()
{
    return unity_CameraProjection[2][0] != 0.f || unity_CameraProjection[2][1] != 0.f;
}
```

## Default Texture Parameters

Default values available for texture properties:
```
red
gray
grey
linearGray
linearGrey
grayscaleRamp
greyscaleRamp
bump
blackCube
lightmap
unity_Lightmap
unity_LightmapInd
unity_ShadowMask
unity_DynamicLightmap
unity_DynamicDirectionality
unity_DynamicNormal
unity_DitherMask
_DitherMaskLOD
_DitherMaskLOD2D
unity_RandomRotation16
unity_NHxRoughness
unity_SpecCube0
unity_SpecCube1
```

And of course, "white" and "black"

IE `_MainTex ("Texture", 2D) = "unity_DynamicLightmap" {}`

Thanks, @Pema

## Depth Textures

If you define a sampler2D the following way, you can read the per-pixel depth.
```glsl
sampler2D _CameraDepthTexture;
```

```glsl
// Compute projective scaling factor...
float perspectiveDivide = 1.0f / i.screenPosition.w;

// Scale our view ray to unit depth.
float3 direction = i.worldDirection * perspectiveDivide;

// Calculate our UV within the screen (for reading depth buffer)
float2 screenUV = (i.screenPosition.xy * perspectiveDivide) * 0.5f + 0.5f;

// No idea
screenUV.y = 1 - screenUV.y; 

// VR stereo support
screenUV = UnityStereoTransformScreenSpaceTex(screenUV);

// Read depth, linearizing into worldspace units.    
float depth = LinearEyeDepth(UNITY_SAMPLE_DEPTH(tex2D(_CameraDepthTexture, screenUV)));

float3 worldspace = direction * depth + _WorldSpaceCameraPos;
```

## Reference Camera

Don't forget to drag your Main Camera into "Reference Camera" property on your VRCWorld.  Not having a reference camera will sometimes set zNear to be too far or other problems.  So having a reference camera is highly advised.

## Making Unity UI much faster

Go under Edit->Project Settings...->Editor->Sprite Packer->Mode->Always Enabled

## VRC Layers

http://vrchat.wikidot.com/worlds:layers

Want to do a camera compute loop?  Shove everything on the "Compute" layer and put it somewhere a long way away.

## Need to iterate on Udon Sharp faster?

Edit -> Preferences -> General -> Auto Refresh, uncheck.

Whenever you do need Unity to reload, ctrl+r

## Iterating through every instance of a behavior or udonsharpbehavior attached to all objects in a scene.

For instance getting all instances of the BrokeredBlockIDer behavior.
```cs
foreach( UnityEngine.GameObject go in GameObject.FindObjectsOfType(typeof(GameObject)) as UnityEngine.GameObject[] )
{
	foreach( BrokeredBlockIDer b in go.GetUdonSharpComponentsInChildren<BrokeredBlockIDer>() )
	{
		b.UpdateProxy();
		b.defaultBlockID = (int)Random.Range( 0, 182.99f );
		ct++;
		b._SetBlockID( b.defaultBlockID );
		b._UpdateID();
		b.ApplyProxyModifications();
	}
}
```

`GetUdonSharpComponentsInChildren` is the magic thing.  **PLEASE NOTE** You must use `UpdateProxy()` before reading from and `ApplyProxyModifications()` when done.


## MRT

This demo is not in this project, but, I wanted to include notes on how to do multiple rendertextures.

1) Set up cameras pointed at whatever you want to compute.
2) Put the things on a layer which is not heavily used by anything else.
3) Make sure to cull that layer on all lights.
4) Use `SetReplacementShader` - this will prevent a mountain of `OnRenderObject()` callbacks.
5) Cull in camrea based on that layer.
6) Put camera on default layer.
7) Don't care about depth because when using MRTs, you want to avoid letting unity figure out the render order, unless you really don't care.
8) I find putting camera calcs on `UiMenu` to work best.

OPTION 1: Cameras ignore their depth and render order when you do this.  Instead they will execute in the order you call SetTargetBuffers on them.

NOTE: OPTION 2: TEST IT WITHOUT EXPLICIT ORDERING (manually executing .Render) FIRST AS THIS WILL SLOW THINGS DOWN  You will need to explicitly execute the order you want for all the cameras.   You can only do this in `Update` or `LateUpdate`, i.e.

```cs
		CamCalcA.enabled = false;
		CamCalcA.SetReplacementShader( <shader>, "" );
		RenderBuffer[] renderBuffersA = new RenderBuffer[] { rtPositionA.colorBuffer, rtVelocityA.colorBuffer };
		CamCalcA.SetTargetBuffers(renderBuffersA, rtPositionA.depthBuffer);
..

		CamCalcA.Render()
		CamCalcB.Render()
```


## Making camera computation loops performant

 * Put the camera, the culling mask and the object you're looking at on a unique layer from your scene in general.  Find all lights, and cull off that layer.  If a camera is on a layer that is lit by a light, things get slow.
 * Make sure your bounding box for your geometry you are computing on doesn't leak into other cameras.
 * Use the following script on your camera: `<camera>.SetReplacementShader( <shader>, "");` where `<camera>` is your camera and `<shader>` is the shader you are using on the object you care about.  As a warning, using a tag here will slow things back down.  This prevents a ton of callbacks like `OnRenderObject()` in complicated scenes.
 * Doing these things should make your camera passes take sub-200us in most situations.



## Additional Links

These are links other people have given me, these links are surrounding U#.

 * https://github.com/jetdog8808/Jetdogs-Prefabs-Udon
 * https://github.com/Xytabich/UNet
 * https://github.com/FurryMLan/VRChatUdonSharp
 * https://github.com/Guribo/BetterAudio
 * https://github.com/squiddingme/UdonTether
 * https://github.com/cherryleafroad/VRChat_Keypad
 * https://github.com/aiya000/VRChat-Flya
 * https://github.com/MerlinVR/USharpVideo
 * https://github.com/Reimajo/EstrelElevatorEmulator/tree/master/ConvertedForUdon

Lit vertex shader
 * https://github.com/Xiexe/Unity-Lit-Shader-Templates

Interesting looking mesh tool (Still need to use)
 * https://github.com/lyuma/LyumaShader/blob/master/LyumaShader/Editor/LyumaMeshTools.cs
 
Basic global profiling scripts for Udon:
 * https://gist.github.com/MerlinVR/2da80b29361588ddb556fd8d3f3f47b5

This explaination of how the fallback system works (Linked by GenesisAria)
 * https://pastebin.com/92gwQqCM

Making procedural things like grids that behave correctly for going off in the distance.
 * https://www.iquilezles.org/www/articles/filterableprocedurals/filterableprocedurals.htm
 * https://www.iquilezles.org/www/articles/bandlimiting/bandlimiting.htm

 Convert detp function:
 ```c
     //Convert to Corrected LinearEyeDepth by DJ Lukis
     float depth = CorrectedLinearEyeDepth(sceneZ, direction.w);

     //Convert from Corrected Linear Eye Depth to Raw Depth 
     //Credit: https://www.cyanilux.com/tutorials/depth/#eye-depth

     depth = (1.0 - (depth * _ZBufferParams.w)) / (depth * _ZBufferParams.z);
     //Convert to Linear01Depth
     depth = Linear01Depth(depth);
```


This SLERP function, found by ACiiL,
```c
        ////============================================================
        //// blend between two directions by %
        //// https://www.shadertoy.com/view/4sV3zt
        //// https://keithmaggio.wordpress.com/2011/02/15/math-magician-lerp-slerp-and-nlerp/
        float3 slerp(float3 start, float3 end, float percent)
        {
            float d     = dot(start, end);
            d           = clamp(d, -1.0, 1.0);
            float theta = acos(d)*percent;
            float3 RelativeVec  = normalize(end - start*d);
            return      ((start*cos(theta)) + (RelativeVec*sin(theta)));
        }
```

Thanks, error.mdl for telling me how to disable batching.  This fixes issues where shaders need to get access to their local coordinates.
```
            Tags {  "DisableBatching"="true"}
```

## Notes on grabpass avatar->map data exfiltration

@d4rkpl4y3r notes that you can use queue < 2000 and zwrite off to exfiltrate data without horrible visual artifacts.  You can also use points to do the export instead of being limited to quads by exporting points from a geometry shader on the avatar with the following function:

```glsl
float4 pixelToClipPos(float2 pixelPos)
{
    float4 pos = float4((pixelPos + .5) / _ScreenParams.xy, 0.5, 1);
    pos.xy = pos.xy * 2 - 1;
    pos.y = -pos.y;
    return pos;
}
```

(TODO: Expand upon this with actual demo)


## HALP The Unity compiler is emitting really bizarre assembly code.

Eh, just try using a different shader model, add a 
```glsl
#pragma  target 5.0
```
in your code or something.  Heck 5.0's been supported since the GeForce 400 in 2010.

## Udon events.

Re: `FixedUpdate`, `Update`, `LateUpdate`

From Merlin: https://docs.unity3d.com/ScriptReference/MonoBehaviour.html most of the events under Messages, with some exceptions like awake and things that don't make sense like the Unity networking related ones
you can look at what events UdonBehaviour.cs registers to see if they are actually there on Udon's side

## UdonSharp Get All Node Names

Edit->Project Settings->Player Settings->Configuration->Scripting Define Values

Add `UDONSHARP_DEBUG`

Then, reload.

Then, Window->Udon Sharp->Node Definition Grabber

Press the button. Your clipboard now contains a present.

Once you've done this, go back and remove `UDONSHARP_DEBUG`

## Getting big buffers

From @lox9973

BIG WARNING: After a lot of testing, we've found that this is slower than reading from a texture if doing intensive reads.  If you need to read from like 100 of these in a shader, probably best to move it into a texture first.

```c
cbuffer SampleBuffer {
    float _Samples[1023*4] : packoffset(c0);  
    float _Samples0[1023] : packoffset(c0);
    float _Samples1[1023] : packoffset(c1023);
    float _Samples2[1023] : packoffset(c2046);
    float _Samples3[1023] : packoffset(c3069);
};
float frag(float2 texcoord : TEXCOORD0) : SV_Target {
    uint k = floor(texcoord.x * _CustomRenderTextureInfo.x);
    float sum = 0;
    for(uint i=k; i<4092; i++)
        sum += _Samples[i] * _Samples[i-k];
    if(texcoord.x < 0)
        sum = _Samples0[0] + _Samples1[0] + _Samples2[0] + _Samples3[0]; // slick
    return sum;
}
```
and
```c
void Update() {
    source.GetOutputData(samples, 0);
    System.Array.Copy(samples, 4096-1023*4, samples0, 0, 1023);
    System.Array.Copy(samples, 4096-1023*3, samples1, 0, 1023);
    System.Array.Copy(samples, 4096-1023*2, samples2, 0, 1023);
    System.Array.Copy(samples, 4096-1023*1, samples3, 0, 1023);
    target.SetFloatArray("_Samples0", samples0);
    target.SetFloatArray("_Samples1", samples1);
    target.SetFloatArray("_Samples2", samples2);
    target.SetFloatArray("_Samples3", samples3);
}
```
https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-constants

CBuffers:
```
Properties {
...

    _Spread00 ("Spine",     Vector) = (40, 40, 40, 1)
    _Spread01 ("Head",        Vector) = (40, 40, 80, 1)
    ...
    _Spread50 ("IndexProximal",    Vector) = (45, 20,  9, 1)
    _Spread51 ("IndexDistal",    Vector) = (45,  9,  9, 1)

    _Finger00 ("LeftThumb",        Vector) = (0, 0, 0, 0)
    _Finger01 ("RightThumb",    Vector) = (0, 0, 0, 0)
    ...
    _Finger40 ("LeftLittle",    Vector) = (0, 0, 0, 0)
    _Finger41 ("RightLittle",    Vector) = (0, 0, 0, 0)
}

CGPROGRAM
...
cbuffer SpreadBuffer {
    float4 _Spread[6][2] : packoffset(c0);  
    float4 _Spread00 : packoffset(c0);
    float4 _Spread01 : packoffset(c1);
    ...
    float4 _Spread50 : packoffset(c10);
    float4 _Spread51 : packoffset(c11);
};
cbuffer FingerBuffer {
    float4 _Finger[10] : packoffset(c0);  
    float4 _Finger00 : packoffset(c0);
    ...
    float4 _Finger40 : packoffset(c8);
    float4 _Finger41 : packoffset(c9);
}
ENDCG
```

## Keywords.

(1) DO NOT INCLUDE `[Toggle]`!!
INSTEAD, use `[ToggleUI]`

(2) If you do want to use keywords, you can from this list:  

(3) To use keywords, do the following: https://pastebin.com/83fQvZ3n

In your properties block: 
```
[Toggle(_ALPHAMODULATE_ON)] _ALPHAMODULATE_ON ( "Some Feature", int ) = 0
```

In your shader block, add:
```
#pragma shader_feature _ALPHAMODULATE_ON
```
or
```
#pragma multi_compile __ _ALPHAMODULATE_ON
```

And in your shader
```
#if _ALPHAMODULATE_ON
 // Do something
#endif
```

## VRChat "Build & Test" Overrides

You can insert additional parameters into VRC for "Build & Test" with the following (compiled TCC build of code included.) For instance, this one adds the `--fps=0` command-line parameter.
```c
#include <stdio.h>
#include <stdlib.h>

int main( int argc, char ** argv )
{
    char cts[8192];
    char * ctsp = cts;
    int i;
    ctsp += sprintf( ctsp, "vrchat.exe --fps=0" );
    for( i = 1; i < argc; i++ )
    {
        ctsp += sprintf( ctsp, " \"%s\"", argv[i] );
    }
    printf( "Launching: %s\n", cts );
    system( cts );
}
```
Command-line to compile application:
```
c:\tcc\tcc.exe vrc-uncapped.c
```

Then, in VRC SDK Settings, set the path to the VRC Exe to be vrc-uncapped.exe

## 3D CC0 / Public Domain Resources (Compatible with MIT-licensed projects)

 * https://quaternius.com/
 * https://www.kenney.nl/assets

## Making audio play

Thanks, @lox9973 for informing me of this: https://gitlab.com/-/snippets/2115172

# ATTIC

(Stuff below here may no longer be valid)


## CRT Perf Testing (Unity 2018)
Test number results were performed on a laptop RTX2070.

**BIG NOTE**: CNLohr has observed lots of oddities and breaking things when you use the "period" functionality in CRTs.  I strongly recommend setting "period" to 0.

NSIGHT Tests were not performed in VR.

SINGLE CRT TESTS

Swapping buffers between passes:

 * SAME CRT: 1024x1024 RGBAF Running a CRT with with 25x .2x.2 update zones, double buffering every time yields a total run time of 
   * 2.25ms in-editor. 2.5ms in-game.
   * Each pass executed over a 10us window.
   * There was 100us between executions.
 * Same as above, except for 128x128.
   * 330us in-editor. 250us in-game.
   * Each pass executed over a 6us window.
   * There was 6us between executions.
 * Same as above, except for 20x20.
   * 340us in-editor. 270us in-game.
   * Each pass executed over a 6us window.
   * There was 6us between executions.
   
Not swapping buffers between passes:

 * SAME CRT: 1024x1024 RGBAF Running a CRT with with 25x .2x.2 update zones, double buffering only on last pass yields a total run time of 
   * 230-280us in-editor. 185us in-game.
   * Each pass executed over a 3.5us window.
   * There is no time between executions.
   * There was a 100us lag on the very last update.
 * Same as above, except for 128x128.
   * 63us in-editor. 22us (+/- a fair bit) in-game.
   * Each pass executed over a between 400ns and 1us window.
   * There are random lags surrounding this in game, but the lags are all tiny.
   
   
With chained CRTs, **but** using the same shader.  The mateials were set up so that each passing CRT.  All tests run 25 separate CRTs, using the last one.
 * 1024x1024 RGBAF running a CRT, but only .2x.2 of the update zone. (So it's a fair test).
   * ~80us in-editor, 140us in-game.
   * ~6.5us between passes.
   * First material uses a fixed material.
   * OBSERVATION: This looks cheaty, as it's all the same rendertarget.
 * Same, except forces the chain to be circular.
   * Same in-game perf.
 * Same, except verifying that each step is not stepped upon.
   * Unity a little slower (110us), 160us in-game.

 * Forced different rendertarget sizes, pinging between 1024x1024 and 512x512.
   * ~85us in-editor.
   * 120us in-game.

 * Forcefully inserting one double-buffered frame, to allow data tobe fed back in on itself
    * 190us in-editor
	* 250us in-game.
	* The frame with the double buffer incurs a huge pipeline hit of ~110us.

### Cameras
 * Created 25 cameras, pointed at 25 quads, each on an invisible layer.
 * No depth buffer, no clear.
 * Quad was only .2x.2
 * Basically same shader as chain CRT.
 * 1024x1024 RGBA32F
 
 * In-Editor 234us per camera. 5.8ms total.
 * In-Game 300us per camera. 7.8ms total.

Trying 128x128
 * In-Editor: 35us per camera. around 600us total.
 * In-game: ~30us per camera.  But, weird timing. Has stalls. Takes ~2.1ms total.

### Conclusions:
 * You can work with large CRTs that chain with virtually no overhead.
 * Small (say 128x128) buffers can double buffer almost instantly.
 * Larger buffers, for instance 1024x1024 RGBA32F take ~110us to double-buffer.
 * No penalty is paid by chaining CRTs target different texture sizes.
 * Note that all tests were performed with the same shader for all CRTs.
 * Cameras work surprisingly well for smaller textures and really poorly for big textures.

## FYI
 * For maximum platform support, make all edges of your RenderTexture divisible by 16.  (Note: ShaderFes, and VRSS)

## General notes for working from git (NOTES ONLY)

 * Use .gitignore from cnballpit-vrc
 * Import the VRC SDK
 * Import the U# SDK
 * Import VRWorldToolkit
(Reopen project)
 * Run "Refresh all UdonSharp Assets"

## General 2019 Beta Info:

1. React to this post: https://discord.com/channels/189511567539306508/449348023147823124/500437451085578262
2. Read this https://discord.com/channels/189511567539306508/503009489486872583/865421330698731541
4. Download this: https://files.vrchat.cloud/sdk/U2019-VRCSDK3-WORLD-2021.07.15.13.46_Public.unitypackage
5. Download & Install Unity Hub: https://unity3d.com/get-unity/download
7. Install Unity 2019.4.28f1.
8. Backup your project.
9. Follow this guide: https://docs.vrchat.com/v2021.3.2/docs/migrating-from-2018-lts-to-2019-lts
Basically:
1. Open the project to an empty scene.
2. Import the BETA SDK - if you ever import the not beta SDK you will likely have to start over.
3. Import Udon sharp aftert the beta SDK.
4. Import CyanEmu.

Import anything else you need.

Then open your scene.


NOTE: If you are going from a fresh git tree of a project, you should open a blank scene, import the new beta SDK and all your modules then close unity and reopen your scene.

--> TODO --> Include in PR.
```cs
		[MenuItem("Window/Udon Sharp/Refresh All UdonSharp Assets")]
		static public void UdonSharpCheckAbsent()
		{
			Debug.Log( "Checking Absent" );

			string[] udonSharpDataAssets = AssetDatabase.FindAssets($"t:{nameof(UdonSharpProgramAsset)}");
			string[] udonSharpNames = new string[udonSharpDataAssets.Length];
			Debug.Log( $"Found {udonSharpDataAssets.Length} assets." );

			_programAssetCache = new UdonSharpProgramAsset[udonSharpDataAssets.Length];

			for (int i = 0; i < _programAssetCache.Length; ++i)
			{
				udonSharpDataAssets[i] = AssetDatabase.GUIDToAssetPath(udonSharpDataAssets[i]);
			}

			foreach(string s in AssetDatabase.GetAllAssetPaths() )
			{
				if(!udonSharpDataAssets.Contains(s))
				{
					Type t = AssetDatabase.GetMainAssetTypeAtPath(s);
					if (t != null && t.FullName == "UdonSharp.UdonSharpProgramAsset")
					{
						Debug.Log( $"Trying to recover {s}" );
						Selection.activeObject = AssetDatabase.LoadAssetAtPath<UnityEngine.Object>(s);
					}
				}
			}

			ClearProgramAssetCache();

			GetAllUdonSharpPrograms();
		}
```
