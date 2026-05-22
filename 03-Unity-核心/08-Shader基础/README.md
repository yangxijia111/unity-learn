# 08 - Shader 基础 (Shader Basics)

Shader 控制物体表面如何渲染——颜色、光照、特效，是提升画面表现的核心技术。

## ShaderLab 语法结构

Unity 使用 ShaderLab 包装 Shader 代码，结构如下：

```hlsl
Shader "MyShader/Basic"
{
    // 属性面板暴露的参数
    Properties
    {
        _MainTex ("主纹理", 2D) = "white" {}
        _Color ("颜色", Color) = (1,1,1,1)
        _Gloss ("光泽度", Range(0,1)) = 0.5
    }

    // 可以有多个 SubShader（适配不同硬件）
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry" }

        Pass
        {
            // Shader 代码写在这里
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            // 变量声明（对应 Properties）
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;
            float _Gloss;

            // 顶点着色器输入
            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : NORMAL;
            };

            // 顶点到片元的传递
            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 worldNormal : TEXCOORD1;
            };

            // 顶点着色器
            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                return o;
            }

            // 片元（像素）着色器
            fixed4 frag(v2f i) : SV_Target
            {
                fixed4 texColor = tex2D(_MainTex, i.uv);
                return texColor * _Color;
            }
            ENDCG
        }
    }

    // 兜底方案（所有 SubShader 都不支持时）
    FallBack "Diffuse"
}
```

## 基础属性类型

| 类型 | 语法 | 说明 |
|------|------|------|
| `Color` | `_Color ("Color", Color) = (1,1,1,1)` | RGBA 颜色 |
| `Float` | `_Value ("Value", Float) = 0.5` | 浮点数 |
| `Range` | `_Gloss ("Gloss", Range(0,1)) = 0.5` | 滑动条范围值 |
| `2D` | `_MainTex ("Texture", 2D) = "white" {}` | 2D 纹理 |
| `Cube` | `_CubeMap ("Cubemap", CUBE) = "" {}` | 立方体贴图 |
| `Vector` | `_Dir ("Direction", Vector) = (0,1,0,0)` | 四维向量 |

## URP Shader（推荐）

URP 使用 HLSL 代替 CG，需要引用 URP 的库：

```hlsl
Shader "MyShader/URPBasic"
{
    Properties
    {
        _BaseColor ("基础颜色", Color) = (1,1,1,1)
        _BaseMap ("主纹理", 2D) = "white" {}
    }

    SubShader
    {
        Tags {
            "RenderPipeline" = "UniversalPipeline"
            "RenderType" = "Opaque"
            "Queue" = "Geometry"
        }

        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                half4 _BaseColor;
            CBUFFER_END

            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
                float3 normalOS : NORMAL;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 normalWS : TEXCOORD1;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                OUT.normalWS = TransformObjectToWorldNormal(IN.normalOS);
                return OUT;
            }

            half4 frag(Varyings IN) : SV_Target
            {
                half4 tex = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);

                // 简单兰伯特光照
                Light mainLight = GetMainLight();
                float NdotL = saturate(dot(normalize(IN.normalWS), mainLight.direction));
                half3 color = tex.rgb * _BaseColor.rgb * (mainLight.color * NdotL);

                return half4(color, 1);
            }
            ENDHLSL
        }
    }
}
```

## 常用特效 Shader

### 描边效果

```hlsl
Shader "MyShader/Outline"
{
    Properties
    {
        _MainTex ("纹理", 2D) = "white" {}
        _OutlineColor ("描边颜色", Color) = (0,0,0,1)
        _OutlineWidth ("描边宽度", Range(0, 0.1)) = 0.02
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" }

        // Pass 1: 描边（背面放大法）
        Pass
        {
            Name "Outline"
            Cull Front // 剔除正面，只渲染背面

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            float _OutlineWidth;
            fixed4 _OutlineColor;

            struct appdata { float4 vertex : POSITION; float3 normal : NORMAL; };
            struct v2f { float4 pos : SV_POSITION; };

            v2f vert(appdata v)
            {
                v2f o;
                // 沿法线方向向外扩展
                float3 expandedPos = v.vertex.xyz + v.normal * _OutlineWidth;
                o.pos = UnityObjectToClipPos(float4(expandedPos, 1));
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                return _OutlineColor;
            }
            ENDCG
        }

        // Pass 2: 正常渲染
        Pass
        {
            Name "Base"
            Cull Back

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            float4 _MainTex_ST;

            struct appdata { float4 vertex : POSITION; float2 uv : TEXCOORD0; };
            struct v2f { float4 pos : SV_POSITION; float2 uv : TEXCOORD0; };

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                return tex2D(_MainTex, i.uv);
            }
            ENDCG
        }
    }
}
```

### 溶解效果

