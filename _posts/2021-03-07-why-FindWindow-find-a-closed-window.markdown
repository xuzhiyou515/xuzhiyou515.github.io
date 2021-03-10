---
layout: post
title:  "为何FindWindow找到“已关闭”的窗口？"
date:   2021-03-07 16:47:23 +0800
categories: debug
---

## 背景
&emsp;为什么 FindWindow 会找到一个“已关闭”的窗口呢？在正式提出这个问题之前，先介绍一下bug产生的背景。功能实现的需求是这样的：用户在设置界面点击 `升级检测` 按钮，客户端会向服务器请求版本数据，当当前版本为最新版本时，会在设置界面弹出消息框提示用户已经是最新版本。而在测试过程中，由于返回检测结果较慢，于是我测试在消息框弹出之前便关闭设置界面，观察是否有异常现象。果然，bug 出现了，在消息框弹出之前便关闭设置界面导致主界面不再响应输入！这是为何呢？

## 定位代码
&emsp;在 debug 调试版本下，我重复了以上描述的操作，在主界面不再响应时，点击调试器的全部中断，此时代码中断在以下代码中：
```C++
while( ::IsWindow(m_hWnd) && ::GetMessage(&msg, NULL, 0, 0) ) {
    if( msg.message == WM_CLOSE && msg.hwnd == m_hWnd ) {
        nRet = msg.wParam;
        ::EnableWindow(hWndParent, TRUE);
        ::SetFocus(hWndParent);
    }
    if( !CPaintManagerUI::TranslateMessage(&msg) ) {
        ::TranslateMessage(&msg);
        ::DispatchMessage(&msg);
    }
    if( msg.message == WM_QUIT ) break;
}
```
很明显这是一个模态对话框的消息循环函数，而这个模态对话框应该就是提示用户检测信息的消息框。正是这个模态对话框的存在导致主界面不再接受输入。但是为什么这个消息框并不显示呢？于是我进一步找到了显示消息框的函数：
```C++
CWnd* pWndExist;
pWndExist = CWnd::FindWindow(NULL, STR_PERSIONAL_TITLE);
if (pWndExist)
{
    if (pWndExist->GetSafeHwnd())
    {
        MessageBox(_T("当前已是最新版本！"), NULL, MB_OK, pWndExist->GetSafeHwnd());
    }
}
```
函数很简单，先通过FindWindow函数找到设置界面的窗口，若该窗口存在则以其为 Owner 窗口展示消息框。按照代码的逻辑，之前的操作，设置窗口关闭时不应该造成消息框这个模态对话框的生成呀！于是我在上面代码中FindWindow 的返回处打下了断点，而断点触发时FindWindow居然返回了一个非 NULL 值！FindWindow 竟然会找到一个“已关闭”的窗口吗？

## 问题排查
&emsp;根据过往的认识，FindWindow是不可能找到一个“已关闭”的窗口的，那么会不会是因为那不是一个“已关闭”的窗口而是一个“正在关闭”的窗口，抱着侥幸的心理，我给判断设置窗口是否存在的语句打了个补丁：
```C++
if (pWndExist && pWndExist->IsWindowVisible())
{
    ...// 省略
}
```
事实是残酷的，实践告诉我我的修改并没有任何作用。问题出在哪呢？
&emsp;我决定向工具求助，通过断点，我获取到了窗口的句柄，然后通过 spy++ 进行了查看。
<div style="text-align: center"><img src="/assets/images/2021-03-07-spy++.png" /></div>
在 spy++ 的帮助下，罪魁祸首浮出了水面。spy++ 显示系统中仍存在一个标题为“设置”的窗口，而消息框正是以这个设置窗口作为了 Owner 窗口。消息框的顺利生成导致了主界面的disable。

## 反思
&emsp;经过以上体验，很明显通过标题查找对应窗口是是十分不可靠的，尤其是标题为简单而又易见的“设置”时。在 window class 可知的情况下，通过 window class 和标题来确定唯一的窗口是更加可信的。那么要是 window class 和标题都同时和别的窗口重复了怎么办？首先给窗口更加独特的 window class 名能够大大降低冲突的概率。其次，根据[ 微软文档 ][window class]:
> A window class is a set of attributes that the system uses as a template to create a window. Every window is a member of a window class. All window classes are process specific.

window class 是各进程独有的，可以提前对自己注册的 window class 通过[ SetClassLongPtr ][SetClassLongPtr]设置 GCL_CBCLSEXTRA 数据，而后在查找窗口时对 GCL_CBCLSEXTRA 数据进行验证从而避免冲突。不过本人认为在 window class 取名得当的情况下，重名的概率还是很小的。





[image 0]: /assets/images/2021-03-07-spy++.png
[window class]: https://docs.microsoft.com/en-us/windows/win32/winmsg/window-classes
[SetClassLongPtr]: https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setclasslongptra