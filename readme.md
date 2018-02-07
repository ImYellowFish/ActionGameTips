# 动作游戏通用框架
------

# 1. 动作游戏打击感

## 1.1. 卡帧/硬直
### 两种方案

#### 简化的方案: Time.timeScale

* 需要卡帧时直接去改Time.timeScale
* 提供一个曲线来设置timeScale的渐变效果
* 缺点：可能让玩家感觉游戏很卡，不够流畅

#### 通用的方案: LocalTimeScale
* 每个角色有自己的LocalTimeScale，会叠加上全局的TimeScale
* 影响自己身上的状态机，动画和特效
* 有localTimeScale的情况下，globalTimeScale一般是1，除非暂停.
* 玩家打到Boss时，Boss卡顿一下而玩家几乎不卡，这样操作感依然很流畅
* 容易实现子弹时间

可以很方便地用来做子弹时间，别人很慢而自己很快，塞尔达里那种时快时慢的子弹时间也可以.
扩展以后还可以支持时间回溯 (参考Braid)，需要用到帧同步或者状态同步.

#### 实现时需要注意的点:
    * 动画系统 –> Animator speed
    * 粒子系统 -> Simulate 或者 simulationSpeed
    * 需要提供Yield return waitForLocalSeconds方法，延时操作都需要用这个

### 卡帧动画
```csharp
void DoUpdate(){
    hero.animator.speed = hero.localTimeScale;
}
```
### 卡帧特效

通用方案下，每个粒子系统的TimeScale需要单独调整
* Unity 5: 
```csharp
IEnumerator PlayEffectCrt(float duration){
    if(hero.localTimeScale == 1)
        yield break;

    float timer;
    while(timer < duration){
        particleSystem.Simulate(hero.localDeltaTime);
        timer += hero.localDeltaTime;
        yield return null;
    }
}
```

Simulate()方法会产生大量GC.

* Unity 2017: 
particleSystem.main.simulationSpeed = hero.localTimeScale;

某些特效可能不受TimeScale影响，始终按照正常速度播放

```csharp
if(effect.playInUnscaledTime)
    particleSystem.main.simulationSpeed = 1;
```

### Yield return方法
```csharp
IEnumerator WaitForSecondsLocal(float duration) {
    float timer = 0;
    while(timer <= duration) {
        timer += LocalDeltaTime;
        yield return null;
    }
}

// Example
IEnumerator DoSkill(Skill skill){
    SkillOn(skill);
    yield return hero.WaitForSecondsLocal(skill.duration);
    SkillEnd();
}
```

### 关于硬直
区分轻重硬直和动画，打击感才会丰富，后面会详细讲.
硬直动画用状态机来实现


## 1.2. 震屏
主要是摄像机晃动的效果，表现地面震动

推荐插件: <https://github.com/ewersp/CameraShake>

### 实现要点:
* 多个震屏之间相互独立，不干扰
* 平移和转动
* 多种震动曲线
* 使用频次太高会感觉头晕

## 1.3. 精准播放的声音和特效
音效和特效都要卡在合适的动画帧播放

### 时间配置方法：
#### 方案1: 
* 策划通过表格配置延迟时间，需要找到动画帧数，并换算成延迟
* 技能分为前摇，激活，后摇三个阶段，方便生成判定和特效
* 容易上手但修改和预览很麻烦，尤其是动画有频繁改动的时候

![alt text][skillPhase]

#### 方案2:
* 用Animation Event 控制特效和声音播放
* 省去了换算时间和配表的步骤. 
* 还是没有解决预览的问题, 特效配置可能不容易集中管理
* 适合Demo快速迭代，如果用于大项目需要自己写插件，管理所有的AnimationEvent

![alt text][animEvent]

API参考:

<http://gad.qq.com/article/detail/38140>

#### 预览问题
最好制作一个方便预览的场景，但很难满足特效和策划的要求。设计代码的时候就要考虑预览问题.




# 2. 伤害和硬直

## 伤害抽象类
* 对于动作游戏，只传入一个数字的伤害肯定是不够的。
* 封装成Damage类, 记录释放者，Skill, Buff, 气绝伤害，破防，硬直类型，物理受力等
* 根据技能硬直类型和受伤者的硬直抵抗来决定最后的状态
* 参考Dota2的伤害日志. 应该做到这样的详细程度

## 硬直类型是状态机的状态抽象类
* 播放不同动画，持续不同时间，是否无敌
* 多阶段的动画, 有特殊运动，比如浮空后倒地，或者投技
* 状态机要支持Coroutine，这样分阶段的时候方便很多

