---
layout: post
title: Cinemachine
date:   2023-08-09 14:27
description: Cinemachine 使用指南
toc: true
comments: false
categories:
 - blog
tags:
 - Unity
---

### 一、核心组件系统
```csharp
// 系统工作流程
1. CinemachineBrain(主相机) 
2. → 激活优先级最高的Virtual Camera 
3. → 通过Transposer/Aim控制位置和旋转
```
---

**Virtual Camera 详解**

**0. 相机重要属性**

1. Body属性深度解析与实战应用

| Body模式              | 核心功能         | 应用案例                    | 关键参数                                                     |
|---------------------|--------------|-------------------------|----------------------------------------------------------|
| Do Nothing          | 完全固定相机位置     | 监控摄像头、静态场景展示            | 无需配置                                                     |
| Framing Transposer  | 屏幕空间保持目标相对位置 | 2.5D平台游戏（如《奥日与黑暗森林》）    | `m_ScreenX/Y` (目标在画面位置)<br>`m_CameraDistance` (相机距离)     |
| Hard Lock to Target | 与目标位置完全一致    | 第一人称角色视角<br>物体内部视角      | `Follow Offset` 应设为 `(0,0,0)`                            |
| Orbital Transposer  | 可变距离+玩家输入控制  | 第三人称RPG（如《巫师3》）         | `m_RecenterToTargetHeading` (自动回中)<br>`m_XAxis` (水平旋转设置) |
| Tracked Dolly       | 沿预设轨道移动      | 过场动画路径镜头<br>赛车游戏回放      | `m_Path` (轨道路径)<br>`m_PathPosition` (位置进度)               |
| Transposer          | 世界空间固定偏移     | 俯视角游戏（如《暗黑破坏神》）<br>跟随载具 | `m_FollowOffset` (偏移向量)                                  |
案例：RPG自由视角配置
```yaml
Body: Orbital Transposer
Follow Offset: (0, 2, -5)    # 角色后方5米，高度2米
Binding Mode: Lock To Target With World Up
X Axis:
  Value Range: 0-360         # 水平360度自由旋转
  Max Speed: 300             # 鼠标移动速度
Y Axis: 
  Value Range: 0.1-0.9       # 垂直角度限制
```
2. Aim模式详解与实战配置

| Aim模式                 | 核心功能      | 应用案例              | 关键参数                                                 |
|-----------------------|-----------|-------------------|------------------------------------------------------|
| Do Nothing            | 不控制旋转     | 固定镜头场景            | -                                                    |
| Composer              | 保持目标在画面内  | 动作游戏战斗视角          | `m_DeadZone` (死区范围)<br>`m_SoftZone` (平滑区)            |
| Group Composer        | 多目标同框     | 多人游戏同屏<br>团体合影镜头  | `m_FrameSize` (画面框定范围)<br>`m_Damping` (平滑度)          |
| Hard Look At          | 锁定目标至画面中心 | 物品特写展示<br>BOSS聚焦  | `Lookahead` (运动预测)                                   |
| POV                   | 玩家输入控制旋转  | 第一人称射击            | `m_HorizontalAxis` (水平控制)<br>`m_VerticalAxis` (垂直控制) |
| Same As Follow Target | 同步目标旋转    | 载具驾驶舱视角<br>跟随旋转物体 | -                                                    |

案例：多人同屏镜头
```csharp
public void SetupGroupCamera(Transform[] targets)
{
    // 创建目标组
    var targetGroup = new GameObject("CameraTargets").AddComponent<CinemachineTargetGroup>();
    foreach(var t in targets) {
        targetGroup.AddMember(t, 1f, 2f); // 权重1，半径2米
    }

    // 配置VCam
    var vcam = GetComponent<CinemachineVirtualCamera>();
    vcam.LookAt = targetGroup.transform;
    
    var composer = vcam.AddCinemachineComponent<CinemachineGroupComposer>();
    composer.m_MinimumFOV = 40;
    composer.m_MaximumFOV = 60;
}
```
3. Binding Mode空间绑定深度解析

| 绑定模式                         | 坐标系              | 典型应用          | 视觉表现                 |
|------------------------------|------------------|---------------|----------------------|
| Lock To Target On Assign     | 目标本地空间(初始化时)     | 需要精确初始位置的场景   | 相机以目标初始朝向为基准         |
| Lock To Target With World Up | 目标本地空间(Y轴强制世界向上) | 90%第三人称游戏适用   | Y轴永远垂直向上<br>防止相机翻转   |
| Lock To Target No Roll       | 目标本地空间(无Z轴旋转)    | 飞行模拟器         | X/Y旋转自由<br>Z轴保持水平    |
| Lock To Target               | 完全目标本地空间         | 太空游戏<br>无重力环境 | 完全跟随目标旋转             |
| World Space                  | 世界坐标系            | 战略游戏<br>固定视角  | 偏移固定在世界位置            |
| Simple Follow With World Up  | 简化本地空间(Y轴世界向上)   | 2D/2.5D游戏     | 类似World Space但Z轴保持跟随 |

案例：空战游戏摄像机配置

```yaml
Body: Transposer
Follow Offset: (0, 5, -10)   # 飞机后方10米，上方5米

Aim: POV                     # 玩家自由控制观察
  Vertical Axis:
    Max Speed: 2             # 垂直旋转速度
    Min Value: -90           # 下视角限制
    Max Value: 90            # 上视角限制

Binding Mode: Lock To Target No Roll # 保持水平不翻转
```

**1. 基础配置模板**
```csharp
public void SetupThirdPersonCamera(Transform target) 
{
    var vcam = GetComponent<CinemachineVirtualCamera>();
    vcam.Follow = target;
    vcam.LookAt = target;
    
    // 基础第三人称配置
    var transposer = vcam.GetCinemachineComponent<CinemachineTransposer>();
    transposer.m_FollowOffset = new Vector3(0, 2f, -3f); // 相机偏移
    
    var composer = vcam.GetCinemachineComponent<CinemachineComposer>();
    composer.m_DeadZoneHeight = 0.1f; // 目标跟踪容差
}
```
 **2. 实用配置方案**

案例 1：TPS 战斗镜头
```yaml
Body: Transposer
Binding Mode: Lock To Target With World Up
Follow Offset: (0, 2, -4)  // 右肩视角

Aim: Composer
Dead Zone: (0.2, 0.1)      // 目标在80%画面内不调整
Soft Zone: (0.8, 0.6)      // 目标超出时平滑跟随

扩展功能：
- 添加CinemachineCollider防穿墙
- 绑定CameraShake组件受击震动
```

案例 2：2.5D 平台跳跃
```yaml
Body: Framing Transposer
Camera Distance: 10
Dead Zone: (0, 0.8)         // 垂直方向80%稳定区

Aim: Do Nothing             // 固定角度

特殊配置：
m_Lens.Orthographic = true
m_Lens.OrthographicSize = 5
```
案例 3：动态过场镜头
```csharp
// 轨道移动配置
public CinemachinePathBase path;

void Start() {
    var dolly = vcam.AddCinemachineComponent<CinemachineTrackedDolly>();
    dolly.m_Path = path;
    dolly.m_PathPosition = 0;
    dolly.m_PathUnits = PathUnits.Normalized;
    
    // 随时间推进
    StartCoroutine(MoveCameraAlongPath());
}

IEnumerator MoveCameraAlongPath() {
    while(dolly.m_PathPosition < 1) {
        dolly.m_PathPosition += Time.deltaTime * 0.25f;
        yield return null;
    }
}
```
---

**FreeLook Camera 进阶**

 **1. 环视系统结构**
```csharp
// 三环配置参数
[Header("Rig Settings")]
public float TopRig_Height = 5f;
public float TopRig_Radius = 3f;

public float MiddleRig_Height = 1.5f;
public float MiddleRig_Radius = 5f;

public float BottomRig_Height = 0.5f;
public float BottomRig_Radius = 3f;
```
**2. RPG 角色环视案例**
```csharp
void ConfigureFreeLook(CinemachineFreeLook freelook) {
    // 基本设置
    freelook.m_Orbits[0].m_Height = TopRig_Height;
    freelook.m_Orbits[0].m_Radius = TopRig_Radius;
    
    // 输入配置
    freelook.m_XAxis.m_MaxSpeed = 300;   // 水平旋转速度
    freelook.m_YAxis.m_MaxSpeed = 2;    // 垂直旋转灵敏度
    
    // 限制角度
    freelook.m_YAxis.m_MinValue = 0.1f; // 最低视角（防止穿地）
    freelook.m_YAxis.m_MaxValue = 0.9f; // 最高视角
    
    // 自动居中
    freelook.m_RecenterToTargetHeading.m_enabled = true;
    freelook.m_RecenterToTargetHeading.m_RecenterTime = 1.5f;
}
```
 **3. 车辆驾驶特写镜头**
```csharp
// 在车辆脚本中添加
public CinemachineFreeLook driverCam;

void EnterVehicle() {
    driverCam.Priority = 20; // 激活驾驶员视角
    
    // 动态调整距离
    var followOffset = driverCam.GetRig(1).GetCinemachineComponent<CinemachineComposer>();
    followOffset.m_TrackedObjectOffset = new Vector3(0, 0.8f, 0);
}
```

---

### 二、镜头混合系统

**1. 过渡效果配置**

| 混合类型      | 适用场景 | 参数示例                           |
|-----------|------|--------------------------------|
| Cut       | 瞬间切换 | Duration:0                     |
| EaseInOut | 平滑过渡 | Duration:1.5s, Curve:Quadratic |
| Custom    | 特殊动画 | 自定义AnimationCurve              |

```csharp
// 代码控制优先级
public void SwitchToCamera(CinemachineVirtualCamera targetCam)
{
    foreach(var vcam in allCameras) {
        vcam.Priority = (vcam == targetCam) ? 20 : 10;
    }
    
    // 等待混合完成
    StartCoroutine(OnCameraBlendComplete());
}
```
**2. 分镜序列案例**
```csharp
// 创建分镜序列
public List<CinemachineVirtualCamera> sequence;
private int currentIndex = 0;

IEnumerator PlayCutscene() {
    foreach (var cam in sequence) {
        cam.Priority = 100;
        yield return new WaitForSeconds(camCuts[currentIndex].duration);
        cam.Priority = 0;
        currentIndex++;
    }
}
```
---
### 三、URP 高级扩展
 **1. Camera Stack 实战**
```csharp
// 添加相机堆栈
public Camera mainCamera;
public Camera uiCamera;
public Camera vfxCamera;

void SetupCameraStack() {
    var cameraData = mainCamera.GetUniversalAdditionalCameraData();
    
    // 设置渲染层级
    cameraData.renderType = CameraRenderType.Base;
    cameraData.cameraStack.Add(uiCamera);
    cameraData.cameraStack.Add(vfxCamera);
    
    // 配置覆盖相机
    var vfxData = vfxCamera.GetUniversalAdditionalCameraData();
    vfxData.renderType = CameraRenderType.Overlay;
    vfxData.renderPostProcessing = true;
    vfxData.antialiasing = AntialiasingMode.SubpixelMorphologicalAntiAliasing;
}
```
 **2. 特殊渲染场景方案**
场景分层渲染方案：
```
主摄像机 (Base)
  ├─ 环境光遮蔽层 (Overlay, renderFeatures: SSAO)
  ├─ UI层 (Overlay, ClearDepth: true)
  └─ 后期特效层 (Overlay, PostProcessing)
```
画中画实现：
```csharp
public Camera pictureInPictureCam;

void EnablePIP() {
    var mainData = mainCamera.GetUniversalAdditionalCameraData();
    mainData.cameraStack.Add(pictureInPictureCam);
    
    // 配置画中画
    var pipData = pictureInPictureCam.GetUniversalAdditionalCameraData();
    pipData.renderType = CameraRenderType.Overlay;
    pipData.requiresColorOption = CameraOverrideOption.On;
    pipData.requiresDepthOption = CameraOverrideOption.On;
    pipData.SetRenderer(1); // 使用指定渲染器
}
```

### 四、性能优化技巧
1. 预算控制：

```csharp
// 限制同时激活的虚拟相机数量
CinemachineCore.VirtualCameraCount = 5; 
```
2. 精度分级：
```csharp
// 根据距离降低更新频率
vcam.m_UpdateInterval = distance > 20f ? 0.2f : 0.05f;
```
3. 碰撞检测优化：
 ```csharp
var collider = vcam.GetComponent<CinemachineCollider>();
collider.m_Optimization = ColliderOptimization.FixedCache;
collider.m_DistanceLimit = 10f; // 最大检测距离
```

### 五、完整示例：BOSS战镜头系统

```csharp
public class BossFightCameraSystem : MonoBehaviour
{
    [Header("相机配置")]
    public CinemachineVirtualCamera wideCam;
    public CinemachineVirtualCamera playerCam;
    public CinemachineVirtualCamera bossCloseUp;

    [Header("参数")]
    public float closeUpDuration = 3f;
    public float transitionTime = 1.5f;

    private bool isInPhaseChange;

    void Start() {
        // 初始为广角镜头
        SwitchCamera(wideCam);
    }

    public void OnPlayerDamaged() {
        if(!isInPhaseChange) SwitchCamera(playerCam);
    }

    public void OnBossPhaseChange() {
        StartCoroutine(PhaseChangeSequence());
    }

    IEnumerator PhaseChangeSequence() {
        isInPhaseChange = true;
        
        // 切到BOSS特写
        SwitchCamera(bossCloseUp, transitionTime);
        yield return new WaitForSeconds(closeUpDuration);
        
        // 返回广角镜头
        SwitchCamera(wideCam, transitionTime);
        
        isInPhaseChange = false;
    }

    void SwitchCamera(CinemachineVirtualCamera target, float blendTime = 0f) {
        CinemachineBrain brain = Camera.main.GetComponent<CinemachineBrain>();
        brain.m_DefaultBlend.m_Time = blendTime;
        
        wideCam.Priority = target == wideCam ? 20 : 10;
        playerCam.Priority = target == playerCam ? 20 : 10;
        bossCloseUp.Priority = target == bossCloseUp ? 20 : 10;
    }
}
```

**使用建议：**
1. 快捷键加速调试：
    - 场景视图按 Ctrl+` 显示Cinemachine调试信息
    - Virtual Camera对象上按F切换预览
2. 资源包推荐：
    - Cinemachine Timeline Extensions：专业过场工具
    - Cinemachine Pixel Perfect：2D像素游戏适配
    - Impulse Listener Extension：高级震动控制