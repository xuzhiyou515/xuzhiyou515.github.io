---
layout: page
title: 待解疑问
permalink: /unsloved_problems/
---

* 自绘combobox控件，点击展开下拉列表时闪烁。  
> workaround: 点击时使用 SystemParametersInfo 禁用combobox下滑动效 [ SPI_SETCOMBOBOXANIMATION ][1]

* 内嵌cef控件拉伸时闪烁。

* 多继承时，class CWebView :  public CBrowserBase, public CWnd 情况下 OnSize 函数内 CWnd::OnSize 调用会导致中断。
> 怀疑与对象布局有关，待进一步排查。

* 窗口 GetDC 后，将 hdc 保存为成员变量长期持有，但在 ReleaseDC 之前其他窗口 GetDC 获取到了同样的 hdc, 导致 hdc 失效。




[1]: https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfoa