虽然大三曾经短暂的接触 Android，但都只是囫囵吞枣，仅仅停留在调用 API 和开源库完成效果的程度上。既然真的要开始搞 Android，还是有必要刨根问底一下的。

作为入门，最近开始看 Google 的 [Android Training](https://developer.android.com/training/index.html)。最简单的肯定是创建一个 Hello World 工程，不过在写 LinearLayout 的时候，我发现一个比较奇怪的问题：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal">
</LinearLayout>
```

XML 文件中的前两行代码似乎很啰嗦，定义了两个 `xmlns` 和一长串没有意义的 URL。

实际上这里的 `xmlns` 指的是 XML 中的命名空间（namespace）概念。比如这里的 `android:layout_width` 属性，它就是 `android` 命名空间下的属性。如果没有命名空间的约束，整个 XML 中就不能出现重复的属性，事情就会很麻烦。

除了安卓默认提供的命名空间和控制 UI 样式的属性外，有时候，我们还可以自定义命名空间和属性，比如对于某个颜色来说，我希望它在普通模式和夜间模式下具有不同的样式，但对使用者完全透明（即对外只有一个颜色名）。

此时，就可以自定义一个 `app:bg_color`。要做到这一点，我们需要实现 `LayoutInflater.Factory` 接口并实现 `onCreateView` 方法。在将 XML 转化（inflate）为 View 的时候，实际上就是读取 XML 树中的各种属性和值，而 `onCreateView` 方法可以理解为这一过程的 Hook。

除此以外，我们也可以简单的添加几个常用的属性，[这篇文章](http://stackoverflow.com/questions/2695646/declaring-a-custom-android-ui-element-using-xml) 详细讲述了实现过程。
