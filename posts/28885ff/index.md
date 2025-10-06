# MFC 静态文本设置超链接


## MFC 静态文本设置超链接

### 1.1 MFC 静态文本超链接介绍

用 MFC 开发软件时，有时候需要设置一个超链接并用其他颜色显示出来，且鼠标点击后跳转到指定的网页。
这时候可以设置一个静态文本，弄一个超链接。

### 1.2 添加成员变量及初始化代码

在头文件中添加成员变量 `CRect m_rect;`
在初始化函数 OnInitDialog() 中添加

```cpp
GetDlgItem(IDC_STATIC_AUTHOR)-&gt;GetWindowRect(&amp;m_rect);
ScreenToClient(&amp;m_rect);
```
&gt; [!NOTE]
&gt; GetDlgItem(IDC_STATIC_AUTHOR) 里添加的是你的控件 ID。目的在于获取到 Static Text 的矩形区域。

### 1.3 添加 WM_LBUTTONUP 事件

右键控件-&gt;类向导-&gt;消息-&gt;添加 WM_LBUTTONUP 事件，OnLButtonUp() 函数实现如下
```cpp
void CAntiHashDlg::OnLButtonUp(UINT nFlags, CPoint point)
{
  if (point.x &gt; m_rect.left &amp;&amp; point.x &lt; m_rect.right &amp;&amp; point.y &lt; m_rect.bottom &amp;&amp; point.y &gt; m_rect.top)
  {
    // 网址中添加你需要打开的指定网页地址
    ShellExecute(NULL, NULL, _T(&#34;网址&#34;), NULL, NULL, SW_SHOWNORMAL);
  }
  CDialog::OnLButtonUp(nFlags, point);
}
```
### 1.4 添加鼠标移动事件 WM_MOUSEMOVE

添加方法如1.3，OnMouseMove() 函数实现如下
```cpp
void CAntiHashDlg::OnMouseMove(UINT nFlags, CPoint point)
{
  if (point.x &gt; m_rect.left &amp;&amp; point.x &lt; m_rect.right &amp;&amp; point.y &lt; m_rect.bottom &amp;&amp; point.y &gt; m_rect.top)
  {
    HCURSOR hCursor;
    hCursor = ::LoadCursor ( NULL, IDC_HAND );
    ::SetCursor ( hCursor );
  }
  CDialog::OnMouseMove(nFlags, point);
}
```
### 1.5 添加 WM_CTLCOLOR 事件

同上，OnCtlColor() 函数实现如下
```cpp
HBRUSH CAntiHashDlg::OnCtlColor(CDC* pDC, CWnd* pWnd, UINT nCtlColor)
{
  HBRUSH hbr = CDialog::OnCtlColor(pDC, pWnd, nCtlColor);

  // TODO:  在此更改 DC 的任何属性
  if (pWnd-&gt;GetDlgCtrlID() == IDC_STATIC_AUTHOR)
  {
     // RGB里面填写你所需要的颜色的 RGB 值，例如红色的值是 RGB(255,0,0)
     pDC-&gt;SetTextColor(RGB(64,148,199));
  }
  // TODO:  如果默认的不是所需画笔，则返回另一个画笔
  return hbr;
}
```
&gt; [!TIP]
&gt; 现在，点击带颜色的静态文本就会跳转指定网页。


---

> Author: Auseper  
> URL: https://xueyu-code.github.io/posts/28885ff/  

