---
layout: post
title: Unity 设置弹窗Tips位置
date:   2026-02-10 17:41
description: 根据鼠标位于屏幕的区域，设置弹窗锚点以及位置
toc: true
comments: false
categories:
 - blog
tags:
 - Unity
---

```csharp
public static void TipsPos(Transform tf)
{
	//获取ui相机
    var uiCamera = GetUICamera();
    var popup = tf.GetComponent<RectTransform>();
	//获取鼠标位置
    Vector2 mousePos = Input.mousePosition;

    float screenWidth = Screen.width;
    float screenHeight = Screen.height;

    if (mousePos.x < screenWidth / 2 && mousePos.y > screenHeight / 2)
    {
        // 左上角
        popup.anchorMin = new Vector2(0, 1);
        popup.anchorMax = new Vector2(0, 1);
        popup.pivot = new Vector2(0, 1);
    }
    else if (mousePos.x > screenWidth / 2 && mousePos.y > screenHeight / 2)
    {
        // 右上角
        popup.anchorMin = new Vector2(1, 1);
        popup.anchorMax = new Vector2(1, 1);
        popup.pivot = new Vector2(1, 1);
    }
    else if (mousePos.x < screenWidth / 2 && mousePos.y < screenHeight / 2)
    {
        // 左下角
        popup.anchorMin = new Vector2(0, 0);
        popup.anchorMax = new Vector2(0, 0);
        popup.pivot = new Vector2(0, 0);
    }
    else if (mousePos.x > screenWidth / 2 && mousePos.y < screenHeight / 2)
    {
        // 右下角
        popup.anchorMin = new Vector2(1, 0);
        popup.anchorMax = new Vector2(1, 0);
        popup.pivot = new Vector2(1, 0);
    }
    Vector2 localPoint;
    RectTransformUtility.ScreenPointToLocalPointInRectangle(popup.GetComponent<RectTransform>(), mousePos, uiCamera, out localPoint);
    popup.anchoredPosition = localPoint;
}
```