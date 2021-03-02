---
 layout:     post
 title:      TownShip的海水效果
 subtitle:   通过Unity复刻海水效果
 date:       2021-02-21
 author:     Bob
 header-img: img/post-bg-unity.jpg
 catalog: true
 tags:
     - Unity
---

##### TownShip海水效果抓帧

 分为海水效果、鱼群游动、鱼跳跃、海岸线浪花、帆船航行、点击水面特效、海鸟滑翔等效果

 ![image](/img/water/splash.png)


##### 海面Mesh

整个海面是由多块小区域不规则海面构成，每个小区域是由多个菱形mesh构成。菱形比方形波动效果更好。

 ![image](/img/water/mesh.png)

##### 海岸线浪花Mesh

海岸线的浪花是叠加的一组mesh
 ![image](/img/water/beach.png)


##### 4个通道贴图

 Waterx贴图
 ![image](/img/water/waterx_normals.webp)

 前三个通道分别如下：
 ![image](/img/water/rgb.png)

Red通道：小细碎浪头闪烁的白花
Green通道:大面积浪花
Blue通道：海面暗影区域

#### Unity Shader

```c
Shader "Unlit/Sea"
{
	Properties
	{
		_MainTex("Texture", 2D) = "white" {}
		_SeaColor("SeaColor", Color) = (0, 0.45, 0.63,1)
		_u0("u0",Range(0,1)) = 0.08
		_u1("u1",Range(0,1)) = 0.1
		_speed("Speed",Range(0,1)) = 0.015
		_waveSpeed("WaveSpeed",Range(0,2)) = 2
	}
		SubShader
		{
			Tags { "RenderType" = "Opaque" }
			LOD 100

			Pass
			{
				CGPROGRAM
				#pragma vertex vert
				#pragma fragment frag

				#include "UnityCG.cginc"

				struct appdata
				{
					float4 vertex : POSITION;
					float4 col : COLOR;
				};

				struct v2f
				{
					float2 uv0 : TEXCOORD0;
					float2 uv1 : TEXCOORD1;
					float4 vcolor : COLOR;
					float4 vertex : SV_POSITION;
					float2 v : TEXCOORD2;
				};

				sampler2D _MainTex;
				float4 _MainTex_ST;
				float4 _SeaColor;
				float _u0;
				float _u1;
				float _speed;
				float _waveSpeed;

				v2f vert(appdata v)
				{
					v2f o;
					//uv移动动画
					float t = _Time.y * _speed;
					o.uv0 = v.vertex.xz* _u0 * v.col.b;
					o.uv0.y -= t;
					o.uv1 = v.vertex.xz * _u1 * v.col.b;
					o.uv1.y += t;

					//波浪效果
					float wave_m = (v.col.b * _waveSpeed);
					float w_t0 = wave_m * o.uv0.x * 40.0 + wave_m * o.uv0.y * 20.0;
					float w_t1 = wave_m * o.uv0.x * 20.0 + wave_m * o.uv0.y * 10.0 - t * 143.5;
					float wa   = (v.col.r - 127.0 / 255.0) * 100.0 * (1.0 / wave_m);
					float yoff = sin(w_t0) * sin(w_t1) * 0.03 * wa;
					v.vertex.y += yoff;
					o.vertex = UnityObjectToClipPos(v.vertex);

					o.v.xy = o.vertex.xy;
					o.v.y -= 0.5;

					o.vcolor = v.col;
					o.vcolor.b = yoff * 0.004;
					o.vcolor.a *= 0.5;

					return o;
				}

				fixed4 frag(v2f i) : SV_Target
				{

					//高光
					float hlight = exp(-i.v.x * i.v.x * 8.0 - i.v.y * i.v.y * 1.0) * 1.25;

					fixed4 col0 = tex2D(_MainTex, i.uv0);
					fixed4 col1 = tex2D(_MainTex, i.uv1);
					fixed4 col = col0 * col1;

					return _SeaColor
							+ col.r
							+ col.g
							- col.b * i.vcolor.g *0.5
							- (1.0 - col.a) * 0.3
							+ hlight * col.r + hlight * 0.15
							+ i.vcolor.a - 0.5;

				}
				ENDCG
			}
		}
}
```

##### Unity海水复刻渲染效果
 ![image](/img/water/wave.gif)

##### Waterx.fsh

```c
uniform mediump sampler2D sampler;

varying mediump vec4 v_color;
varying highp vec2 v_texcoord0;
varying highp vec2 v_texcoord1;

const mediump vec3 v_seaColor = vec3(0, 0.45, 0.63);

varying mediump float v_x;
varying mediump float v_y;

// первый вариант 第一方案


void main()
{
    // блик 耀斑
    mediump float hlight = exp(-v_x * v_x * 8.0 - v_y * v_y * 1.0) * 1.25;

    // выборки из текстуры 纹理抽样
    mediump vec4 col0 = texture2D(sampler, v_texcoord0);
    mediump vec4 col1 = texture2D(sampler, v_texcoord1);

    // перемножаем значения и получаем общий цвет 我们得到了一个共同的颜色
    mediump vec4 col = col0 * col1;

    // вычисляем финальный цвет пикселя 计算像素的最后颜色
    mediump vec4 pix;
    pix.rgb = v_seaColor                        // цвет моря 海色
              + col.r                           // засветы на волнах (red) 闪烁的波浪
              + col.g                           // более крупные засветы на волнах (green) 波浪上更大的耀斑
              - (1.0 - col.a) * .3              // затемнение для имитации объема (alpha) 变暗以模拟量?
              + hlight * col.r + hlight * .15   // блик 耀斑
              - col.b * v_color.g               // затенение на базе blue канала для придания объема 基于蓝色通道的阴影以增加量
              //+ (v_color.a - 1.0) * .5        // общее затемнение удаленных областей (alpha вершины) 偏远地区普遍变黑
              + v_color.a - .5
    ;

    gl_FragColor.rgb = pix.rgb;
    gl_FragColor.a = 1.0;
}

```

##### Waterx.vsh

```c

attribute highp vec4 a_vertex;
attribute mediump vec4 a_color;
attribute highp vec2 a_texcoord;

uniform highp mat4 u_modelview;
uniform highp float u_t;

varying mediump vec4 v_color;
varying highp vec2 v_texcoord0;
varying highp vec2 v_texcoord1;
varying mediump float v_x;
varying mediump float v_y;
varying mediump float t;

void main()
{
	v_color = a_color;
	
	// a_color.r - амплитуда волнения 幅度
	// a_color.g - затемнение (имитация впадин) 变暗（模仿凹陷）
	// a_color.b - множитель плотности текстуры и волнения (тут обратная зависимость) 纹理密度和幅度的乘数（此处为反比）
	// a_color.a - затемнение удаленных областей 使偏远地区变暗
	
	highp float t = u_t * 0.015;  // Общий коэффициент скорости 总速比
	//~ float m = 0.8 * 2.0;    // Множитель координат текстуры 纹理坐标倍增
	highp float wave_m = (v_color.b * 2.0); // Множитель волнения 幅度乘数
	
	// генерируем текстурные координаты и анимируем их 生成纹理坐标并对它们进行动画处理
	v_texcoord0 = a_vertex.xy * 0.008 * v_color.b;
	v_texcoord0.y -= t;
	v_texcoord1 = a_vertex.xy * 0.010 * v_color.b;
	v_texcoord1.y += t;
	
	highp vec4 vertex = a_vertex;
	
	// волнение 波动,波涛 扰动？
	
	highp float w_t0 = v_texcoord0.x * 40.0 * wave_m + v_texcoord0.y * 20.0 * wave_m;
	highp float w_t1 = v_texcoord0.x * 20.0 * wave_m + v_texcoord0.y * 10.0 * wave_m - t * 143.5;// * (1.0 / wave_m);
	highp float wa   = (a_color.r - 127.0 / 255.0) * 100.0 * (1.0 / wave_m); // Амплитуда волнения 幅度
	highp float yoff = sin(w_t0) * sin(w_t1) * wa; // Волнение
	vertex.y += yoff;
	
	// сохраняем смещение по y для дальнейшего учета во фрагментном шейдере 保存y偏移以在片段着色器中进一步考虑
	v_color.b = yoff * 0.004;

	v_color.a *= 0.5;
	
	gl_Position = u_modelview * vertex;
	
	v_x = gl_Position.x; // Для блика - позиция относительно экрана 对于耀斑-相对于屏幕的位置
	v_y = gl_Position.y - .5;
}

```

 Water贴图
 ![image](/img/water/water_normals.webp)

##### Water.vsh

```c
uniform mediump sampler2D sampler;

varying mediump vec4 v_color;
varying highp vec2 v_texcoord;
varying mediump float v_st;
varying mediump float v_ct;

const mediump vec3 v_waterLightSea = vec3(0, 0.635, 0.886);
const mediump vec3 v_waterDarkSea = vec3(0, 0.408, 0.647);

varying mediump float v_x;
varying mediump float v_y;

void main()
{
	mediump vec4 normalMapValue = texture2D(sampler, v_texcoord);
	
	gl_FragColor.rgb = mix(v_waterLightSea, v_waterDarkSea, normalMapValue.x * v_st + normalMapValue.y * v_ct) // Меняется основная текстура
		+ (normalMapValue.z * v_st + normalMapValue.w * v_ct) * exp(-v_x * v_x * 8.0 - v_y * v_y * 2.0) * 0.5 // Блик
		+ (v_color.a - 1.0) // Затемнение (передаётся в альфе).
		;
	
	gl_FragColor.a = 1.0;
}

```

##### Water.fsh

