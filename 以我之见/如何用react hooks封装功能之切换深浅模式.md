## 如何用react hooks封装功能之切换 Dark/Light Mode？

#### Dark Mode?

Coder们喜欢Dark mode，为什么？因为深色模式让环境变暗，在阅读或者使用编辑器的场景非常利于人的注意力的集中，可以专注在亮色信息上；但其实也只有在上述的两个场景下使用深色模式才是被需求的场景，而在其他场景下（比如浏览淘宝、京东等），深色模式其实是对眼睛不友好的：“原因在于，在白底黑字的情况下，人类的双眼能接收到更多的光线，虹膜会有更多部分选择闭合，晶状体的形变也相对较少；但在黑底白字的情况下，为了提高对光线的吸收，虹膜便会张得更开，晶状体的形变也会变大，反而更容易造成眼部疲劳。”

但深色/浅色模式已经成为系统内置的一部分，在平时的UI设计中也会涉及到主题切换时的样式变化、web icon变化等，那如何去判断去获取当前的主题色系呢？主题色系变化后如何监听到呢？

#### MediaQueryList API

[MDN 文档]: https://developer.mozilla.org/zh-CN/docs/Web/API/MediaQueryList

我们先看下其兼容性，emmmm....不容乐观呢，此实验属性为实验功能，因此其兼容性尚得不到保证。

<img src="https://github.com/qianghe/blogs/blob/main/imgs/mediaQueryList-support.png?raw=true" alt="MediaQueryList Support" style="zoom:60%;" />

我们洗三看下mediaQueryList是啥？

> 一个`MediaQueryList`对象在一个[`document`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document)上维持着一系列的[媒体查询 (en-US)](https://developer.mozilla.org/en-US/docs/Web/CSS/Media_Queries/Using_media_queries)，并负责处理当媒体查询在其document上发生变化时向监听器进行通知的发送。

那我们通过什么来创建这样一个媒体查询呢？答案：**matchMedia**。

总结一下，流程如下：

1. 通过matcheMedia('xxx')创建一个新的mediaQueryList对象；
2. mediaQueryList.matches ：当前document是否匹配媒体查询；
3. 如果匹配媒体查询，可以通过addEventListener('change', function listenChange() {})来添加对媒体查询变更的监听，通过removeEventListener('change', function listenChange() {}) 卸载监听。

#### 使用hook封装

这个hook的功能如下：

1. 当前活跃的activeTheme；（用户通过这个theme也知道其是否为有效theme）
2. 提供onChange，作为theme变更时的用户回调监听函数；

```javascript
import { useState, useEffect, useCallback } from 'react'

function useTheme() {
  const [theme, setTheme] = useState(() => {
    const supportCurrentMedia = window.matchMedia(
      '(prefers-color-scheme)',
    ).media
    return supportCurrentMedia !== 'not all'
      ? window.matchMedia('(prefers-color-scheme: dark)').matches
        ? 'dark'
        : 'light'
      : ''
  })

  const onChange = useCallback(
    (fn) => {
      return fn(theme)
    },
    [theme],
  )

  useEffect(() => {
    if (theme === 'not all') {
      throw Error('Browser doesn\'t support dark mode')
    }

    const listenThemeChange = (theme) => (e) => {
      if (e.matches) {
        setTheme(theme)
      }
    }
    const darkMediaQuery = window.matchMedia('(prefers-color-scheme:dark)')
    const lightMediaQuery = window.matchMedia('(prefers-color-scheme:light)')
    const [darkThemeChangeListener, lightThemeChangeListener] = [
      listenThemeChange('dark'),
      listenThemeChange('light'),
    ]
    // 深色
    darkMediaQuery.addEventListener('change', darkThemeChangeListener)
    // 浅色
    lightMediaQuery.addEventListener('change', lightThemeChangeListener)

    return () => {
      darkMediaQuery.removeEventListener('change', darkThemeChangeListener)
      lightMediaQuery.removeEventListener('change', lightThemeChangeListener)
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [])

  return {
    theme,
    onChange,
  }
}

export default useTheme
```

使用：

```javascript
const [theme, listenThemeChange] = useTheme()

console.log('current theme', theme)

listenThemeChange((theme) => {
  // do sth if theme change~
})
```

自定义hooks: Just package top logic code, don't repeat yourself（DRY）。

