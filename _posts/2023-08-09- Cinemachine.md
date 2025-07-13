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

### 二、Virtual Camera 详解
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

### 三、FreeLook Camera 进阶
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
** 2. RPG 角色环视案例**
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

### 四、镜头混合系统
** 1. 过渡效果配置**
|混合类型|适用场景|参数示例|
|Cut|瞬间切换|Duration:0|
|EaseInOut|平滑过渡|Duration:1.5s, Curve:Quadratic|
|Custom|特殊动画|自定义AnimationCurve|

            混合类型
            适用场景
            参数示例
            Cut
            瞬间切换
            Duration:0
            EaseInOut
            平滑过渡
            Duration:1.5s, Curve:Quadratic
            Custom
            特殊动画
            自定义AnimationCurve
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
** 2. 分镜序列案例**
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
### 五、URP 高级扩展
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
---
### 六、性能优化技巧
**1. 预算控制：**
    ```csharp
// 限制同时激活的虚拟相机数量
CinemachineCore.VirtualCameraCount = 5; 
```
1. 精度分级：
    ```csharp
// 根据距离降低更新频率
vcam.m_UpdateInterval = distance > 20f ? 0.2f : 0.05f;
```
1. 碰撞检测优化：
    ```csharp
var collider = vcam.GetComponent<CinemachineCollider>();
collider.m_Optimization = ColliderOptimization.FixedCache;
collider.m_DistanceLimit = 10f; // 最大检测距离
```
---

### 七、完整示例：BOSS战镜头系统
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
### 使用建议：
1. 快捷键加速调试：
    - 场景视图按 Ctrl+` 显示Cinemachine调试信息
    - Virtual Camera对象上按F切换预览
2. 版本控制技巧：
    ```gitignore

###  资源包推荐：
    - Cinemachine Timeline Extensions：专业过场工具
    - Cinemachine Pixel Perfect：2D像素游戏适配
    - Impulse Listener Extension：高级震动控制