```c
attribute highp vec4 a_vertex;
attribute mediump vec4 a_color;
attribute highp vec2 a_texcoord;

uniform highp mat4 u_modelview;
uniform highp float u_t;

varying mediump vec4 v_color;
varying highp vec2 v_texcoord;
varying mediump float v_st;
varying mediump float v_ct;
varying mediump float v_x;
varying mediump float v_y;

void main()
{
	v_color = a_color;
	
	highp float t = u_t * 0.5; // Общий коэффициент скорости
	
	v_texcoord = a_texcoord;
	v_texcoord.y += t * 0.02; // Сдвиг для течения
	
	highp vec4 vertex = a_vertex;
	
	highp float w_t = a_texcoord.y * 3.14 * 3.0 + a_texcoord.x * 1.5 + t * 3.14;
	
	highp float wa = (a_color.r - 127.0 / 255.0) * 2.0 * 25.0; // Амплитуда волнения.
	
	vertex.y += sin(w_t) * wa; // Волнение
	
	highp float tex_t = (a_texcoord.y * 3.14 * 3.0 + a_texcoord.x * 1.5 + t * 3.14) * 1.2;
	v_st = sin(tex_t) * 0.5 + 0.5; // Texture a coef
	v_ct = 1.0 - v_st; // Texture b coef
	
	gl_Position = u_modelview * vertex;
	
	v_x = gl_Position.x; // Для блика - позиция относительно экрана
	v_y = gl_Position.y;
}

```

##### wavesDistortion.vsh

```c

attribute highp vec4 a_vertex;
attribute mediump vec2 a_texcoord;
attribute mediump vec4 a_color;

uniform highp mat4 u_modelview;
uniform highp float u_time; // умножается на 4.0 в коде, скорость фазы - скорость волны
uniform highp float u_scale;

varying mediump vec4 v_color;
varying mediump vec2 v_texcoord;

void main()
{
	highp vec4 pos = u_modelview * a_vertex;

	// 1) 512.0 * 0.05 = 25.6; 512 - маппинг экранных координат в длину волны; 0.05 - подобранный множитель
	// 2) Добавлен учет u_scale, отражающего масштабирование рабочей области текстуры по отношению к ее размеру
	v_texcoord = a_texcoord + sin(((pos.x * 25.6) / u_scale) + u_time) * 0.001 * u_scale * a_color.r;

	v_color = a_color;

	gl_Position = pos;
}


```

##### Fish.vsh

```c
uniform mediump sampler2D sampler;
uniform mediump sampler2D sampler1; // шум
uniform mediump vec3 params; // альфа (x), масштаб (y), таймер (z)
varying mediump vec4 v_color;
varying highp vec2 v_texcoord;

mediump vec4 sample1(in highp vec2 texCoord) {
	return texture2D(sampler1, texCoord);
}

void main()
{
    mediump vec4 noise0 = sample1(v_texcoord * params.y * 2.0 + vec2(params.z, 0));
    mediump vec4 noise1 = sample1(v_texcoord * params.y * 2.0 - vec2(params.z, 0));
    mediump vec2 noise  = ((noise0.xy + noise1.xy) * 2.0 - 1.0) * 0.075;
    
    gl_FragColor = texture2D(sampler, v_texcoord + noise);
    gl_FragColor.a *= params.x;
}
```


##### Foam.vsh

```c
attribute highp vec4 a_vertex;
attribute mediump vec4 a_color;
attribute highp vec2 a_texcoord;

uniform highp mat4 u_modelview;
uniform highp float u_t;

varying mediump vec4 v_color;
varying highp vec2 v_texcoord;

void main()
{
	v_color = vec4(1.0, 1.0, 1.0, 1.0);
	v_texcoord = a_texcoord;
	
	highp float t = u_t * 0.25; // Общий коэффициент скорости.
	
	// Дополнительные параметры передаются через цвет.
	highp float wcoord = a_color.r; // Фаза в пространстве.
	
	highp float w_t = (wcoord + t) * 3.14 * 2.0; // Здесь не должно быть других коэффициентов. Коэффициент для wcoord задавайте в коде, коэффициент для t задаётся парой строк выше.
	
	v_color.a = sin(w_t) * 0.5 * a_color.a + 0.6;
	v_color.a = v_color.a * v_color.a; // Для большей резкости.
	
	highp float dx = (a_color.g - 127.0 / 255.0) * 2.0 * 25.0; // Направление и амплитуда колебания пены.
	highp float dy = (a_color.b - 127.0 / 255.0) * 2.0 * 25.0;
	
	highp vec4 vertex = a_vertex;
	
	w_t = w_t - 3.14 * 0.25; // Небольшое смещение фазы, чтобы альфа лучше совпадала с набегающими волнами.
	vertex.x += sin(w_t) * dx; // Волнение.
	vertex.y += sin(w_t) * dy;
	
	gl_Position = u_modelview * vertex;
}
```

