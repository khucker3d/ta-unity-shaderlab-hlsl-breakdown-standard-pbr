# Tech Art: ShaderLab & HLSL Breakdown (Standard PBR)
By Kellie Hucker

---

# Overview
This document breaks down the structure of a **Standard HLSL URP PBR Shader** in Unity.

It covers:
- ShaderLab structure
- HLSL shader stages
- Material properties
- Rendering pipeline flow
- Full annotated shader example

---

# Standard URP PBR Shader Parts

## Properties

The Properties block defines material inputs.
These appear in the Material Inspector, allowing artists and developers to control shader behavior without modifying code.

Purpose:
- Expose controls to artists
- Drive shader behavior from materials
- Enable reusable and flexible shaders

Example:
<pre>Properties
{
    // Material properties here
}</pre>


## SubShader

The SubShader block defines how the shader is rendered.

It is the core section that tells Unity:
- Which passes to use
- How to process vertex and fragment data
- What pipeline and render conditions apply

Example:
<pre>SubShader
{
    Pass
    {                
        Name "ExamplePassName"
        Tags { "LightMode" = "ExampleLightModeTagValue" }

        HLSLPROGRAM
        // HLSL shader code
        ENDHLSL
    }
}</pre>

Pass

## A Pass represents a single rendering step.

Each pass:
- Runs its own vertex shader
- Runs its own fragment shader
- Contributes to a specific part of rendering

Example:
<pre>Pass
{                
    Name "ExamplePassName"
    Tags { "LightMode" = "ExampleLightModeTagValue" }

    HLSLPROGRAM
    // HLSL shader code
    ENDHLSL
}</pre>

## Vertex Shader

The vertex shader runs once per vertex. It is the first programmable stage of the GPU pipeline.

Responsibilities
- Transform positions from object space → clip space
- Prepare vertex data for the fragment shader
- Pass UVs, normals, tangents, and world position

Example:
<pre>Varyings vert(Attributes IN)
{
    Varyings OUT;

    // Convert spaces
    // Set clip-space position
    // Pass UVs
    // Pass world-space data

    return OUT;
}</pre>

## Fragment (Pixel) Shader

The fragment shader runs once per pixel. It determines the final color of each pixel.

Responsibilities:
- Sample textures
- Apply lighting (PBR)
- Combine material inputs
- Output final color

Example:
<pre>half4 frag(Varyings IN) : SV_Target
{
    return UniversalFragmentPBR(inputData, surfaceData);
}</pre>

Standard URP PBR Shader Code Breakdown: Full Shader Example
<pre>Shader "Custom/URP/StandardPBR"
{
    Properties
    {
        _BaseMap("Albedo", 2D) = "white" {}
        _BaseColor("Base Color", Color) = (1,1,1,1)
        _MetallicMap("Metallic Map", 2D) = "black" {}
        _Metallic("Metallic", Range(0,1)) = 0.0
        _Smoothness("Smoothness", Range(0,1)) = 0.5
        _OcclusionMap("AO Map", 2D) = "white" {}
        _OcclusionStrength("AO Strength", Range(0,1)) = 1.0
        _NormalMap("Normal Map", 2D) = "bump" {}
        _BumpScale("Normal Strength", Range(0,2)) = 1.0
        _EmissionMap("Emission Map", 2D) = "black" {}
        _EmissionColor("Emission Color", Color) = (0,0,0,1)
    }

    SubShader
    {
        Tags 
        {
            "RenderPipeline" = "UniversalPipeline"
            "RenderType" = "Opaque"
        }

        LOD 300 

        Pass
        {
            Name "ForwardLit" 

            Tags 
            {
                "LightMode" = "UniversalForward" 
            } 

            HLSLPROGRAM 

            #pragma vertex vert
            #pragma fragment frag
            #pragma target 3.0  

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float4 tangentOS : TANGENT;
                float2 uv : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 positionWS : TEXCOORD1;
                float3 normalWS : TEXCOORD2;
                float4 tangentWS : TEXCOORD3;
            };

            TEXTURE2D(_MetallicMap); SAMPLER(sampler_MetallicMap);
            TEXTURE2D(_OcclusionMap); SAMPLER(sampler_OcclusionMap);
            TEXTURE2D(_NormalMap);    SAMPLER(sampler_NormalMap);

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseColor;
                float4 _EmissionColor;
                float _Metallic;
                float _Smoothness;
                float _OcclusionStrength;
                float _BumpScale;
            CBUFFER_END

            Varyings vert(Attributes IN)
            {
                Varyings OUT;

                VertexPositionInputs posInputs = GetVertexPositionInputs(IN.positionOS.xyz);
                VertexNormalInputs normInputs = GetVertexNormalInputs(IN.normalOS, IN.tangentOS);

                OUT.positionHCS = posInputs.positionCS;
                OUT.uv = IN.uv;
                OUT.positionWS = posInputs.positionWS;
                OUT.normalWS = normInputs.normalWS;
                OUT.tangentWS = float4(normInputs.tangentWS, 1.0);

                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                float4 albedoSample = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv) * _BaseColor;
                float4 emissionSample = SAMPLE_TEXTURE2D(_EmissionMap, sampler_EmissionMap, IN.uv) * _EmissionColor;

                float metallicSample = SAMPLE_TEXTURE2D(_MetallicMap, sampler_MetallicMap, IN.uv).r;
                float aoSample = SAMPLE_TEXTURE2D(_OcclusionMap, sampler_OcclusionMap, IN.uv).r;

                float3 normalTS = UnpackNormalScale(
                    SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, IN.uv),
                    _BumpScale
                );

                float3x3 tangentToWorld = CreateTangentToWorld(
                    IN.normalWS,
                    IN.tangentWS.xyz,
                    IN.tangentWS.w
                );

                float3 normalWS = normalize(mul(normalTS, tangentToWorld));

                InputData inputData;
                inputData.positionWS = IN.positionWS;
                inputData.normalWS = normalWS;
                inputData.viewDirectionWS = GetWorldSpaceViewDir(IN.positionWS);
                inputData.shadowCoord = 0;
                inputData.vertexLighting = 0;
                inputData.bakedGI = SAMPLE_GI(IN.uv, IN.positionWS, normalWS);
                inputData.normalizedScreenSpaceUV = 0;
                inputData.shadowMask = 1;

                SurfaceData surfaceData;
                surfaceData.albedo = albedoSample.rgb;
                surfaceData.metallic = metallicSample > 0.001 ? metallicSample : _Metallic;
                surfaceData.specular = 0;
                surfaceData.smoothness = _Smoothness;
                surfaceData.normalTS = float3(0, 0, 1);
                surfaceData.occlusion = aoSample * _OcclusionStrength;
                surfaceData.emission = emissionSample.rgb;
                surfaceData.alpha = 1;
                surfaceData.clearCoatMask = 0;
                surfaceData.clearCoatSmoothness = 0;

                return UniversalFragmentPBR(inputData, surfaceData);
            }</pre>

            ENDHLSL
        }
    }

    FallBack "Hidden/InternalErrorShader"
}
