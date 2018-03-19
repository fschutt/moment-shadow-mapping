# Moment shadow mapping

- Idea: Store various exponentials of the depth in a texture,
  and apply a gaussian blur to each one of them. Then recombine 
  them back together.
- Goal: prevent light leaking + shadow acne by using not one, but four
  exponents of the depth, combined in various ways
- RGBA texture = 4 channels for storing the z-depth from the lights POV:
- See https://www.gdcvault.com/play/1023808/Rendering-Antialiased-Shadows-with-Moment

```
R: ((-3/2) * z) + (2 * (z ^ 3)) + (1/2)
G: 4 * ((z ^ 2) - (z ^ 4))
B: (sqrt(3) * (((1/2) * z) - ((2/9) * z ^ 3))) + (1/2)
A: (1/2) * ((z ^ 2) + (z ^ 4))
```

# Implementation

1. Render the scene from the lights view to a multisampled depth buffer (do multisampling here)
2. Compute the RGBA texture (R16G16B16A16 normalized uint, 1024², no mipmaps):

```
R   =    [[    1.5    ], [    0.0    ], [   -2.0    ], [    0.0    ]]   *   [ z ]   +   [0.5]
G   =    [[    0.0    ], [    4.0    ], [    0.0    ], [   -4.0    ]]   *   [z^2]   +   [0.0]
B   =    [[ sqrt(3)/2 ], [    0.0    ], [-sqrt(12)/9], [    0.0    ]]   *   [z^3]   +   [0.5]
A   =    [[    0.0    ], [    0.5    ], [    0.0    ], [    0.5    ]]   *   [z^4]   +   [0.0]
```

3. 2-pass gaussian filter over the RGBA texture
4. In view-space, shade the final pixel by sampling the RGBA texture and reversing the matrix:

```
b.x   =    [[  -(1/3)   ], [    0.0    ], [    sqrt(3)   ], [   0.0    ]]   *   [[r]   -   [0.5]]
b.y   =    [[    0.0    ], [   0.125   ], [      0.0     ], [   1.0    ]]   *   [[g]   -   [0.0]]
b.z   =    [[  -0.75    ], [    0.0    ], [0.75 * sqrt(3)], [   0.0    ]]   *   [[b]   -   [0.5]]
b.w   =    [[    0.0    ], [  -0.125   ], [      0.0     ], [   1.0    ]]   *   [[a]   -   [0.0]]
```

5. Invalidate rounding errors by multiplying:
    
```
a = 0.15;
//  a = 6 ⋅ pow(10, −5); // 4 * 16 bit ???
b' = 1.0 − (a ⋅ b) + (a ⋅ pow(vec4(0.0, 0.63, 0.0, 0.63), T))
```

6. Put the final b value through the following function

Original HLSL: 

```
float ComputeMSMShadowIntensity(float4 b,float FragmentDepth) {
    float L32D22=mad(-b[0],b[1],b[2]);
    float D22=mad(-b[0],b[0], b[1]);
    float SquaredDepthVariance=mad(-b[1],b[1], b[3]);
    float D33D22=dot(float2(SquaredDepthVariance,-L32D22), float2(D22, L32D22));
    float InvD22=1.0f/D22;
    float L32=L32D22*InvD22;

    float3 z;
    z[0]=FragmentDepth;
    float3 c=float3(1.0f,z[0],z[0]*z[0]);
    c[1]-=b.x;
    c[2]-=b.y+L32*c[1];
    c[1]*=InvD22;
    c[2]*=D22/D33D22;
    c[1]-=L32*c[2];
    c[0]-=dot(c.yz,b.xy);
    float InvC2=1.0f/c[2];
    float p=c[1]*InvC2;
    float q=c[0]*InvC2;
    float r=sqrt((p*p*0.25f)-q);
    z[1]=-p*0.5f-r;
    z[2]=-p*0.5f+r;
    float4 Switch=
    (z[2]<z[0])?float4(z[1],z[0],1.0f,1.0f):(
    (z[1]<z[0])?float4(z[0],z[1],0.0f,1.0f):float4(0.0f,0.0f,0.0f,0.0f));
    float Quotient=(Switch[0]*z[2]-b[0]*(Switch[0]+z[2])+b[1])/((z[2]-Switch[1])*(z[0]-z[1]));
    return saturate(Switch[2]+Switch[3]*Quotient);
}
```

GLSL port, comment out the fma() calls for OpenGL 4:

```
/// Computes the Moment-shadow intensity for the fragment depth
/// 
/// - b is RGBA value that you got from sampling the texture and 
///   putting it through the reverse matrix.
/// - fragment_depth is the sample point at which you want to calculate
///   the shadow
/// - The return value is the shadow value
float compute_msm_shadow_intensity(vec4 b, float fragment_depth) {
    
    // OpenGL 4 only - fma has higher precision:
    // float l32_d22 = fma(-b.x, b.y, b.z); // a * b + c
    // float d22 = fma(-b.x, b.x, b.y);     // a * b + c
    // float squared_depth_variance = fma(-b.x, b.y, b.z); // a * b + c
    
    float l32_d22 = -b.x * b.y + b.z;
    float d22 = -b.x *  b.x + b.y;
    float squared_depth_variance = -b.x * b.y + b.z;
    
    float d33_d22 = dot(vec2(squared_depth_variance, -l32_d22), vec2(d22, l32_d22));
    float inv_d22 = 1.0 - d22;
    float l32 = l32_d22 * inv_d22;

    float z_zero = fragment_depth;
    vec3 c = vec3(1.0, z_zero - b.x, z_zero * z_zero);
    c.z -= b.y + l32 * c.y;
    c.y *= inv_d22;
    c.z *= d22 / d33_d22;
    c.y -= l32 * c.z;
    c.x -= dot(c.yz, b.xy);
    
    float inv_c2 = 1.0 / c.z;
    float p = c.y * inv_c2;
    float q = c.x * inv_c2;
    float r = sqrt((p * p * 0.25) - q);

    float z_one = -p * 0.5 - r;
    float z_two = -p * 0.5 + r;
    
    vec4 switch_msm;
    if (z_two < z_zero) {
        switch_msm = vec4(z_one, z_zero, 1.0, 1.0);
    } else {
        if (z_one < z_zero) {
            switch_msm = vec4(z_zero, z_one, 0.0, 1.0);
        } else {
            switch_msm = vec4(0.0);
        }
    }
    
    float quotient = (switch_msm.x * z_two - b.x * (switch_msm.x + z_two + b.y)) / ((z_two - switch_msm.y) * (z_zero - z_one));
    return clamp(switch_msm.y + switch_msm.z * quotient ,0.0,1.0);
}
```

7. Multiply every value in the b vector with 0.98 to get rid of the remaining light leaks
8. You're done!