```hlsl
Shader "MyShader/Dissolve"
{
    Properties
    {
        _MainTex ("主纹理", 2D) = "white" {}
        _NoiseTex ("噪声纹理", 2D) = "white" {}
        _Threshold ("溶解阈值", Range(0,1)) = 0
        _EdgeColor ("边缘颜色", Color) = (1, 0.5, 0, 1)
        _EdgeWidth ("边缘宽度", Range(0, 0.1)) = 0.03
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            sampler2D _NoiseTex;
            float4 _MainTex_ST;
            float _Threshold;
            fixed4 _EdgeColor;
            float _EdgeWidth;

            struct appdata { float4 vertex : POSITION; float2 uv : TEXCOORD0; };
            struct v2f { float4 pos : SV_POSITION; float2 uv : TEXCOORD0; };

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                float noise = tex2D(_NoiseTex, i.uv).r;

                // 低于阈值的像素丢弃（溶解掉）
                clip(noise - _Threshold);

                fixed4 col = tex2D(_MainTex, i.uv);

                // 边缘发光
                float edge = smoothstep(0, _EdgeWidth, noise - _Threshold);
                col.rgb = lerp(_EdgeColor.rgb, col.rgb, edge);

                return col;
            }
            ENDCG
        }
    }
}
```

### 边缘发光（Fresnel）

```hlsl
Shader "MyShader/FresnelGlow"
{
    Properties
    {
        _MainColor ("主颜色", Color) = (1,1,1,1)
        _GlowColor ("发光颜色", Color) = (0, 0.5, 1, 1)
        _GlowPower ("发光强度", Range(0, 5)) = 2
        _FresnelPower ("菲涅尔指数", Range(0.1, 5)) = 3
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            fixed4 _MainColor;
            fixed4 _GlowColor;
            float _GlowPower;
            float _FresnelPower;

            struct appdata { float4 vertex : POSITION; float3 normal : NORMAL; };
            struct v2f { float4 pos : SV_POSITION; float3 worldNormal : TEXCOORD0; float3 viewDir : TEXCOORD1; };

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.viewDir = normalize(_WorldSpaceCameraPos - worldPos);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                float NdotV = saturate(dot(normalize(i.worldNormal), i.viewDir));

                // Fresnel：边缘值大，中心值小
                float fresnel = pow(1 - NdotV, _FresnelPower);

                fixed4 col = _MainColor;
                col.rgb += _GlowColor.rgb * fresnel * _GlowPower;

                return col;
            }
            ENDCG
        }
    }
}
```

## 在 C# 中控制 Shader 参数

```csharp
public class ShaderController : MonoBehaviour
{
    private Material mat;

    void Start()
    {
        // 获取材质实例（不修改共享材质）
        mat = GetComponent<Renderer>().material;
    }

    void Update()
    {
        // 修改溶解阈值（实现溶解动画）
        float threshold = Mathf.PingPong(Time.time * 0.5f, 1f);
        mat.SetFloat("_Threshold", threshold);

        // 修改颜色
        mat.SetColor("_EdgeColor", Color.Lerp(Color.red, Color.yellow, Mathf.PingPong(Time.time, 1f)));

        // 修改纹理偏移（UV 动画，如水流、滚动文字）
        Vector2 offset = mat.GetTextureOffset("_MainTex");
        offset.y += Time.deltaTime * 0.5f;
        mat.SetTextureOffset("_MainTex", offset);
    }

    void OnDestroy()
    {
        // 销毁实例化材质，避免内存泄漏
        Destroy(mat);
    }
}
```

## Shader 编写建议

| 建议 | 说明 |
|------|------|
| 用 Shader Graph | URP 项目推荐可视化编辑，不需要手写代码 |
| 分析已有 Shader | 学习 URP 内置 Shader 的写法 |
| 注意性能 | 片元着色器中避免复杂运算，用贴图查表代替计算 |
| 适配平台 | 移动端用 URP Shader，避免 Built-in 的 CG 代码 |
| 调试技巧 | 输出中间值为颜色（如 `return float4(normal, 1)` 查看法线） |

## 学习路径

1. **Shader Graph**（可视化，零代码入门）
2. **基础 ShaderLab**（理解结构和语法）
3. **URP/HLSL Shader**（正式项目用）
4. **数学基础**（向量、矩阵、三角函数、点积叉积）
5. **渲染管线原理**（了解顶点→片元→输出的完整流程）

## 注意事项

1. URP 项目**必须**用 URP Shader，Built-in Shader 会粉屏
2. `Properties` 中声明的变量需要在 CG/HLSL 中**再次声明**才能使用
3. `CBUFFER_START(UnityPerMaterial)` 在 URP 中是**必须的**，否则 SRP Batcher 不兼容
4. 纹理采样在 URP 中用 `SAMPLE_TEXTURE2D` 代替 `tex2D`
5. 用 `clip()` 函数在片元着色器中丢弃像素，实现镂空、溶解等效果
