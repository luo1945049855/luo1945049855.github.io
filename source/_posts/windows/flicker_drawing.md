---
title:  "Flicker Drawing"
date:   2021-06-28
categories: Windows
---

## 屏幕闪烁原因

* 闪烁根本原因是：屏幕上内容`快速连续`刷新导致。

### WM_ERASEBKGND/WM_PAINT

* `WM_ERASEBKGND`: 擦除背景
* `WM_PAINT`: 背景之上绘制内容

擦除背景之后紧接着绘制内容，会导致屏幕内容快速刷新，导致闪屏现象。如何避免擦除背景操作：

1. 在注册窗口类时设置`WNDCLASS::hbrBackground=NULL`。
2. `WM_ERASEBKGND`消息处理中`return TRUE`。
3. `InvalidateRect(hwnd, &rect, FALSE)`第三个参数传参`FALSE`。

### CS_VREDRAW/CS_HREDRAW

当窗口style中包含两者中任意一个时，窗口在缩放时会全部重绘。
注意：当进程主窗口有任意一个以上两个属性时，当窗口缩放，除主窗口外子窗口也会全部重绘。

### WS_CLIPCHILDREN

如果父窗口没有该属性，上次子窗口内容将被父窗口覆盖，导致闪烁。
如果父窗口有该属性，则上次子窗口区域被排除在父窗口更新区域外。

## 脏区域

不要绘制非`脏区域`，下面为跟`脏区域`相关内容：

```cpp
// 1. 第二个参数为需要更新区域
WINUSERAPI BOOL WINAPI InvalidateRect(__in_opt HWND hWnd, 
                                      __in_opt CONST RECT *lpRect,
                                      __in BOOL bErase); 

// 2. 获取更新区域，可能为非规则图形
WINUSERAPI
int WINAPI GetUpdateRgn(__in HWND hWnd, __in HRGN hRgn, __in BOOL bErase);

// 3. 获取更新的矩形区域
WINUSERAPI BOOL WINAPI GetUpdateRect(__in HWND hWnd, __out_opt LPRECT lpRect, __in BOOL bErase);

// 4. BeginPaint, EndPaint
// PAINTSTRUCT::rcPaint是需要更新区域
// 注意：当使用BeginPaint和EndPaint时，Windows会自动只更新脏区域，但是额外的计算不能避免

PAINTSTRUCT ps;
HDC hdc;
case WM_PAINT:
    hdc = BeginPaint(hwnd, &ps);
    // do painting here
    EndPaint(hwnd, &ps);
    return 0;
```

## 双缓冲(Double-Buffering)

`双缓冲`实现：创建`离屏缓存`，并向离屏缓存绘制，最后将离屏缓存内容一次绘制到屏幕上。

```cpp
HDC         hdcMem;
HBITMAP     hbmMem;
HANDLE      hOld;
PAINTSTRUCT ps;
HDC         hdc;

case WM_PAINT:

    // Get DC for window
    hdc = BeginPaint(hwnd, &ps);

    // Create an off-screen DC for double-buffering
    hdcMem = CreateCompatibleDC(hdc);
    hbmMem = CreateCompatibleBitmap(hdc, win_width, win_height);

    hOld = SelectObject(hdcMem, hbmMem);

    // Draw into hdcMem here

    // Transfer the off-screen DC to the screen
    BitBlt(hdc, 0, 0, win_width, win_height, hdcMem, 0, 0, SRCCOPY);

    // Free-up the off-screen DC
    SelectObject(hdcMem, hOld);

    DeleteObject(hbmMem);
    DeleteDC (hdcMem);
    EndPaint(hwnd, &ps);

    return 0;
```

## Reference

1. [Flicker-free Drawing](http://www.catch22.net/tuts/win32/flicker-free-drawing#)
