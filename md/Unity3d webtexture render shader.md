
``` c
Shader "_Game/WebCamStyle"
{
	Properties
	{
		_MainTex ("Texture", 2D) = "white" {}
		_CoverTex ("Texture", 2D) = "black" {}
		_FlipStyle("Flip Style",Range(-5,5)) = 0
	}
	SubShader
	{
		Tags
		{ 
			"Queue"="Transparent" 
			"IgnoreProjector"="True" 
			"RenderType"="Transparent" 
			"PreviewType"="Plane"
			"CanUseSpriteAtlas"="True"
		}
		LOD 100
		Lighting Off
		ZWrite Off
		Blend SrcAlpha OneMinusSrcAlpha
		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			// make fog work
			#pragma multi_compile_fog
			
			#include "UnityCG.cginc"

			struct appdata
			{
				float4 vertex : POSITION;
				float4 color    : COLOR;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float4 uv : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				float4 vertex : SV_POSITION;
				float4 color    : COLOR;
			};

			sampler2D _MainTex;
			sampler2D _CoverTex;
			float4 _MainTex_ST;
			float4 _CoverTex_ST;

			float _FlipStyle;
			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
				o.uv.zw = TRANSFORM_TEX(v.uv, _CoverTex);
				o.color = v.color;
				UNITY_TRANSFER_FOG(o,o.vertex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				// sample the texture
				fixed2 mainCoord;
				if(_FlipStyle == 2){
					mainCoord = fixed2(1-i.uv.y,i.uv.x);
				}

				else if(_FlipStyle == 3){
					mainCoord = fixed2(1-i.uv.x,1-i.uv.y);
				}
				else if(_FlipStyle == 4){
					mainCoord = fixed2(i.uv.y,1-i.uv.x);
				}else{
					mainCoord = i.uv.xy;
				}



				fixed4 col = tex2D(_MainTex, mainCoord);
				fixed4 cover= tex2D(_CoverTex,i.uv.zw);

				col = col * (1- cover.a) + cover.a * cover; 
				col *=i.color;
//				col  = fixed4(1,1,0,0.3);
				return col;
			}
			ENDCG
		}
	}
}


```


and C# script:
``` c#
public int FlipCode {
		get{ return flipMat (); }
	}

	private int flipMat ()
	{
		int flipCode = 0;

		if (webCamTexture.videoRotationAngle == 0) {
			flipCode = 1;
		} else if (webCamTexture.videoRotationAngle == 90) {
			flipCode = 2;
		}
		if (webCamTexture.videoRotationAngle == 180) {
			flipCode = 3;
		} else if (webCamTexture.videoRotationAngle == 270) {
			flipCode = 4;
		}
		return flipCode;
	}
```
