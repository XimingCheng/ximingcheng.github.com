<!-- 
.. link: 
.. description: 
.. tags: wxWidgets,
.. date: 2013/06/28 20:40:02
.. title: wxWidgets的双缓存与高级绘图wxGraphicsContext
.. slug: wxWidgets-doubleBuffer
-->

防止绘图的闪烁，终极利器就是使用双缓存来防止闪烁，wxWidgets处理双缓存的类是wxAutoBufferedPaintDC，可以直接处理双缓存而不需要处理太多的问题。

```CPP
void PicViewCtrl::OnPaint(wxPaintEvent& event)
{
    wxAutoBufferedPaintDC dc(this);
    PrepareDC(dc);
    Render(dc);
}
```
此时要注意style的设置：

```CPP
SetBackgroundStyle(wxBG_STYLE_CUSTOM);
```

所有的绘图放在Render(dc);里面，这样可以防止绘图闪烁。

另外一种方案是自己处理双缓存的函数：

```CPP
if(IsDoubleBuffered())
{
    DrawBackground(dc, xbase, ybase, xbase+virtualwidth, ybase+virtualheight);
    DrawGrid(dc, xbase, ybase, xbase+virtualwidth, ybase+virtualheight);
    DrawFocusLine(dc);
    for(int i = xindexstart; i <= xindexend; i++)
        for(int j = yindexstart; j <= yindexend; j++)
            ShowOneCell(dc, i, j);
}
else
{
    wxMemoryDC mdc;
    wxBitmap bm = wxBitmap(2*m_iXOffset+m_iCUWidth*m_iWidthPerPixel+30, 2*m_iYOffset+m_iCUHeight*m_iHeightPerPixel+30);
    mdc.SelectObject(bm);
    DrawBackground(mdc, xbase, ybase, xbase+virtualwidth, ybase+virtualheight);
    DrawGrid(mdc, xbase, ybase, xbase+virtualwidth, ybase+virtualheight);
    DrawFocusLine(mdc);
    for(int i = xindexstart; i <= xindexend; i++)
        for(int j = yindexstart; j <= yindexend; j++)
            ShowOneCell(mdc, i, j);
    dc.Blit(xbase, ybase, virtualwidth, virtualheight, &mdc, xbase, ybase);
}
```

wxGraphicsContext是高级绘图组建，在windows下面是GDIplus，支持alpha透明色，wxGraphicsContext使用和wxDC很像，但是想要使用他，在WIN32下需要

change the wxWidgets-2.8-12 source code setup.h from：

```CPP
#ifndef wxUSE_GRAPHICS_CONTEXT
#define wxUSE_GRAPHICS_CONTEXT 0
#endif
```

To

```CPP
#ifndef wxUSE_GRAPHICS_CONTEXT
#define wxUSE_GRAPHICS_CONTEXT 1
#endif
```

you must set `-USE_GDIPLUS=1`

wxGraphicsContext的效果：

![image](../galleries/wxWidgets-doubleBuffer/result.jpg)