轻量状态机，带调试功能:

<https://github.com/ImYellowFish/Unity3d-Finite-State-Machine>


# 3. 连续技和记忆输入

## 基本框架: 
* 玩家原始输入->中间处理器->最终输入->状态机。各个中间处理器需要协调好彼此关系.

## 连续技系统
如果可以构成连续技，实际触发的Skill可能并不是Input中指定的Skill
比如普攻第二段是上挑，第三段下劈, 等等.

## 记忆输入
Input传入攻击的时候，如果该技能不能立刻释放（正在放其他技能，或者硬直），则等到玩家就绪 的时候释放. 

* 基本实现是一个深度只有1的指令栈，处理优先级高于连续技. 
* 当玩家进入就绪状态后自动触发.
* 需要状态机提供查询接口(FireEvent需要传回一个bool而不是void)

## 自动靠近攻击
Input传入攻击的时候，如果玩家就绪，但是距离要求不满足，则走进范围后再释放. 也是一个中间处理器.
以上三个处理器的耦合需要小心处理




# 4. 技能取消机制

## 类似拳皇那种A技能取消B技能的设定
比如：A是否能取消B前摇，A能否取消B激活，A能否取消B后摇

## 实现方法：与状态机结合,  一般优先级 +Flag + 特殊例外
* 判断下是否已经在释放技能，如果是，则判断取消规则.

* 具体用什么规则看战斗系统的设计
比如每个技能都有int优先级，优先级高的可以取消优先级低的前摇.

* 不要让策划拍脑子想，程序制定的规则肯定比策划靠谱

特殊规则例如：
* 构成连续技的可以取消前一技能的后摇
* 某些技能可以取消大部分技能的任何阶段（比如闪避）
* 某些技能可以取消某些硬直状态(比如受身取消倒地)

###	特殊Flag
动作游戏的技能经常会用到. 某些技能会带有特殊属性，比如破防，强制取消，带格挡，等等.
在读表系统中加入对Flag Enum类型的支持，可以减少很多列

Flag规则可以参考2D格斗游戏:

<http://wiki.shoryuken.com/Skullgirls/Glossary>

### 我们用到的Flag:
* 无视boss霸体但只造成轻受击=interrupt
* 无视boss霸体=ignore_guard
* 格挡=block
* 格挡反击=counter
* 霸体=armor
* 无视体积碰撞=no_collision
* 回避可以在动作的任何时间点取消=evade_cancel
* 所有动作都无法取消==no_cancel
* 跑步取消后摇=run_cancel




# 5. Standard Shader魔改

<https://github.com/ImYellowFish/UnityUtility/tree/master/ModedStandardV2017>

## 5.1. 思路:
* 只考虑Forward rendering的Fragment shader.
* 原版StandardShader已经计算出了所需的LightDir, Normal, ViewDir等变量.
* 用ShaderFeature关键字生成多个shader
* 自定义Editor用来开关各个Feature

![alt text][shaderOff]
![alt text][shaderOn]

```c++
// file: UnityStandardCore.cginc
half4 fragForwardBaseInternal (VertexOutputForwardBase i)
{
    FRAG_SETUP(i);

    // Unity standard PBS
    half4 c = UNITY_BRDF_PBS (s.diffColor, s.specColor, s.oneMinusReflectivity, s.smoothness, s.normalWorld, -s.eyeVec, gi.light, gi.indirect);
    
    //-----------custom functions (forward base)-------------
    // apply custom Emission
    c.rgb += Emission(i.tex.xy) * _EmissionMultiplier;

#ifdef _ZGAME_HSBC
	// apply HSBC effect
	c.rgb = HSBC(c.rgb, _HsbcParam, i.tex.xy);
#endif

#ifdef _ZGAME_RIM
    // apply rim color
    c.rgb += Rim(s);
#endif

	// apply extra light
#ifdef _ZGAME_FORWARD_BASE_MULTI_LIGHT
	c.rgb += ExtraLight(s, _ExtraLightDir_1, _ExtraLightColor_1, ExtraLightAtten(s.posWorld, _ExtraLightPos_1, _ExtraLightDir_1));
	c.rgb += ExtraLight(s, _ExtraLightDir_2, _ExtraLightColor_2, ExtraLightAtten(s.posWorld, _ExtraLightPos_2, _ExtraLightDir_2));
#endif

#ifdef _ZGAME_SUBSURFACE_SCATTERING
	// apply subsurface scattering
	c.rgb = c.rgb + SSS(mainLight.dir, s.normalWorld, -s.eyeVec, mainLight.color, i.tex.xy);
#endif

#ifdef _ZGAME_UV_FLOW
	// apply uv flow
	c.rgb += UV_Flow(i.tex);
#endif

    //------------end custom functions --------

    UNITY_APPLY_FOG(i.fogCoord, c.rgb);
    return OutputForward (c, s.alpha);
}
```


## 5.2. Extra light

* 给单个物体加两个单独的虚拟方向光
* 在ForwardBase pass中，除了主光源还额外计算两个光照
* 好处：美术调节更自由. 
* 节省Draw call，不用设置Layer.
* 防止多个灯光破坏Batching.

![alt text][shaderLight]

```c++
#ifdef _ZGAME_FORWARD_BASE_MULTI_LIGHT
half4 _ExtraLightDir_1;
fixed4 _ExtraLightColor_1;
fixed4 _ExtraLightPos_1;

half4 _ExtraLightDir_2;
fixed4 _ExtraLightColor_2;
fixed4 _ExtraLightPos_2;

inline fixed3 ExtraLight(FragmentCommonData s, half4 lightDir, fixed4 lightColor, fixed atten)
{
	UnityLight l;
	l.color = lightColor.rgb * lightColor.a * 2 * atten;
	l.dir = lightDir.xyz;
	UnityIndirect noIndirect = ZeroIndirect();
	return UNITY_BRDF_PBS(s.diffColor, s.specColor, s.oneMinusReflectivity, s.smoothness, s.normalWorld, -s.eyeVec, l, noIndirect);	
}

inline half ExtraLightAtten(half3 worldPos, half4 lightPos, half4 lightDir) {
	half dist_proj = dot(worldPos - lightPos.xyz, lightDir.xyz);
	half dist2 = dist_proj * dist_proj;
	return saturate((lightPos.w + 1) / (dist2 + 0.01) - lightPos.w);
}
#endif
```

## 5.3. Subsurface scattering
* 用Ramp Texture修正光照，取样参数是dot(normal, lightDir)
* 在明暗交界处加上淡红色
* 支持Mask，过滤掉头发和衣服等部位
* 低消耗的SSS效果，作为补光来说够用了

![alt text][shaderSSS]

```c++
#ifdef _ZGAME_SUBSURFACE_SCATTERING
half _SSS_Sigma;
half _SSS_Power;
fixed4 _SSS_Color;
sampler2D _SSS_Ramp;
sampler2D _SSS_Mask;

// subsurface scattering
inline fixed3 SSS(half3 lightDir, half3 normal, half3 viewDir, fixed3 lightColor, half2 uv) {	
	// half f = _SSS_Sigma * max(dot(-normal, lightDir), 0) + (1 - _SSS_Sigma) * (dot(-viewDir, lightDir) * 0.5 + 0.5);
	// fixed3 c = pow(f, _SSS_Power) * _SSS_Color.rgb * 2;
	
	half nl = dot(normal, lightDir);
	fixed3 c = pow(tex2D(_SSS_Ramp, half2(nl / 2 + 0.5, 0.5)), _SSS_Power) * _SSS_Color.rgb * lightColor;
	
	// apply mask
	c = c * tex2D(_SSS_Mask, uv).r;

	return c;
}
#endif
```

## 5.4 Rim
边缘光, 用来制作受击的闪光效果, 比描边算法要省一点.

![alt text][shaderRim]

```c++
#ifdef _ZGAME_RIM
fixed3 _RimColor;
half _RimPower;
// Add rim color to this character
inline fixed3 Rim(FragmentCommonData s) {
    half rimDot = 1 - saturate(dot(s.normalWorld, -s.eyeVec));
    return _RimColor * pow(rimDot, _RimPower);
}
#endif
```

## 5.5. HSBC
改变色调,饱和度,亮度和对比度, 用来制作变身后的色调变化

<https://forum.unity3d.com/threads/hue-saturation-brightness-contrast-shader.260649/>

## 5.6. Terrain专用DetailMap
根据美术需求做一些Mask blending.

[shaderOn]: ./md_image/shaderOn.jpg "Shader On"
[shaderOff]: ./md_image/shaderOff.jpg "Shader Off"
[shaderLight]: ./md_image/shaderLight.png "Shader Light"
[shaderSSS]: ./md_image/shaderSSS.png "Shader SSS"
[shaderRim]: ./md_image/shaderRim.png "Shader Rim"
[animEvent]: ./md_image/animEvent.jpg "Animation Events"
[skillPhase]: ./md_image/skillPhase.jpg "Skill Phase"